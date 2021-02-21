---
layout: post
title:  "Android系统-预安装应用"
date:   2018-05-10 14:19:21 +0800
categories: [Android,Framework]
tags: [Android,Framework]
---

#### 前言
一般打包Android固件都会有一些不可避免的操作,在此分享一下,逐渐更新完善.

#### Android系统预安装应用
一般做定制的固件，都是给某个客户使用的，客户多数会要求预先安装他们提供的apk，开启自启动（这个apk里面可以完成）。

1.先分析这个apk，这个时候，可以直接解压，或者AS查看下，或者jadx，看看里面是否有so文件，如果没有可以跳过到下一步了，如果看到so库的话，解压出来。
（注 : 用Test.apk来作为例子）

2.找到源码下的**./vendor/rockchip/common/apps** , 在这个目录下新建一个包，可以随便起名，这里举例Test，把Test.apk和so(如果有so)放到Test下。
(注 : rockchip是这个目录是会变的，根据不同厂商有会所不同，但是一般会是厂商的名字)

3.在**./vendor/rockchip/common/apps**下有个apps.mk文件(反正总会有个.mk文件的)，在这个文件下加入 
```
PRODUCT_PACKAGES += Test
```

4.到Text下新建一个Android.mk
```
##############################################################################
#Test
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := Test   #这里要和你apk一个名字
LOCAL_MODULE_CLASS := APPS
LOCAL_MODULE_TAGS := optional
#LOCAL_BUILT_MODULE_STEM := package.apk
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
#LOCAL_PRIVILEGED_MODULE := true
LOCAL_CERTIFICATE := PRESIGNED
  
LOCAL_SRC_FILES := $(LOCAL_MODULE).apk 


#LOCAL_DEX_PREOPT := false
LOCAL_PREBUILT_JNI_LIBS := 1.so 2.so #这里是写入你包下的so库，名字要一样，没有顺序。 
include $(BUILD_PREBUILT) 
```

5.重新打包固件









