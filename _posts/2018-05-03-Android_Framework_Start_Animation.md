---
layout: post
title:  "Android系统-修改开机动画"
date:   2018-05-03 09:56:05 +0800
categories: Android Framework
tags:
- Android
- Framework
---

#### 前言
一般打包Android固件都会有一些不可避免的操作,在此分享一下,逐渐更新完善.

#### 修改Android开机动画
先吐槽一下Android的开机动画,因为这个动画是依靠很多图片堆起来的,可以理解为放了个GIF图上去,但是相对的,这个操作就变得很容易了.


1.把全部图片都按照**顺序命名**好,并且放在android文件夹下,例如:

![](/img/2018-05-03-Android_Framework_Start_Animation_1.png)

2.新建**desc.txt**文件,写入
```
1366 768 10
p 0 0 android
```

3.把android文件和desc.txt一起压缩(.zip格式),并且改名为**bootanimation**

4.把这个bootanimation.zip发到 ./device/rockchip/common/

5.在./device/rockchip/rk3399/rk3399.mk添加上
```
PRODUCT_COPY_FILES += device/rockchip/common/bootanimation.zip:system/media/bootanimation.zip
```
6.重新打包固件,完成.

**注意**:**第四点**和**第五点**中的路径请根据**开发板的文档来确认**,不同的平台路径可能有所不同.
