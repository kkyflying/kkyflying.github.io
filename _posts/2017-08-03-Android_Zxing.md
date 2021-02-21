---
layout: post
title:  "关于Android Zxing 3.3.0 的填坑"
date:   2017-08-03 00:25:28 +0800
categories: Android
tags: 
- Android
- Zxing
---

#### 前言
一开始公司的项目已经集成了Zxing,也实现了扫二维码的功能,但是前一阵子,领导发现在扫码的时候,预览的成像有问题.当时让我去修改,但是我发现公司用的zxing版本比较新,用的3.3.0的版本,我当时在网上大概的找了一些,并没有找到这个版本直接的具体修改方案,但是却提供了思路.最后我是填了一些坑,完成了任务,然后我还把这些代码都集成到一个库中. 

**因此本文重点是讲如何填坑**
-  扫描预览走样
-  扫描框走样

[源码地址](https://github.com/kkyflying/CodeScaner) 
使用方法都在github的README上面了,所以本文并不会再次讲解.

## 正文
#### Zxing 扫描走样
先上一个图说明一下原来的问题:(因为整个手机截图的图片比较大,我就放上关键的地方)

![](/img/2017-08-03-Android_Zxing_1.png)

![](/img/2017-08-03-Android_Zxing_2.png)

图1和图2相比,在预览成像的后,图1中显示的比例不对,导致硬币的Y轴被拉长,而图2中的预览成像则是正常的
**而我们的目的就是要把图1的效果改成图2**

接下来就会讲解如何修改.
 - 思路: 
首先这个形成的图像是用Camera来实现的.

![](/img/2017-08-03-Android_Zxing_3.png)

在zxing的camera包下可以看到这些类.
可以推测出

```java
CameraConfigurationManager
CameraConfigurationUtils
CameraManager
```
以上三个类是比较关键的.

在CameraConfigurationManager中的initFromCameraParameters方法里(仅展示重要的地方)
```java
void initFromCameraParameters(OpenCamera camera) {
					.
					.
					.                           
    Point theScreenResolution = new Point();
    display.getSize(theScreenResolution);
    screenResolution = theScreenResolution;
    Log.i(TAG, "Screen resolution in current orientation: " + screenResolution);
    cameraResolution = CameraConfigurationUtils.findBestPreviewSizeValue(parameters, screenResolution);
    Log.i(TAG, "Camera resolution: " + cameraResolution);
    bestPreviewSize = CameraConfigurationUtils.findBestPreviewSizeValue(parameters, screenResolution);
    Log.i(TAG, "Best available preview size: " + bestPreviewSize);
					.
					.
					.
}
```

看到这里,可以明显的看到这里就是设置预览的分辨率的地方了.
这里先是获取到屏幕的分辨率,并把这个分辨率传入到CameraConfigurationUtils.findBestPreviewSizeValue方法里,以获取到最合适的预览大小
因此我们再进入到CameraConfigurationUtils.findBestPreviewSizeValue方法里面,对这个方法进行解释.(**因为这个方法是关键的地方,以此全部拿出来进行解释**)

```java
  public static Point findBestPreviewSizeValue(Camera.Parameters parameters, Point screenResolution) {
    //在这里获取全部支持预览的分辨率
    List<Camera.Size> rawSupportedSizes = parameters.getSupportedPreviewSizes();
    if (rawSupportedSizes == null) {
      Log.w(TAG, "Device returned no supported preview sizes; using default");
      Camera.Size defaultSize = parameters.getPreviewSize();
      if (defaultSize == null) {
        throw new IllegalStateException("Parameters contained no preview size!");
      }
      return new Point(defaultSize.width, defaultSize.height);
    }

    // Sort by size, descending 对所有的分辨率进行排序
    List<Camera.Size> supportedPreviewSizes = new ArrayList<>(rawSupportedSizes);
    Collections.sort(supportedPreviewSizes, new Comparator<Camera.Size>() {
      @Override
      public int compare(Camera.Size a, Camera.Size b) {
        int aPixels = a.height * a.width;
        int bPixels = b.height * b.width;
        if (bPixels < aPixels) {
          return -1;
        }
        if (bPixels > aPixels) {
          return 1;
        }
        return 0;
      }
    });
     
     //这里会把所有支持预览的分辨率打印出来,这里可以方便进行调试	
    if (Log.isLoggable(TAG, Log.INFO)) {
      StringBuilder previewSizesString = new StringBuilder();
      for (Camera.Size supportedPreviewSize : supportedPreviewSizes) {
        previewSizesString.append(supportedPreviewSize.width).append('x')
            .append(supportedPreviewSize.height).append(' ');
      }
      Log.i(TAG, "Supported preview sizes: " + previewSizesString);
    }

    // 分辨率比例,这个一个关键点1
    double screenAspectRatio = screenResolution.y / (double) screenResolution.x;
//    double screenAspectRatio = screenResolution.x / (double) screenResolution.y;

    // Remove sizes that are unsuitable
    Iterator<Camera.Size> it = supportedPreviewSizes.iterator();
    while (it.hasNext()) {
      Camera.Size supportedPreviewSize = it.next();
      int realWidth = supportedPreviewSize.width;
      int realHeight = supportedPreviewSize.height;
      if (realWidth * realHeight < MIN_PREVIEW_PIXELS) {
        it.remove();
        continue;
      }

      boolean isCandidatePortrait = realWidth < realHeight;
      int maybeFlippedWidth = isCandidatePortrait ? realHeight : realWidth;
      int maybeFlippedHeight = isCandidatePortrait ? realWidth : realHeight;
      double aspectRatio = maybeFlippedWidth / (double) maybeFlippedHeight;
      double distortion = Math.abs(aspectRatio - screenAspectRatio);
      if (distortion > MAX_ASPECT_DISTORTION) {
        it.remove();
        continue;
      }

      if (maybeFlippedWidth == screenResolution.x && maybeFlippedHeight == screenResolution.y) {
        Point exactPoint = new Point(realWidth, realHeight);
        Log.i(TAG, "Found preview size exactly matching screen size: " + exactPoint);
        return exactPoint;
      }
    }

    // If no exact match, use largest preview size. This was not a great idea on older devices because
    // of the additional computation needed. We're likely to get here on newer Android 4+ devices, where
    // the CPU is much more powerful.
    if (!supportedPreviewSizes.isEmpty()) {
    //关键点2 ,这里是添加上去进行优化的.
      for (int i = 0;i<supportedPreviewSizes.size();i++){
        Camera.Size largestPreview = supportedPreviewSizes.get(i);
        if (largestPreview.width == screenResolution.y && largestPreview.height == screenResolution.x){
          Log.i(TAG, "Using same suitable preview size: " + largestPreview);
          return new Point(largestPreview.width, largestPreview.height);
        }
      }
      Camera.Size largestPreview = supportedPreviewSizes.get(0);
      Point largestSize = new Point(largestPreview.width, largestPreview.height);
      Log.i(TAG, "Using largest suitable preview size: " + largestSize);
      return largestSize;
    }

    // If there is nothing at all suitable, return current preview size
    Camera.Size defaultPreview = parameters.getPreviewSize();
    if (defaultPreview == null) {
      throw new IllegalStateException("Parameters contained no preview size!");
    }
//    Point defaultSize = new Point(defaultPreview.width, defaultPreview.height);
    Point defaultSize = new Point(defaultPreview.height, defaultPreview.width);
//    Log.i(TAG, "defaultPreview " + defaultPreview.width + " " + defaultPreview.height);
    Log.i(TAG, "No suitable preview sizes, using default: " + defaultSize);
    return defaultSize;
  }
```

在这里解释一下在上述代码中注释上写的关键点1的部分
```java
// 分辨率比例,这个一个关键点1(修改后)
    double screenAspectRatio = screenResolution.y / (double) screenResolution.x;
// 修改前
//    double screenAspectRatio = screenResolution.x / (double) screenResolution.y;
```
可以看到这里部分的代码我仅仅是调换x和y而已,但是就是这么简单就可以基本的修改了扫描走样的问题.
原因是,例如**在之前传入的分辨率Point(1080, 1920)和在接下来的分辨率对比获取最佳的分辨率的时候,刚好是相反的*,具体看下log*

![](/img/2017-08-03-Android_Zxing_4.png)

举个例子:
传入的是1080/1920=0.5625,而下面的代码计算的比例是 1920/1080=1.7777...
所以显示比例永远不对.
因此只要把x和y对换就可以了.在这样对比的时候,就能获取到分辨率比例一样的分辨率,从而解决预览走样的问题.
但是这样仅仅是第一步.
接下来讲解下关键点2
在算了传入的分辨率比例后,和支持的分辨率比例做对比,只要比例不同的就会移除.
但是问题来,例如:
3840x2160和1920x1080的比例是一样的.
而原本的代码是直接返回来了 supportedPreviewSizes.get(0);
所以导致比例一样,分辨率不一样,在形成图像上是不会有问题,但是会影响扫描的速度.在一些设备上会有很明显的差异!!!

```java
 if (!supportedPreviewSizes.isEmpty()) {
    //关键点2 ,这里是添加上去进行优化的.
      for (int i = 0;i<supportedPreviewSizes.size();i++){
        Camera.Size largestPreview = supportedPreviewSizes.get(i);
        if (largestPreview.width == screenResolution.y && largestPreview.height == screenResolution.x){
          Log.i(TAG, "Using same suitable preview size: " + largestPreview);
          return new Point(largestPreview.width, largestPreview.height);
        }
      }
      Camera.Size largestPreview = supportedPreviewSizes.get(0);
      Point largestSize = new Point(largestPreview.width, largestPreview.height);
      Log.i(TAG, "Using largest suitable preview size: " + largestSize);
      return largestSize;
    }
```

### Zxing 扫描框走样
这个问题我一开始也并没有发现,但是我经过测试过其他一些设备发现,在一些分辨率上,扫码框会严重变形,成一个长方形...(这个大家可以自行脑补一下,我就不上图了,刚好身边没有设备)
在ViewfinderView.java这个类中,在ondraw中有
```java
 Rect frame = cameraManager.getFramingRect();
 Rect previewFrame = cameraManager.getFramingRectInPreview();
```
从这里看到扫描框的大小就是从cameraManager.getFramingRect()获取的.进入到这个具体的方法来分析一下
在这个类中可以看到设定宽高的最大最小值

```java
    private static final int MIN_FRAME_WIDTH = 240;
    private static final int MAX_FRAME_WIDTH = 675; 

    private static final int MIN_FRAME_HEIGHT = 240;
    private static final int MAX_FRAME_HEIGHT = 675; // = 5/8 * 1080
```

进入到getFramingRect()
```java
/**
     * Calculates the framing rect which the UI should draw to show the user where to place the
     * barcode. This target helps with alignment as well as forces the user to hold the device
     * far enough away to ensure the image will be in focus.
     *
     * @return The rectangle to draw on screen in window coordinates.
     */
    public synchronized Rect getFramingRect() {
        if (framingRect == null) {
            if (camera == null) {
                return null;
            }
            Point screenResolution = configManager.getScreenResolution();
            if (screenResolution == null) {
                // Called early, before init even finished
                return null;
            }
            Log.i(TAG, "getFramingRect: " + screenResolution);
//            width
            int width = findDesiredDimensionInRange(screenResolution.x, MIN_FRAME_WIDTH,
                    MAX_FRAME_WIDTH);
//            height
            int height = findDesiredDimensionInRange(screenResolution.y, MIN_FRAME_HEIGHT,
                    MAX_FRAME_HEIGHT);
            int min = width > height ? height : width ;
            Log.i(TAG, "findDesiredDimensionInRange: width" + width + " height "+height);
            int leftOffset = (screenResolution.x - min) / 2;
            int topOffset = (screenResolution.y - min) / 2;
            framingRect = new Rect(leftOffset, topOffset, leftOffset + min, topOffset + min);
            Log.i(TAG, "getFramingRect: "+framingRect);
            if (DEBUG)
                Log.d(TAG, "Calculated framing rect: " + framingRect + ", w: " + framingRect.width() + ", h: " + framingRect.height());
        }
        return framingRect;
    }

    private static int findDesiredDimensionInRange(int resolution, int hardMin, int hardMax) {
        int dim = 5 * resolution / 8; // Target 5/8 of each dimension
        if (dim < hardMin) {
            return hardMin;
        }
        if (dim > hardMax) {
            return hardMax;
        }
        return dim;
    }
```

首先按照上面的代码进行修改就可以解决扫描框走样的问题.
可以到上面的代码中计算width和height ,因为屏幕的宽和高是肯定不一样的,所以在源码是分别算入到Rect中,出来的效果肯定不可能是个正方形,因为,只有选择width和height较小值算入Rect后,出来的效果才是一个正方形.(这里涉及到具体的view一些知识就不细讲了),当然,这里是长方形还是正方形本身都是没有错的,都是看项目的具体需求来定,所以想要什么形状都可以按照这个思路来改~

#### 注：
1. 如果说想方便快速接入zxing的话,可以直接去[源码地址](https://github.com/kkyflying/CodeScaner) 下载代码,先看看demo的部分,然后把zxing库整个放到项目中导入即可使用!十分快捷没有副作用!至于为什么没有打包,因为这个扫描的UI肯定是需要根据不同的项目要求来进行修改的,如果打包了就不方便根据需求自定义了.
2. 如果想自己导入官方的[zxing](https://github.com/zxing/zxing) 的话,可以根据上述的思路进行修改即可.
3. 若有任何问题和意见都可以在[issues](https://github.com/kkyflying/CodeScaner/issues)上留言给我.