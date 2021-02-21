---
layout: post
math: true
title:  "Android分辨率"
date:   2021-01-08 18:30:28 +0800
categories: [Android,适配方案]
tags: [Android]
---



## **概念**

### 1. dpi(dots-per-inch)像素密度
每英尺像素数
### 2. dp(Density Independent Pixel)
dp 是一个虚拟像素单位，1 dp 约等于中密度屏幕（160dpi；“基准”密度）上的 1 像素。对于其他每个密度，Android 会将此值转换为相应的实际像素数。（注：早期android定义为dip，后改城dp，即是dip=dp）
### 3. sp(Scale Independent Pixel) 
在定义文本大小时，您应改用可缩放像素 (sp) 作为单位（但切勿将 sp 用于布局尺寸）。默认情况下，sp 单位与 dp 大小相同，但它会根据用户的首选文本大小来调整大小。
### 4. px
像素单位，我们通常映射到屏幕像素的标准像素单位。
### 5. in
英寸，物理屏幕尺寸单位。

### 6. pt

磅 , 1pt=1/72英寸=0.035厘米

### 7. mm

毫米

### 8. dp,px,dpi之间的关系

$$
px = density * dp
$$

$$
density = dpi / 160
$$

$$
px = dp * (dpi / 160)
$$

$$
dpi=\frac {\sqrt (w^2 +h^2)(单位px)}{屏幕尺寸inch}
$$

注：屏幕尺寸是真实设备的对角线长度


## **方案**

### 1. 屏幕分辨率限定符

#### 思路

1. 选取一个基准值来进行缩放，例如1920*1080（高×宽）

2. 根据基准值生成多套资源

3. 还需要注意区分方向，即是横竖屏是的区分的

#### 优缺点 
屏幕分辨率限定符适配需要设备分辨率与 values-xx 文件夹完全匹配才能达到适配，而Android设备分辨率一大堆，**这样就需要大量的 dimens.xml 文件**，**导致apk的增大**，当匹配不上时候，会去找分辨率比例最接近的values-xx 文件夹，在效果上是不能保证的。但是如果项目仅仅只需要适配几个分辨率的情况下，理论上这个方案是十分实用的。

### 2. 最小宽度限定符
#### 思路

1. 最小宽度限定符和屏幕分辨率限定符的适配原理是一样的，系统都是根据限定符去寻找对应的 dimens.xml 文件，不同的是屏幕分辨率限定符用px，而最小宽度限定符用dp
2. **最小宽度限定符是不区分方向的**！无论是高度还是宽度，较小的那一边就认为是最小宽度
3. 需要选取一个基准值进行缩放的。例如实用320dp

#### 优缺点
相对于适配分辨率的dimens.xml 文件数量，使用最小宽度限定符的话，dimens.xml 文件数量会少很多，而且不需要区分方向！而且最小宽度限定符是不需要完全与values-xx 文件夹匹配上才能达到适配，如果没找到完全匹配的values-xx 文件夹，则是从大往小找到最接近的values-xx 文件夹，往往也是可以达到一定的适配效果。但是这个方案会有一个致命问题，如果当屏幕相关的参数不符合公式的话，极有可能出现布局整体走样，举一个本人遇到到真实案例。
```
屏幕尺寸 75
heightPixels: 2160px
widthPixels: 3840px
xdpi: 1280.0dpi
ydpi: 1280.0dpi
densityDpi: 1280dpi
density: 8.0
scaledDensity: 8.0
heightDP: 270.0dp
widthDP: 480.0dp
smallestWidthDP: 270.0dp
```
当看到这里，一定会有人问，density是8，这什么玩意？？你是不是瞎说？我保证这参数不是我乱写的，是真实抓到的数据，**请相信Android设备定制后什么神仙都有**。

前提，我的最小宽度限定符也是以360dp为基准，这个时候，往values-sw270dp匹配，我一个ImageView设置宽高都为100dp。

先根据实际的屏幕尺寸计算

$$
dpi=\frac {\sqrt (3840^2+2160^2)} {75}=58.744191202\approx60
$$

$$
density = \frac {dpi} {160} = \frac {60}{160} = 0.375
$$

这个时候是不是发现这个density也差了太远了吧！

此时，在values-sw270dp下100dp实际是约等于是58dp，按照density=8来计算的话

$$
px = density * dp = 8 * 58 = 464
$$

但实际上

$$
px = density * dp = 0.375 * 58 = 21.75
$$

这转换出来的px相差如此巨大，布局必定走样。

### 3. AndroidAutoSize

建议使用这个库，回头再补上这个库的源码分析

[AndroidAutoSize](https://github.com/JessYanCoding/AndroidAutoSize)


## 参考链接
[支持不同的像素密度](https://developer.android.google.cn/training/multiscreen/screendensities#java) 
