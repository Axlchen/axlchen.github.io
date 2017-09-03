---
title: LVM动态扩展根分区实战
comments: true
date: 2017-08-15 00:49:20
updated:
categories: Linux
tags: LVM
---


## 前言
最近希望在工作电脑上下载android源码，但因为平时工作的电脑运行的是win7系统，无法下载android源码，所以想到了在VirtualBox上的Linux系统上下载。

不过，android源码体积随着版本升高已经越来越大了，一不小心就占据了几十个G，当初在虚拟机上装的系统只预留了20G左右，并且是使用了LVM技术的，还是挂载在根分区。

无奈我又不想重新装一个系统，凭着对Linux的热情和执着，一番捣鼓之后，终于得偿所愿了。在这里记录一下，希望给以后遇到同样问题的人参考参考，再这里重新演示整个过程。（换回了自己的电脑）


## 环境
- OS：OSX10.12

- VirtualBox版本：5.1.14

- Linux发行版：linux mint 17.3

## 操作
---

1. 增大虚拟硬盘

	使用VirtualBox提供了命令行工具VBoxManage，*unix系统应该在安装的时候直接加入了环境变量了，如果是源码安装或者win系统，这命令在安装目录下可以找到，首先列出已经安装的虚拟系统的硬盘：`VBoxManage list hdds`
	
	{% asset_img 2.png %}
	
	其中uuid就是这个虚拟硬盘的标识符，然后通过modifymedium命令就可以改变硬盘的大小：VBoxManage modifymedium uuid --resize xxxx
	
	{% asset_img 2.png 1.png %}

	现在，我把虚拟硬盘的容量扩大到14000mb

2. 添加物理卷（PV）

	列出现在已经有的PV：`sudo pvs`
	
	{% asset_img 8.png %}
	
	可以看到现在只有一个PV
	
	增加PV，需要用到磁盘管理工具fdisk，具体步骤
	
	- `sudo fdisk /dev/sda` (/dev/sda为对应的设备名，也可能是其它名字)
	- 按`n`新建分区
	- 一直回车选择默认
	- 按`t`改变分区的system id
	- 选择分区号
	- 设置分区system id为8e，其实就是设置分区类型为Linux LVM，通过`sudo fdisk -l`命令可以看到分区的类型

	{% asset_img 12.png %}
	
	下一步，重启使分区表生效
	
	{% asset_img 3.png %}
	
	现在，用刚才新建的分区 /dev/sda4 新建PV
	
	`sudo pvcreate /dev/sda4`

	`sudo pvs`
	
	{% asset_img 4.png %}
	
	PV已经准备好了
	
	
3.扩展卷组（VG）


`sudo vgextend mint-vg /dev/sda4` 
	
mint-vg是卷组名，装系统的时候选LVM方式作为磁盘分区的时候默认生成的
	
{% asset_img 5.png %}

现在卷组已经扩展成功了
	

4.扩展逻辑卷（LV）


查看VG的剩余空间
	
`sudo vgdisplay`
	
{% asset_img 6.png %}
	
留意到Free PE一行，总共有435个空闲的PE，1.7G的空闲空间，也就是之前扩展卷组的大小
	
`sudo lvextend -l +435 /dev/mint-vg/root`
	
-l +435 表示增加435个PE，即全部剩余空间
	
/dev/mint-vg/root 是LV path，可以通过`lvdisplay`命令查看
	
{% asset_img 7.png %}

逻辑卷也已经扩展成功了

	
5.使改变生效
	
现在用`df -h`命令查看磁盘分区的大小，可以看到根分区还是没有改变的
	
{% asset_img 9.png %}
	
`sudo resize2fs /dev/mint-vg/root`
	
{% asset_img 10.png %}
	
这时再看，已经生效了
	
**然而，在这个过程中，我遇到过一直扩展不生效的情况，看下面的重点部分**
	
## 重点

如果逻辑卷扩展后没有生效，则需要进入Resuce模式运行resize2fs命令来改变文件系统的大小

进入Resuce模式（linux mint）：重启过程中不断按esc进入系统选择界面，在选择系统界面按e，进入启动参数设置界面，在linux开头这行最后增加“init=/bin/bash”，按ctrl+x启动系统

{% asset_img 11.png %}

如果提示Read-only file system

将系统挂载成read-write：`mount / -o remount,rw`

这时再resize2fs便可


## 参考

- [https://linux.cn/article-3974-1.html?pr](https://linux.cn/article-3974-1.html?pr)
- <http://blog.csdn.net/gaoming655/article/details/7993102>
- <http://xxrenzhe.blog.51cto.com/4036116/1272838>
