---
layout: post
title:  "RK-Android-Usb无法读取以及原理分析 "
date:   2018-06-12 18:50:24 +0800
categories: Android Framework
tags: Android,Framework,RK
---

#### 前言
如果你不知道RK是啥玩意,请先google或者baidu _(:з」∠)_

#### 环境
1. RK3368
2. Android 6.0.1

#### 需要解决的问题
当设备接入U盘的后,**RK全家桶都读不到U盘里的多媒体的资源**,例如:mp4,mp3之类的. 
(~~不幸的是,这个功能是客户的刚需.~~)

#### 解决方案
##### 分析过程
**1.** 虽然RK全家桶都没有读到U盘里的数据,但是在文件夹管理器上却可以看到挂载上去的U盘,点击进去也能看到U盘里的资料,当时是判断为可能RK全家桶有点问题,回头就去下载了RK的RockVideoPlayer源码来分析.

**2.** 在RockVideoPlayer.java下就能看到这个播放器的基本原理了,在onCreate中调用了initLoader()

 {% highlight java %}

 public void initLoader() {
        int hasWriteContactsPermission = checkSelfPermission(Manifest.permission.READ_EXTERNAL_STORAGE);
        if (hasWriteContactsPermission != PackageManager.PERMISSION_GRANTED) {
            requestPermissions(new String[]{Manifest.permission.READ_EXTERNAL_STORAGE},
                    REQUEST_CODE_ASK_PERMISSIONS);
            return;
        }
        getLoaderManager().initLoader(0, null, this);
    }
 {% endhighlight %}
 再看看
  {% highlight java %}
 public class RockVideoPlayer extends Activity implements View.OnCreateContextMenuListener,DBUtils.Def,LoaderManager.LoaderCallbacks<Cursor>{}
 {% endhighlight %}
基本就确定是用Android的CursorLoader来获得各种多媒体资源,CursorLoader也是有点小坑,以前我写过图片管理器也是用这个实现的.
再看看LoaderManager.LoaderCallbacks 这个接口的实现
 {% highlight java %}
 	@Override
	public Loader<Cursor> onCreateLoader(int arg0, Bundle arg1) {
		// TODO Auto-generated method stub
   	 	LOG("<-------------- onCreateLoader-------------->");
   	 	mSortOrder = MediaStore.Video.Media._ID;
	        StringBuilder where = new StringBuilder();
	        where.append(MediaStore.Video.Media._ID + " != ''");
	        where.append(" AND " + MediaStore.Video.Media.MIME_TYPE + " NOT LIKE 'audio%'");
		Log.i("kky", "onCreateLoader: "+ where.toString());
		Log.i("kky", "onCreateLoader: "+ MediaStore.Video.Media.EXTERNAL_CONTENT_URI);
	    return new CursorLoader(RockVideoPlayer.this, MediaStore.Video.Media.EXTERNAL_CONTENT_URI, PROJECT, where.toString(), null, mSortOrder);
    }

    @Override
    public void onLoadFinished(Loader<Cursor> arg0, Cursor arg1) {
		if(arg1 == null || arg1.getCount() == 0){
			toastNoVideo();
		}
		if(arg1 != null){
			cursorLoader = true;
			mVideoListAdapter.swapCursor(arg1);
		}else{
			cursorLoader = false;
		}		
		//getLoaderManager().getLoader(0).stopLoading();
	}

	@Override
	public void onLoaderReset(Loader<Cursor> arg0) {
		mVideoListAdapter.swapCursor(null);	
	}
 
 {% endhighlight %}
在onCreateLoader下查询并且返回CursorLoader,在onLoadFinished中,把Cursor加载进adapter中,这一套逻辑看起来并没有任何问题.而且我很早以前也写过图片管理器,和这个的原理逻辑基本上一样.



**3.** 这个时候我已经怀疑不是RK全家桶的锅了,进adb看的话,能看到挂载路径.
```
/mnt/media_rw/BC06-A913
```
在这里看到U盘的全部文件,而且find一下U盘的文件,能发现以下路径下也能找到,证明驱动那边应该是没问题的,已经挂载上了,锅可能就在framework里了
```
/storage/BC06-A913/
```
讲道理这个U盘也已经挂上去了,可播放器还是一样读不到,而CursorLoader其实就设置的查询条件去查数据库的数据,最终是调用ContentProvider的query方法进行查询,因此怀疑是数据库没有更新(这个坑我踩过),因此我发送广播进行强制刷新一下其中的一个视频文件.

 {% highlight java %}
	/**
	 * 通知媒体库更新文件
	 *
 	 * @param context
	 * @param filePath 文件全路径
	 */
	public static void scanFile(Context context, String filePath) {
		filePath = "/storage/BC06-A913/21.mp4";
		Intent scanIntent = new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE);
		scanIntent.setData(Uri.fromFile(new File(filePath)));
		context.sendBroadcast(scanIntent);
	}
 {% endhighlight %}
但是结果还是一样,播放器中找不到文件.


**4.** 去看看这个路径下的文件**./packages/providers/MediaProvider**因为播放器是用CursorLoader实现的,那么找providers下的MediaProvider来看看,如果锅在framework的话,这里的可能性就很大了,在**/src/com/android/providers/media**下的**MediaScannerReceiver.java**能搜索到对**Intent.ACTION_MEDIA_SCANNER_SCAN_FILE**的处理.

在对这个**MediaScannerReceiver.java**文件的分析,这个是一个Receiver,在onReceive方法里,可以看到一个很明显的rk的改动(我特意去看了AOSP的代码作为对比)

{% highlight java %}
 if(("true".equals(SystemProperties.get("ro.udisk.visible")))){
	String id = intent.getStringExtra(VolumeInfo.EXTRA_VOLUME_ID);
	int state = intent.getIntExtra(VolumeInfo.EXTRA_VOLUME_STATE,-1);
	if(VolumeInfo.STATE_MOUNTED == state){
		Log.d(TAG,"----MediaScanner get volume mounted,start scan---  state : " + state);
		/*StorageManager mStorageManager = context.getSystemService(StorageManager.class);
		VolumeInfo vol = mStorageManager.findVolumeById(id);
		scanFile(context, vol.getPath().getPath());*/
		scan(context, MediaProvider.EXTERNAL_VOLUME);
	}
}
 {% endhighlight %}
 
看到这里就基本上知道读不到U盘的原因了,必须要配置**ro.udisk.visible=true**

**5.** 为了验证结论,直接进入adb修改**/system/build.prop**文件,添加上**ro.udisk.visible=true**,再reboot.
然后完成重启后进入播放器就能读到U盘中的数据了.但是这个样只是临时修改,要新生产的固件也能使用的话,需要添加**./device/rockchip/common/device.mk**上添加,然后重新编译打包即可.

问题解决了,现在要研究一下原理的东西了



#### Android usb MediaProvider 原理分析

关键的东西还是在**./packages/providers/MediaProvider**下.

{% highlight java %}
public class MediaScannerReceiver extends BroadcastReceiver {}
public class MediaScannerService extends Service implements Runnable{}
{% endhighlight %}

**1.** MediaScannerReceiver负责接受广播,接受一些U盘的路径之类的信息,在开机完成后,系统发了**Intent.ACTION_BOOT_COMPLETED**,
而MediaScannerReceiver过滤到这个action后就会扫描内部和外部储存.
{% highlight java %}
@Override
    public void onReceive(Context context, Intent intent) {
        final String action = intent.getAction();
        final Uri uri = intent.getData();
        if (Intent.ACTION_BOOT_COMPLETED.equals(action)) {
            // Scan both internal and external storage
            scan(context, MediaProvider.INTERNAL_VOLUME);
            scan(context, MediaProvider.EXTERNAL_VOLUME);
            Log.d(TAG,action);
        }
        //....还有一部分,分开解释
   } 
{% endhighlight %}

**2.** 这里rk添加了一个配置其实为了可以在配置文件上修改,可以简单开关此功能,原理是这里拦截的action是指卷的状态改变,就是U盘的插拔..
{% highlight java %}
if((VolumeInfo.ACTION_VOLUME_STATE_CHANGED.equals(action))){
	if(("true".equals(SystemProperties.get("ro.udisk.visible")))){
		String id = intent.getStringExtra(VolumeInfo.EXTRA_VOLUME_ID);
		int state = intent.getIntExtra(VolumeInfo.EXTRA_VOLUME_STATE,-1);
		if(VolumeInfo.STATE_MOUNTED == state){
			Log.d(TAG,"----MediaScanner get volume mounted,start scan---  state : " + state);
			/*StorageManager mStorageManager = context.getSystemService(StorageManager.class);
			VolumeInfo vol = mStorageManager.findVolumeById(id);
			scanFile(context, vol.getPath().getPath());*/
			scan(context, MediaProvider.EXTERNAL_VOLUME);
		}
	 }
{% endhighlight %}

**3.** 这里就是我上文说的,接受发送过来的信息来刷新数据库,可以看到使用了同一个action,**Intent.ACTION_MEDIA_SCANNER_SCAN_FILE**
{% highlight java %}
if (uri != null && uri.getScheme() != null && uri.getScheme().equals("file")) {
                // handle intents related to external storage
                String path = uri.getPath();
                String externalStoragePath = Environment.getExternalStorageDirectory().getPath();
                String legacyPath = Environment.getLegacyExternalStorageDirectory().getPath();

                try {
                    path = new File(path).getCanonicalPath();
                    Log.d(TAG,"File(path).getCanonicalPath() : "+path);
                } catch (IOException e) {
                    Log.e(TAG, "couldn't canonicalize " + path);
                    return;
                }
                if (path.startsWith(legacyPath)) {
                    path = externalStoragePath + path.substring(legacyPath.length());
                    Log.d(TAG,"path.startsWith : "+path);
                }

                String packageName = intent.getStringExtra("package");
                Log.d(TAG, "action: " + action + " path: " + path + " externalStoragePath:"+externalStoragePath);
                if (Intent.ACTION_MEDIA_MOUNTED.equals(action)) {
                    // scan whenever any volume is mounted
                    scan(context, MediaProvider.EXTERNAL_VOLUME);
                } else if (Intent.ACTION_MEDIA_SCANNER_SCAN_FILE.equals(action) &&
                        path != null && path.startsWith(externalStoragePath + "/")) {
                    scanFile(context, path);
                } else if ( Intent.ACTION_MEDIA_SCANNER_SCAN_FILE.equals(action) &&
                        "RockExplorer".equals(packageName)) {
                    scan(context, MediaProvider.INTERNAL_VOLUME);
                    scan(context, MediaProvider.EXTERNAL_VOLUME);
                }
            }
{% endhighlight %}
 
 **4.** 而下面这个两个方法才是去启动扫描文件,上面那些都是接受信息和处理逻辑,又看到这两个方法其实是启动了一个Service,所以真正扫描处理是在MediaScannerService.java中.
 {% highlight java %}
 private void scan(Context context, String volume) {
        Log.d(TAG, "volume: " + volume);
        Bundle args = new Bundle();
        args.putString("volume", volume);
        context.startService(
                new Intent(context, MediaScannerService.class).putExtras(args));
    } 

    private void scanFile(Context context, String path) {
        Log.d(TAG, "filepath: " + path);
        Bundle args = new Bundle();
        args.putString("filepath", path);
        context.startService(
                new Intent(context, MediaScannerService.class).putExtras(args));
    }
{% endhighlight %}

**5.** MediaScannerService也就一个Service,按照生命周期去看,就做了一些初始化,注意这里开了新的线程,毕竟扫描是耗时的,而Service其实是在main中的.
 {% highlight java %}
    @Override
    public void onCreate()
    {
        PowerManager pm = (PowerManager)getSystemService(Context.POWER_SERVICE);
        mWakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, TAG);
        StorageManager storageManager = (StorageManager)getSystemService(Context.STORAGE_SERVICE);
        mExternalStoragePaths = storageManager.getVolumePaths();

        // Start up the thread running the service.  Note that we create a
        // separate thread because the service normally runs in the process's
        // main thread, which we don't want to block.
        Thread thr = new Thread(null, this, "MediaScannerService");
        thr.start();
    }
{% endhighlight %}

**6.** 到onStartCommand下的写法我觉得这就很骚了...一直在等待mServiceHandler的完成new(这里我也特意去对比AOSP的代码,原生的也是这样写...),而且在判断intent是否为空,空的话返回Service.START_NOT_STICKY,这样这个Service被kill后就不会restart了,执行到最后返回Service.START_REDELIVER_INTENT,如果Service被kill了,那么系统会把之前的Intent再次发送给Service,直到Service完成处理.(细节处理好评)
  {% highlight java %}
    @Override
    public int onStartCommand(Intent intent, int flags, int startId)
    {
        while (mServiceHandler == null) {
            synchronized (this) {
                try {
                    wait(100);
                } catch (InterruptedException e) {
                }
            }
        }

        if (intent == null) {
            Log.e(TAG, "Intent is null in onStartCommand: ",
                new NullPointerException());
            return Service.START_NOT_STICKY;
        }

        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent.getExtras();
        Log.d(TAG, "msg.arg1 : "+ startId+" msg.obj : " + msg.obj);
        mServiceHandler.sendMessage(msg);

        // Try again later if we are killed before we can finish scanning.
        return Service.START_REDELIVER_INTENT;
    }
{% endhighlight %}

**7.** 看到这里就直到上文的mServiceHandler一直在等待new吧,毕竟不能确定是先执行到onStartCommand,还是在线程中的mServiceHandler先new,看到这里就能发现,处理扫描的任务应该都是在这个ServiceHandler中了.
 {% highlight java %}
public void run(){
        // reduce priority below other background threads to avoid interfering
        // with other services at boot time.
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND +
                Process.THREAD_PRIORITY_LESS_FAVORABLE);
        Looper.prepare();

        mServiceLooper = Looper.myLooper();
        mServiceHandler = new ServiceHandler();

        Looper.loop();
}
{% endhighlight %}

**8.** 在ServiceHandler中就是一堆逻辑处理后调用了```scanFile```和```scan```两个方法,上文就说了CursorLoader最后也是调用ContentProvider的query方法进行查询,那ContentProvider的数据哪里来?就是这个在scanFile里调用openDatabase了然后getContentResolver().insert()
 {% highlight java %}
private Uri scanFile(String path, String mimeType) {
        String volumeName = MediaProvider.EXTERNAL_VOLUME;
        openDatabase(volumeName);
        MediaScanner scanner = createMediaScanner();
        try {
            // make sure the file path is in canonical form
            String canonicalPath = new File(path).getCanonicalPath();
            return scanner.scanSingleFile(canonicalPath, volumeName, mimeType);
        } catch (Exception e) {
            Log.e(TAG, "bad path " + path + " in scanFile()", e);
            return null;
        }
    }
    
   private void openDatabase(String volumeName) {
        try {
            ContentValues values = new ContentValues();
            values.put("name", volumeName);
            getContentResolver().insert(Uri.parse("content://media/"), values);
        } catch (IllegalArgumentException ex) {
            Log.w(TAG, "failed to open media database");
        }         
    } 
{% endhighlight %}

在scan下用WakeLock.acquire()锁住不让进入休眠,告知系统开始扫描这个路径下的文件,getContentResolver().insert插入数据,完成扫描再告诉系统扫描完成,mWakeLock.release()释放WakeLock,可以进入休眠
 {% highlight java %}
private void scan(String[] directories, String volumeName) {
        Uri uri = Uri.parse("file://" + directories[0]);
        // don't sleep while scanning
        mWakeLock.acquire();

        try {
            ContentValues values = new ContentValues();
            values.put(MediaStore.MEDIA_SCANNER_VOLUME, volumeName);
            Uri scanUri = getContentResolver().insert(MediaStore.getMediaScannerUri(), values);

            sendBroadcast(new Intent(Intent.ACTION_MEDIA_SCANNER_STARTED, uri));

            try {
                if (volumeName.equals(MediaProvider.EXTERNAL_VOLUME)) {
                    openDatabase(volumeName);
                }

                MediaScanner scanner = createMediaScanner();
                scanner.scanDirectories(directories, volumeName);
            } catch (Exception e) {
                Log.e(TAG, "exception in MediaScanner.scan()", e);
            }

            getContentResolver().delete(scanUri, null, null);

        } finally {
            sendBroadcast(new Intent(Intent.ACTION_MEDIA_SCANNER_FINISHED, uri));
            mWakeLock.release();
        }
    }
{% endhighlight %}

#### 总结 
原理 : 插入U盘后MediaScannerReceiver接受到系统发出的action后,启动MediaScannerService去进行扫描文件,把数据插入到ContentProvider,通过ContentResolver来进行操作,然后在播放器中用CursorLoader来读取.




