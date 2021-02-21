---
layout: post
title:  "AndroidAutoSize 1.2.1 源码分析"
date:   2021-02-16 20:33:10  +0800
categories: [Android,适配方案] 
tags: [Android,AndroidAutoSize]
---



## 1. 源码地址

[AndroidAutoSize](https://github.com/JessYanCoding/AndroidAutoSize) 

注：在这份框架源码里，尽管框架的[作者](https://github.com/JessYanCoding)添加十分详尽的注释，但是本人在查阅后还是决定写下这份简略的分析，以加深印象，在此感谢框架的[作者](https://github.com/JessYanCoding)的开源。

## 2. 流程

### 1. 框架入口
1. 根据文档[readme](https://github.com/JessYanCoding/AndroidAutoSize/blob/master/README-zh.md)说明，只要**AndroidManifest 中填写全局设计图尺寸 (单位 dp)**，框架就可以自动对所有页面进行适配，那么就从这里入手。
	```xml
	<manifest>
		<application>            
			<meta-data
				android:name="design_width_in_dp"
            	android:value="360"/>
        	<meta-data
            	android:name="design_height_in_dp"
            	android:value="640"/>           
     	</application>           
	</manifest>
	```

2. 查看**InitProvider#onCreate()**发现，通过InitProvider继承了ContentProvider，是一个provider，在oncreate初始化框架，配置后，利用当应用拉起来时，provider就会自动调用。查看库中的**AndroidManifest**发现，这个provider仅能用于启动框架，并且应用多线程里。

   ```xml
   <application>
		<provider
			android:name="me.jessyan.autosize.InitProvider"
			android:authorities="${applicationId}.autosize-init-provider"
			android:exported="false"
			android:multiprocess="true"/>
   </application>
   ```
   
3. 在**InitProvider#onCreate()**中调用AutoSizeConfig进行初始化配置。
   
   
   
### 2. 框架初始配置

#### 1. **AutoSizeConfig**

1. 设计模式上使用了双重校验锁单列模式
2. AutoSizeConfig#init()只能进行一次(会删除一些不重要的代码，添加更多注释)
3. 在AutoSizeConfig还有涉及的一些比较重要的引用类，例如ExternalAdaptManager，UnitsManager，ActivityLifecycleCallbacksImpl，AutoAdaptStrategy等会在下面接续讲解。用于屏幕适配的核心方法都在这里

 ```java
AutoSizeConfig init(final Application application, boolean isBaseOnWidth, AutoAdaptStrategy strategy) {
	//因为mInitDensity初始值为-1，通过判断mInitDensity的值是否有改变，如果之前初始化过一次，mInitDensity会发生改变，再次调用就会抛出异常
        Preconditions.checkArgument(mInitDensity == -1, "AutoSizeConfig#init() can only be called once");
        Preconditions.checkNotNull(application, "application == null");
        this.mApplication = application;
        this.isBaseOnWidth = isBaseOnWidth;
        final DisplayMetrics displayMetrics = Resources.getSystem().getDisplayMetrics();
        final Configuration configuration = Resources.getSystem().getConfiguration();

        //设置一个默认值, 避免在低配设备上因为获取 MetaData 过慢, 导致适配时未能正常获取到设计图尺寸
        //建议使用者在低配设备上主动在 Application#onCreate 中调用 setDesignWidthInDp 替代以使用 AndroidManifest 配置设计图尺寸的方式
        if (AutoSizeConfig.getInstance().getUnitsManager().getSupportSubunits() == Subunits.NONE) {
            mDesignWidthInDp = 360;
            mDesignHeightInDp = 640;
        } else {
            mDesignWidthInDp = 1080;
            mDesignHeightInDp = 1920;
        }
    
    /**
    *在这个方法里就会调用到设置在AndroidManifest中的design_width_in_dp和design_height_in_dp的参数，
    并且赋值于mDesignWidthInDp和mDesignHeightInDp
    **/
        getMetaData(application);
        
        /**
        *获取屏幕方向
        **/
        isVertical = application.getResources().getConfiguration().orientation == Configuration.ORIENTATION_PORTRAIT;
        
        /**
        *获取屏幕分辨率
        **/
        int[] screenSize = ScreenUtils.getScreenSize(application);
        mScreenWidth = screenSize[0];
        mScreenHeight = screenSize[1];
        
        /**
        *获取状态栏高度
        **/
        mStatusBarHeight = ScreenUtils.getStatusBarHeight();


		
        mInitDensity = displayMetrics.density;
        mInitDensityDpi = displayMetrics.densityDpi;
        mInitScaledDensity = displayMetrics.scaledDensity;
        mInitXdpi = displayMetrics.xdpi;
        mInitScreenWidthDp = configuration.screenWidthDp;
        mInitScreenHeightDp = configuration.screenHeightDp;
        
        /**
        *监听Configuration是否有改变，例如屏幕旋转，调整系统字体大小等会触发。
        *更新屏幕分辨率的值
        *更新屏幕方向
        *更新字体大小配置
        **/
        application.registerComponentCallbacks(new ComponentCallbacks() {
            @Override
            public void onConfigurationChanged(Configuration newConfig) {
                if (newConfig != null) {
                    if (newConfig.fontScale > 0) {
                        mInitScaledDensity =
                                Resources.getSystem().getDisplayMetrics().scaledDensity;
                        AutoSizeLog.d("initScaledDensity = " + mInitScaledDensity + " on ConfigurationChanged");
                    }
                    isVertical = newConfig.orientation == Configuration.ORIENTATION_PORTRAIT;
                    int[] screenSize = ScreenUtils.getScreenSize(application);
                    mScreenWidth = screenSize[0];
                    mScreenHeight = screenSize[1];
                }
            }
    
            @Override
            public void onLowMemory() {
    
            }
        });
    	
    /**
    *ActivityLifecycleCallbacksImpl是实现了Application.ActivityLifecycleCallbacks接口
    *ActivityLifecycleCallbacks会同步activity的生命周期回调
    *AutoAdaptStrategy让用户实用适配策略的接口
    *为了添加拓展性，方便用户实现自己的适配方案。
    **/
    	mActivityLifecycleCallbacks = new ActivityLifecycleCallbacksImpl(new WrapperAutoAdaptStrategy(strategy == null ? new DefaultAutoAdaptStrategy() : strategy));
        application.registerActivityLifecycleCallbacks(mActivityLifecycleCallbacks);
        return this;
    }
 ```

#### 2. **AutoAdaptStrategy**

1. AutoAdaptStrategy仅仅只是一个接口，用来方便用户自定义适配逻辑

```java
   public interface AutoAdaptStrategy {
       /**
        * 开始执行屏幕适配逻辑
        *
        * @param target   需要屏幕适配的对象 (可能是 {@link Activity} 或者 Fragment)
        * @param activity 需要拿到当前的 {@link Activity} 才能修改 {@link DisplayMetrics#density}
        */
       void applyAdapt(Object target, Activity activity);
   }
```

#### 3. **DefaultAutoAdaptStrategy**

1. DefaultAutoAdaptStrategy实现了AutoAdaptStrategy接口，是框架中的默认适配策略。
2. 逻辑是先判断检查target是否开启了外部三方库的适配模式，再判断target是否实现了CancelAdapt，再判断是否实现CustomAdapt

#### 4. **WrapperAutoAdaptStrategy**

1. WrapperAutoAdaptStrategy实现了AutoAdaptStrategy接口，在AutoSizeConfig#init和AutoSizeConfig#setAutoAdaptStrategy被调用
2. 为其他的AutoAdaptStrategy接口实现类做一个包裹，主要是为了实现AutoSizeConfig#getOnAdaptListener的适配前后的监听作用

#### 5. **ActivityLifecycleCallbacksImpl**

1. 在上文中的AutoSizeConfig#init()调用到new ActivityLifecycleCallbacksImpl(),并且也是框架中唯一一次调用到。

2. ActivityLifecycleCallbacksImpl是实现了Application.ActivityLifecycleCallbacks接口，实现了activity的生命周期回调，减少对BaseActivity的入侵。

3. **构造方法**中添加对是否要androidx或者是support的Fragment添加适配同时把autoAdaptStrategy传入，DEPENDENCY_ANDROIDX和DEPENDENCY_SUPPORT是来自AutoSizeConfig的公共静态常量，FragmentLifecycleCallbacksImplToAndroidx和FragmentLifecycleCallbacksImpl都是FragmentManager.FragmentLifecycleCallbacks抽象类的实现方法，只是分别对应在androidx包和support包，逻辑和ActivityLifecycleCallbacksImpl也是相似就不再赘述了。

   ```java
   public ActivityLifecycleCallbacksImpl(AutoAdaptStrategy autoAdaptStrategy) {
   	if (DEPENDENCY_ANDROIDX) {
   		mFragmentLifecycleCallbacksToAndroidx = new FragmentLifecycleCallbacksImplToAndroidx(autoAdaptStrategy);
   	} else if (DEPENDENCY_SUPPORT){
   		mFragmentLifecycleCallbacks = new FragmentLifecycleCallbacksImpl(autoAdaptStrategy);
   	}	
   	mAutoAdaptStrategy = autoAdaptStrategy;
   }
   ```

   4. 在onActivityCreated回调的时候，判断是否有自定义fragment适配，再把在构造方法中new的mFragmentLifecycleCallbacksToAndroidx和mFragmentLifecycleCallbacks注册到对应的FragmentManager，而且也调用AutoAdaptStrategy#applyAdapt，在onActivityStarted再次调用的原因，个人理解是：**DisplayMetrics#density是公有的，谁都有权限修改** ，有可能会出现在onActivityCreated修改好了对应的参数后，又被其他的框架给修改了，那么就会出现ui没适配好的情况，在onActivityStarted是一个保险的手段。

#### 6. **ExternalAdaptManager**

1. ExternalAdaptManager只在被AutoSizeConfig实例且持有，都通过AutoSizeConfig#getExternalAdaptManager获得。

2. ExternalAdaptManager的功能是用来管理第三方库中不需要适配的activity(不一定要第三方的)，又或者是需要配置单独的参数来适配的activity

3. 类中就是list保存需要保存不需要适配的activity的getCanonicalName，map就是保存需要单独配置额外参数的activity

   ```java
   private List<String> mCancelAdaptList;
   private Map<String, ExternalAdaptInfo> mExternalAdaptInfos;
   ```

#### 7. **UnitsManager**

1. UnitsManager只在被AutoSizeConfig实例且持有，都通过AutoSizeConfig#getUnitsManager获得。
2. UnitsManager用来保存主单位(dp,sp)和副单位(pt,in,mm),通过AutoSizeConfig可知道可以默认设置的设计图尺寸只有一组。

## 3. 框架核心

### 1. AutoSize

1. AutoSize#autoConvertDensity是框架中用于屏幕适配的核心方法，核心思路原理就是[今日头条官方适配方案](https://mp.weixin.qq.com/s/d9QCoBP6kV9VSWvVldVVwA)

   ```java
   public static void autoConvertDensity(Activity activity, float sizeInDp, boolean isBaseOnWidth) {
   	//检查activity是否为空，为空则是抛出异常
           Preconditions.checkNotNull(activity, "activity == null");
           //检查当前是否在main线程中，不是则抛出异常
           Preconditions.checkMainThread();
   		
   	//计算副单位的大小，判断是否以宽来适配，获得UnitsManager中配置
   	       float subunitsDesignSize = isBaseOnWidth ? AutoSizeConfig.getInstance().getUnitsManager().getDesignWidth()
   	               : AutoSizeConfig.getInstance().getUnitsManager().getDesignHeight();
   	       //如果是说在配置中不支持副单位的话，从获得UnitsManager得到是0，那么设置的dp参数来赋值
   	       subunitsDesignSize = subunitsDesignSize > 0 ? subunitsDesignSize : sizeInDp;
   		
   	//根据isBaseOnWidth来判断是以高还是宽适配，获取屏幕的实际分辨率(px)
   	       int screenSize = isBaseOnWidth ? AutoSizeConfig.getInstance().getScreenWidth()
   	               : AutoSizeConfig.getInstance().getScreenHeight();
   		
   	//是计算key的过程，这个key是用来DisplayMetricsInfo做缓存使用。
   	       int key = Math.round((sizeInDp + subunitsDesignSize + screenSize) * AutoSizeConfig.getInstance().getInitScaledDensity()) & ~MODE_MASK;
   	       key = isBaseOnWidth ? (key | MODE_ON_WIDTH) : (key & ~MODE_ON_WIDTH);
   	       key = AutoSizeConfig.getInstance().isUseDeviceSize() ? (key | MODE_DEVICE_SIZE) : (key & ~MODE_DEVICE_SIZE);
   	   
   	       DisplayMetricsInfo displayMetricsInfo = mCache.get(key);
   	   
   	       float targetDensity = 0;
   	       int targetDensityDpi = 0;
   	       float targetScaledDensity = 0;
   	       float targetXdpi = 0;
   	       int targetScreenWidthDp;
   	       int targetScreenHeightDp;
   		
   	//如果displayMetricsInfo从缓存拿到，就直接使用，不然就new一个新的
   	       if (displayMetricsInfo == null) {
   	       //计算密度 density=px/dp
   	           if (isBaseOnWidth) {
   	               targetDensity = AutoSizeConfig.getInstance().getScreenWidth() * 1.0f / sizeInDp;
   	           } else {
   	               targetDensity = AutoSizeConfig.getInstance().getScreenHeight() * 1.0f / sizeInDp;
   	           }
   	           //计算字体的缩放因子（正常情况下和density相等，但是调节系统字体大小后会改变这个值）
   	           //如果有设置和density的比例则直接使用
   	           if (AutoSizeConfig.getInstance().getPrivateFontScale() > 0) {
   	               targetScaledDensity = targetDensity * AutoSizeConfig.getInstance().getPrivateFontScale();
   	           } else {
   	           	//如果没有设置比例，则判断是否屏蔽系统字体大小调整的影响，true的话，则不受系统影响，false的话，则会可以通过计算之前							scaledDensity和density的比获得现在的scaledDensity
   	               float systemFontScale = AutoSizeConfig.getInstance().isExcludeFontScale() ? 1 : AutoSizeConfig.getInstance().
   	                       getInitScaledDensity() * 1.0f / AutoSizeConfig.getInstance().getInitDensity();
   	               targetScaledDensity = targetDensity * systemFontScale;
   	           }
   	           
   	           //计算dpi 
   	           targetDensityDpi = (int) (targetDensity * 160);
   			
   			// dp=px/density
   	           targetScreenWidthDp = (int) (AutoSizeConfig.getInstance().getScreenWidth() / targetDensity);
   	           targetScreenHeightDp = (int) (AutoSizeConfig.getInstance().getScreenHeight() / targetDensity);
   			
   	           //副单位的计算
   	           if (isBaseOnWidth) {
   	               targetXdpi = AutoSizeConfig.getInstance().getScreenWidth() * 1.0f / subunitsDesignSize;
   	           } else {
   	               targetXdpi = AutoSizeConfig.getInstance().getScreenHeight() * 1.0f / subunitsDesignSize;
   	           }
   		//加入缓存
   	           mCache.put(key, new DisplayMetricsInfo(targetDensity, targetDensityDpi, targetScaledDensity, targetXdpi, targetScreenWidthDp, targetScreenHeightDp));
   	       } else {
   	           targetDensity = displayMetricsInfo.getDensity();
   	           targetDensityDpi = displayMetricsInfo.getDensityDpi();
   	        targetScaledDensity = displayMetricsInfo.getScaledDensity();
   	           targetXdpi = displayMetricsInfo.getXdpi();
   	           targetScreenWidthDp = displayMetricsInfo.getScreenWidthDp();
   	           targetScreenHeightDp = displayMetricsInfo.getScreenHeightDp();
   	       }
   	       
   		//使用上面计算好的参数
   	       setDensity(activity, targetDensity, targetDensityDpi, targetScaledDensity, targetXdpi);
   	       setScreenSizeDp(activity, targetScreenWidthDp, targetScreenHeightDp);
   	   
   	   }
   ```
2. AutoSize#setDensity是在替换DisplayMetrics的参数
   ```java
private static void setDensity(DisplayMetrics displayMetrics, float density, int densityDpi, float scaledDensity, float xdpi) {
           if (AutoSizeConfig.getInstance().getUnitsManager().isSupportDP()) {
               displayMetrics.density = density;
               displayMetrics.densityDpi = densityDpi;
           }
           if (AutoSizeConfig.getInstance().getUnitsManager().isSupportSP()) {
               displayMetrics.scaledDensity = scaledDensity;
           }
           switch (AutoSizeConfig.getInstance().getUnitsManager().getSupportSubunits()) {
               case NONE:
                   break;
               case PT:
                   displayMetrics.xdpi = xdpi * 72f;
                   break;
               case IN:
                   displayMetrics.xdpi = xdpi;
                   break;
               case MM:
                   displayMetrics.xdpi = xdpi * 25.4f;
                   break;
               default:
           }
   }
   ```
3. AutoSize#setScreenSizeDp替换Configuration的参数
   ```java
private static void setScreenSizeDp(Activity activity, int screenWidthDp, int screenHeightDp) {
		if(AutoSizeConfig.getInstance().getUnitsManager().isSupportDP() &&     															 AutoSizeConfig.getInstance().getUnitsManager().isSupportScreenSizeDP()) {
            Configuration activityConfiguration = activity.getResources().getConfiguration();
            setScreenSizeDp(activityConfiguration, screenWidthDp, screenHeightDp);

            Configuration appConfiguration = AutoSizeConfig.getInstance().getApplication().getResources().getConfiguration();
            setScreenSizeDp(appConfiguration, screenWidthDp, screenHeightDp);
        }
    }
   ```
    ```java
    private static void setScreenSizeDp(Configuration configuration, int screenWidthDp, int screenHeightDp) {
        configuration.screenWidthDp = screenWidthDp;
        configuration.screenHeightDp = screenHeightDp;
    }
    ```

4.  AutoSize#cancelAdapt取消适配，就是把在AutoSizeConfig初始化的保存的各种初始参数再次调用AutoSize#setDensity和AutoSize#setScreenSizeDp进行赋值，恢复成原来的参数

   ```java
   public static void cancelAdapt(Activity activity) {
       Preconditions.checkMainThread();
       float initXdpi = AutoSizeConfig.getInstance().getInitXdpi();
       switch (AutoSizeConfig.getInstance().getUnitsManager().getSupportSubunits()) {
           case PT:
               initXdpi = initXdpi / 72f;
               break;
           case MM:
               initXdpi = initXdpi / 25.4f;
               break;
           default:
       }
       setDensity(activity, AutoSizeConfig.getInstance().getInitDensity()
               , AutoSizeConfig.getInstance().getInitDensityDpi()
               , AutoSizeConfig.getInstance().getInitScaledDensity()
               , initXdpi);
       setScreenSizeDp(activity
               , AutoSizeConfig.getInstance().getInitScreenWidthDp()
               , AutoSizeConfig.getInstance().getInitScreenHeightDp());
   }
   ```


## 4. 框架流程

### 1. 初始化流程
1. 当应用启动时，会通过InitProvider#onCreate调用AutoSizeConfig进行初始化配置
2. 在AutoSizeConfig#init中流程：
	1. 获得设备原本的参数DisplayMetrics和Configuration并保存起来了，这个可以用于在实现了CancelAdapt接口的target恢复原本的参数。
	2. 对application注册ComponentCallbacks回调接口，监听Configuration的变化，因为要字体的大小可能会受到系统的影响以及横竖屏的切换。
	3. 对application注册Application.ActivityLifecycleCallbacks回调接口，获得了activity生命周期的回调，同时用WrapperAutoAdaptStrategy包裹DefaultAutoAdaptStrategy传入ActivityLifecycleCallbacksImpl，完成设置默认的适配策略。

### 2. 适配流程
1. 框架就是通过对application注册的Application.ActivityLifecycleCallbacks回调接口，在每一次activity的构建UI之前修改参数。通过默认策略DefaultAutoAdaptStrategy实现
2. 如果调用AutoSizeConfig#setAutoAdaptStrategy设置自定义的配置模式，那么原来的DefaultAutoAdaptStrategy就被完全代替掉，只会执行自定义的AutoAdaptStrategy

## 5. 类关系图

![](/img/2021-02-16-AndroidAutoSize_class.png)


## 参考链接

[支持不同的像素密度](https://developer.android.google.cn/training/multiscreen/screendensities#java) 

[今日头条官方适配方案](https://mp.weixin.qq.com/s/d9QCoBP6kV9VSWvVldVVwA)

[骚年你的屏幕适配方式该升级了!-今日头条适配方案](https://www.jianshu.com/p/55e0fca23b4f)

