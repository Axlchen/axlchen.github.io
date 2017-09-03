---
title: 基于Glide v4.x的图片加载进度监听
comments: true
date: 2017-08-13 15:48:23
updated:
categories: Android
tags: 
- glide
- 图片加载
---

Glide是一款优秀的图片加载框架，简单的配置便可以使用起来，为开发者省下了很多的功夫。不过，它没有提供其加载图片进度的api，对于这样的需求，实现起来还真颇费一番周折。

## 尝试

遇到这个需求，第一反应是网上肯定有人实现过，不妨借鉴一下别人的经验。

[Glide加载图片实现进度条效果](http://www.jianshu.com/p/548031c62257)

可惜，这个实现是基于3.7版本的，4.0版本以上的glide改动比较大，using函数已经被移除了

>using()

The using() API was removed in Glide 4 to encourage users to register their components once with a AppGlideModule to avoid object re-use. Rather than creating a new ModelLoader each time you load an image, you register it once in an AppGlideModule and let Glide inspect your model (the object you pass to load()) to figure out when to use your registered ModelLoader.

To make sure you only use your ModelLoader for certain models, implement handles() as shown above to inspect each model and return true only if your ModelLoader should be used.

## 思考
又要用最新的版本又希望给其增加功能，鱼与熊掌不可兼得？“贪新厌旧”的我怎会轻易放弃。我对加载图片进度的需求再理了一遍，发现可以用其他思路简化一下：

- 加载图片分为加载本地图片和网络图片
- 加载本地图片速度很快，主要耗时在图片解码的工作，如果图片已经在内存中，解码的步骤也省去了
- 加载网络图片主要时间耗在下载图片数据过程中

所以，能监听到图片的下载进度就大概是图片的加载进度了。那难道要先下载图片再交给glide去加载？不用，glide支持整合其他网络模块，例如OkHttp，并且如果我们愿意的话，也可以利用其接口实现自己的网络加载模块。

glide官方仓库提供了OkHttp的整合模块，于是乎我去这些代码了寻找一下突破口。


## 突破
OkHttp模块是非常简单的，只有4个文件，并且文件都不长
首先，用过glide的都知道，继承GlideModule的类是用于设置glide的配置信息和加载模块的，在OkHttpGlideModule里，向系统注册了一个用于加载GlideUrl类型的组件，简单的可以理解为加载网络图片的组件。

```java
public class OkHttpGlideModule implements GlideModule {
    public OkHttpGlideModule() {
    }

    public void applyOptions(Context context, GlideBuilder builder) {
    }

    public void registerComponents(Context context, Glide glide, Registry registry) {
        registry.replace(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory());
    }
}
```

OkHttpLoader文件也很简单，实现了ModelLoader接口，这里ModelLoader是glide的抽象的资源loader的表示，例如这里，就是说加载GlideUrl的model，返回InputStream，即图片的输入流。关于Glide的loadData、model、fetch的详细介绍，可以查看[官方文档](http://bumptech.github.io/glide/)


OkHttpLoader的最终目的，也就是返回了一个LoadData对象。现在情况明确了，glide框架就是利用这个LoadData对象得到图片的输入流，从而下载图片并经过一系列的解码，裁剪，缓存等操作，最后加载出来的。LoadData的参数有一个OkHttpStreamFetcher，从名字看来，这里一定就是下载图片的地方了，我们继续看下去。

```java
public class OkHttpUrlLoader implements ModelLoader<GlideUrl, InputStream> {
    private final okhttp3.Call.Factory client;

    public OkHttpUrlLoader(okhttp3.Call.Factory client) {
        this.client = client;
    }

    public boolean handles(GlideUrl url) {
        return true;
    }

    public LoadData<InputStream> buildLoadData(GlideUrl model, int width, int height, Options options) {
    	//返回LoadData对象，泛型为InputStream
        return new LoadData(model, new OkHttpStreamFetcher(this.client, model));
    }

    public static class Factory implements ModelLoaderFactory<GlideUrl, InputStream> {
        private static volatile okhttp3.Call.Factory internalClient;
        private okhttp3.Call.Factory client;

        private static okhttp3.Call.Factory getInternalClient() {
            if(internalClient == null) {
                Class var0 = OkHttpUrlLoader.Factory.class;
                synchronized(OkHttpUrlLoader.Factory.class) {
                    if(internalClient == null) {
                        internalClient = new OkHttpClient();
                    }
                }
            }

            return internalClient;
        }

        public Factory() {
            this(getInternalClient());
        }

        public Factory(okhttp3.Call.Factory client) {
            this.client = client;
        }

        public ModelLoader<GlideUrl, InputStream> build(MultiModelLoaderFactory multiFactory) {
            return new OkHttpUrlLoader(this.client);
        }

        public void teardown() {
        }
    }
}
```

可以看到，所有的重点都在loadData函数里面了，用过OkHttp的同学，有没有觉得好简单？哈哈。
这里直接得到图片的InputStream然后通过回调函数callback.onDataReady()返回了
问题来了，callback的glide框架里传过来的，我们并不能控制，我尝试找了一下调用loadData的地方，结果没有找到。怎么办？看最终的实现吧。

```java
public class OkHttpStreamFetcher implements DataFetcher<InputStream> {
    private static final String TAG = "OkHttpFetcher";
    private final Factory client;
    private final GlideUrl url;
    InputStream stream;
    ResponseBody responseBody;
    private volatile Call call;

    public OkHttpStreamFetcher(Factory client, GlideUrl url) {
        this.client = client;
        this.url = url;
    }

    public void loadData(Priority priority, final DataCallback<? super InputStream> callback) {
        Builder requestBuilder = (new Builder()).url(this.url.toStringUrl());
        Iterator request = this.url.getHeaders().entrySet().iterator();

        while(request.hasNext()) {
            Entry headerEntry = (Entry)request.next();
            String key = (String)headerEntry.getKey();
            requestBuilder.addHeader(key, (String)headerEntry.getValue());
        }

        Request request1 = requestBuilder.build();
        this.call = this.client.newCall(request1);
        this.call.enqueue(new Callback() {
            public void onFailure(Call call, IOException e) {
                if(Log.isLoggable("OkHttpFetcher", 3)) {
                    Log.d("OkHttpFetcher", "OkHttp failed to obtain result", e);
                }

                callback.onLoadFailed(e);
            }

            public void onResponse(Call call, Response response) throws IOException {
                OkHttpStreamFetcher.this.responseBody = response.body();
                if(response.isSuccessful()) {
                    long contentLength = OkHttpStreamFetcher.this.responseBody.contentLength();
                    OkHttpStreamFetcher.this.stream = ContentLengthInputStream.obtain(OkHttpStreamFetcher.this.responseBody.byteStream(), contentLength);
                    callback.onDataReady(OkHttpStreamFetcher.this.stream);
                } else {
                    callback.onLoadFailed(new HttpException(response.message(), response.code()));
                }

            }
        });
    }

    public void cleanup() {
        try {
            if(this.stream != null) {
                this.stream.close();
            }
        } catch (IOException var2) {
            ;
        }

        if(this.responseBody != null) {
            this.responseBody.close();
        }

    }

    public void cancel() {
        Call local = this.call;
        if(local != null) {
            local.cancel();
        }

    }

    public Class<InputStream> getDataClass() {
        return InputStream.class;
    }

    public DataSource getDataSource() {
        return DataSource.REMOTE;
    }
}
```

## 实现
在loadData函数里，有这样一句代码

>OkHttpStreamFetcher.this.stream = ContentLengthInputStream.obtain(OkHttpStreamFetcher.this.responseBody.byteStream(), contentLength);


ContentLengthInputStream是glide的工具类，用来读取输入流的，点进去看看，发现有几个read方法

```java
    public synchronized int read() throws IOException {
        int value = super.read();
        this.checkReadSoFarOrThrow(value >= 0?1:-1);
        return value;
    }

    public int read(byte[] buffer) throws IOException {
        return this.read(buffer, 0, buffer.length);
    }

    public synchronized int read(byte[] buffer, int byteOffset, int byteCount) throws IOException {
        return this.checkReadSoFarOrThrow(super.read(buffer, byteOffset, byteCount));
    }
```

所以，我的解决方法就是在这里计算下载进度和进行进度的报告的，把这个类从源码里拷贝出来，替换掉OkHttp模块里面的ContentLengthInputStream，并在这插入自己的下载进度逻辑就行了。
具体的代码实现就不贴出来了，写个大概的思路：

下载图片的时候传入图片进度监听，用集合容器，例如map保存监听的实例，key为url，下载过程中计算下载进度后通过url找到对应的回调返回进度，图片加载完毕后将回调从集合了移除（glide提供了图片加载完毕的回调）。

ok，就这样，有不合理的地方欢迎指出，又更好更优雅的实现方式请不吝指教。











