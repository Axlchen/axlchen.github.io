---
title: Android NoSuchFieldError
date: 2017-04-10 21:48:08
comments: true
categories: Android
tags: NoSuchFieldError
---


前几天在开发的时候，把一个library搬到了一个新的工程中，然后在主应用模块中调用library的Activity，发现出现了NoSuchFieldError:

{% asset_img error.png %}

然而，查看代码明明是没有问题的，layout文件存在且id正确，R文件也正常。

后来在StackOverFlow上找到了答案，原因是主应用模块和library里面的layout文件重名了，把其中一个名字改了就正常运行。后来写了个小demo重现了错误并分析了一下打包的apk


> 主模块和library模块里新建相同名字的layout文件，但两者不同，如图所示

- 主模块的文件

{% asset_img list1.png %}

- library的文件

{% asset_img list2.png %}

最后在打包生成的apk文件中，这个名字的layout文件只有一个，并且是主模块的layout文件

{% asset_img result.png %}

另外，apk文件中的resources.arsc文件中的id确实没有library中定义的id


### 结论
Android的打包机制决定了不能有同名的layout文件，故只能避免模块之间文件的重名
