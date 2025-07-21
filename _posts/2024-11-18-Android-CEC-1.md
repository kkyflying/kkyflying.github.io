---
layout: post
math: true
title:  "HDMI CEC (1)"
date:   2024-11-18 22:34:09 +0800
categories: [Android,Framework,HDMI]
tags: [Android,Framework]
mermaid : true
---

## 前言
1. 以下分析基于android14
2.  CEC（Consumer Electronics Control）信号通过13引脚传输，作为HDMI接口的一部分。CEC总线作为控制信号被分离出来，使得在不增加数据占用宽带的情况下完成高速复杂的通信要求
3. 这优先分析一下，是怎么和cec设备通信的，也就是收发的过程



## HdmiControlService
frameworks/base/services/core/java/com/android/server/hdmi/HdmiControlService.java

cec的相关内容都在HdmiControlService里具体实现

在frameworks/base/core/java/android/content/Context.java里查看

```java
/**
     * Use with {@link #getSystemService(String)} to retrieve a
     * {@link android.hardware.hdmi.HdmiControlManager} for controlling and managing
     * HDMI-CEC protocol.
     *
     * @see #getSystemService(String)
     * @see android.hardware.hdmi.HdmiControlManager
     * @hide
     */
    @SystemApi
    public static final String HDMI_CONTROL_SERVICE = "hdmi_control";
```

frameworks/base/core/java/android/hardware/hdmi/HdmiControlManager.java

接口在通过HdmiControlManager调用到HdmiControlService

frameworks/base/core/java/android/app/SystemServiceRegistry.java 在这里注册了服务

```java
registerService(Context.HDMI_CONTROL_SERVICE, HdmiControlManager.class,
    new StaticServiceFetcher<HdmiControlManager>() {
    @Override
    public HdmiControlManager createService() throws ServiceNotFoundException {
        IBinder b = ServiceManager.getServiceOrThrow(Context.HDMI_CONTROL_SERVICE);
        return new HdmiControlManager(IHdmiControlService.Stub.asInterface(b));
}});
```



## HdmiCecController
frameworks/base/services/core/java/com/android/server/hdmi/HdmiCecController.java

1. 在这个类中，定义了cec操作的接口和实现，并做了对不同版本hal层兼容
2. HdmiControlService中的实现就是调用HdmiCecController里的接口，在HdmiControlService的调用如下：

```java
void initService() {
    //省略
    if (mCecController == null) {
            mCecController = HdmiCecController.create(this, getAtomWriter());
        }
        if (mCecController == null) {
            Slog.i(TAG, "Device does not support HDMI-CEC.");
            return;
        }
    //省略
}
```

看create方法里

```java
    static HdmiCecController create(HdmiControlService service, HdmiCecAtomWriter atomWriter) {
        HdmiCecController controller =
                createWithNativeWrapper(service, new NativeWrapperImplAidl(), atomWriter);
        if (controller != null) {
            return controller;
        }
        HdmiLogger.warning("Unable to use CEC and HDMI Connection AIDL HALs");

        controller = createWithNativeWrapper(service, new NativeWrapperImpl11(), atomWriter);
        if (controller != null) {
            return controller;
        }
        HdmiLogger.warning("Unable to use cec@1.1");
        return createWithNativeWrapper(service, new NativeWrapperImpl(), atomWriter);
    }
```

可以看到这里对连接不同hal版本做了兼容

1. NativeWrapperImplAidl是默认的版本，目前最新的版本是hal层可以通过aidl直接通到java
2. NativeWrapperImpl11是hal层实现的v1.1版本
3. NativeWrapperImpl是hal层实现的v1.0版本

在目前14版本下，会直接走NativeWrapperImplAidl,如下(后面的具体接口分析都是看NativeWrapperImplAidl)

```java
private static final class NativeWrapperImplAidl
            implements NativeWrapper, IBinder.DeathRecipient {
        private IHdmiCec mHdmiCec;
        private IHdmiConnection mHdmiConnection;
        @Nullable private HdmiCecCallback mCallback;
//省略
}
```

NativeWrapper是接口定义，关键的就是初始化，发送指令，接受指令

```java
    protected interface NativeWrapper {
        String nativeInit();
        void setCallback(HdmiCecCallback callback);
        int nativeSendCecCommand(int srcAddress, int dstAddress, byte[] body);
          //省略
    }
```

### 初始化
```java
        @Override
        public String nativeInit() {
            return connectToHal() ? mHdmiCec.toString() + " " + mHdmiConnection.toString() : null;
        }

        boolean connectToHal() {
            mHdmiCec =
                    IHdmiCec.Stub.asInterface(
                            ServiceManager.getService(IHdmiCec.DESCRIPTOR + "/default"));
            if (mHdmiCec == null) {
                HdmiLogger.error("Could not initialize HDMI CEC AIDL HAL");
                return false;
            }
            try {
                mHdmiCec.asBinder().linkToDeath(this, 0);
            } catch (RemoteException e) {
                HdmiLogger.error("Couldn't link to death : ", e);
            }

            mHdmiConnection =
                    IHdmiConnection.Stub.asInterface(
                            ServiceManager.getService(IHdmiConnection.DESCRIPTOR + "/default"));
            if (mHdmiConnection == null) {
                HdmiLogger.error("Could not initialize HDMI Connection AIDL HAL");
                return false;
            }
            try {
                mHdmiConnection.asBinder().linkToDeath(this, 0);
            } catch (RemoteException e) {
                HdmiLogger.error("Couldn't link to death : ", e);
            }
            return true;
        }
```

mHdmiCec这里是获取到了hal层服务android.hardware.tv.hdmi.cec.IHdmiCec/default

IHdmiCec是hardware/interfaces/tv/hdmi/cec/aidl/android/hardware/tv/hdmi/cec/IHdmiCec.aidl生成的代码，在out下,并且在附近还有jar包

```plain
./soong/.intermediates/hardware/interfaces/tv/hdmi/cec/aidl/android.hardware.tv.hdmi.cec-V1-java-source/gen/android/hardware/tv/hdmi/cec/IHdmiCec.java
./soong/.intermediates/hardware/interfaces/tv/cec/1.0/android.hardware.tv.cec-V1.0-java_gen_java/gen/srcs/android/hardware/tv/cec/V1_0/IHdmiCec.java
./soong/.intermediates/hardware/interfaces/tv/cec/1.1/android.hardware.tv.cec-V1.1-java_gen_java/gen/srcs/android/hardware/tv/cec/V1_1/IHdmiCec.java
```

mHdmiConnection的作用是判断cec的设备的插拔监听，获取hal层服务android.hardware.tv.hdmi.connection.IHdmiConnection/default

IHdmiConnection是通过

hardware/interfaces/tv/hdmi/connection/aidl/android/hardware/tv/hdmi/connection/IHdmiConnection.aidl生成的代码，在out下

```plain
./soong/.intermediates/hardware/interfaces/tv/hdmi/connection/aidl/android.hardware.tv.hdmi.connection-V1-java-source/gen/android/hardware/tv/hdmi/connection/IHdmiConnection.java
```

### 发送指令
这是在NativeWrapperImplAidl里的nativeSendCecCommand实现

```java
@Override
public int nativeSendCecCommand(int srcAddress, int dstAddress, byte[] body) {
    CecMessage message = new CecMessage();
    message.initiator = (byte) (srcAddress & 0xF);
    message.destination = (byte) (dstAddress & 0xF);
    message.body = body;
    try {
        return mHdmiCec.sendMessage(message);
    } catch (RemoteException e) {
        HdmiLogger.error("Failed to send CEC message : ", e);
        return SendMessageResult.FAIL;
    }
}
```

这里发送指令，就是把源地址，目前地址，数据，转成CecMessage，打包传到hal层处理

在HdmiControlService中调用HdmiCecController中的sendCommand

```java
    void sendCommand(final HdmiCecMessage cecMessage,
            final HdmiControlService.SendMessageCallback callback) {
        assertRunOnServiceThread();
        List<String> sendResults = new ArrayList<>();
        runOnIoThread(new Runnable() {
            @Override
            public void run() {
                HdmiLogger.debug("[S]:" + cecMessage);
                byte[] body = buildBody(cecMessage.getOpcode(), cecMessage.getParams());
                int retransmissionCount = 0;
                int errorCode = SendMessageResult.SUCCESS;
                do {
                    errorCode = mNativeWrapperImpl.nativeSendCecCommand(
                        cecMessage.getSource(), cecMessage.getDestination(), body);
                    switch (errorCode) {
                        case SendMessageResult.SUCCESS: sendResults.add("ACK"); break;
                        case SendMessageResult.FAIL: sendResults.add("FAIL"); break;
                        case SendMessageResult.NACK: sendResults.add("NACK"); break;
                        case SendMessageResult.BUSY: sendResults.add("BUSY"); break;
                    }
                    if (errorCode == SendMessageResult.SUCCESS) {
                        break;
                    }
                } while (retransmissionCount++ < HdmiConfig.RETRANSMISSION_COUNT);

                final int finalError = errorCode;
                if (finalError != SendMessageResult.SUCCESS) {
                    Slog.w(TAG, "Failed to send " + cecMessage + " with errorCode=" + finalError);
                }
                runOnServiceThread(new Runnable() {
                    @Override
                    public void run() {
                        mHdmiCecAtomWriter.messageReported(
                                cecMessage,
                                FrameworkStatsLog.HDMI_CEC_MESSAGE_REPORTED__DIRECTION__OUTGOING,
                                getCallingUid(),
                                finalError
                        );
                        if (callback != null) {
                            callback.onSendCompleted(finalError);
                        }
                    }
                });
            }
        });

        addCecMessageToHistory(false /* isReceived */, cecMessage, sendResults);
    }
```

这sendCommand里主要就是把操作码和参数拼凑成body，调用nativeSendCecCommand，并且把这次发送的记录保存起来，在dumpsys的时候能看到

### 接受指令
这里是在NativeWrapperImplAidl的setCallback实现，这里HdmiCecCallbackAidl继承了 IHdmiCecCallback.Stub

这个也是hardware/interfaces/tv/hdmi/cec/aidl/android/hardware/tv/hdmi/cec/IHdmiCecCallback.aidl所生成的代码，把上层的callback传递到hal层，在hal层有读到数据后就上报

```java
        public void setCallback(HdmiCecCallback callback) {
            mCallback = callback;
            try {
                // Create an AIDL callback that can callback onCecMessage
                mHdmiCec.setCallback(new HdmiCecCallbackAidl(callback));
            } catch (RemoteException e) {
                HdmiLogger.error("Couldn't initialise tv.cec callback : ", e);
            }
            try {
                // Create an AIDL callback that can callback onHotplugEvent
                mHdmiConnection.setCallback(new HdmiConnectionCallbackAidl(callback));
            } catch (RemoteException e) {
                HdmiLogger.error("Couldn't initialise tv.hdmi callback : ", e);
            }
        }
```

HdmiCecCallback也是HdmiCecController的内部类

```java
    final class HdmiCecCallback {
        @VisibleForTesting
        public void onCecMessage(int initiator, int destination, byte[] body) {
            runOnServiceThread(
                    () -> handleIncomingCecCommand(initiator, destination, body));
        }

        @VisibleForTesting
        public void onHotplugEvent(int portId, boolean connected) {
            runOnServiceThread(() -> handleHotplug(portId, connected));
        }
    }
```

这个在init设置到callback

```java
private void init(NativeWrapper nativeWrapper) {
        mIoHandler = new Handler(mService.getIoLooper());
        mControlHandler = new Handler(mService.getServiceLooper());
        nativeWrapper.setCallback(new HdmiCecCallback());
    }
```

在底层上报后在handleIncomingCecCommand里，就是把上报的数据拼装成HdmiCecMessage，加入记录保存，并onReceiveCommand

```java
private void handleIncomingCecCommand(int srcAddress, int dstAddress, byte[] body) {
        assertRunOnServiceThread();

        if (body.length == 0) {
            Slog.e(TAG, "Message with empty body received.");
            return;
        }

        HdmiCecMessage command = HdmiCecMessage.build(srcAddress, dstAddress, body[0],
                Arrays.copyOfRange(body, 1, body.length));

        if (command.getValidationResult() != HdmiCecMessageValidator.OK) {
            Slog.e(TAG, "Invalid message received: " + command);
        }

        HdmiLogger.debug("[R]:" + command);
        addCecMessageToHistory(true /* isReceived */, command, null);

        mHdmiCecAtomWriter.messageReported(command,
                incomingMessageDirection(srcAddress, dstAddress), getCallingUid());

        onReceiveCommand(command);
    }
```

通过onReceiveCommand给HdmiControlService上报

```java
   void onReceiveCommand(HdmiCecMessage message) {
        assertRunOnServiceThread();
        if (((ACTION_ON_RECEIVE_MSG & CEC_DISABLED_IGNORE) == 0)
                && !mService.isCecControlEnabled()
                && !HdmiCecMessage.isCecTransportMessage(message.getOpcode())) {
            if ((ACTION_ON_RECEIVE_MSG & CEC_DISABLED_LOG_WARNING) != 0) {
                HdmiLogger.warning("Message " + message + " received when cec disabled");
            }
            if ((ACTION_ON_RECEIVE_MSG & CEC_DISABLED_DROP_MSG) != 0) {
                return;
            }
        }
        if (mService.isAddressAllocated() && !isAcceptableAddress(message.getDestination())) {
            return;
        }
        //这里就是上报操作
        @Constants.HandleMessageResult int messageState = mService.handleCecCommand(message);
        if (messageState == Constants.NOT_HANDLED) {
            // Message was not handled
            maySendFeatureAbortCommand(message, Constants.ABORT_UNRECOGNIZED_OPCODE);
        } else if (messageState != Constants.HANDLED) {
            // Message handler wants to send a feature abort
            maySendFeatureAbortCommand(message, messageState);
        }
    }
```

在HdmiControlService收到上报的HdmiCecMessage后，会通过HdmiCecNetwork处理信息，

```java
    protected int handleCecCommand(HdmiCecMessage message) {
        assertRunOnServiceThread();

        @HdmiCecMessageValidator.ValidationResult
        int validationResult = message.getValidationResult();
        if (validationResult == HdmiCecMessageValidator.ERROR_PARAMETER
                || validationResult == HdmiCecMessageValidator.ERROR_PARAMETER_LONG
                || !verifyPhysicalAddresses(message)) {
            return Constants.ABORT_INVALID_OPERAND;
        } else if (validationResult != HdmiCecMessageValidator.OK
                || sourceAddressIsLocal(message)) {
            return Constants.HANDLED;
        }
        //这里处理信息，例如，新增cec设备，或者已经有的cec处理操作码
        getHdmiCecNetwork().handleCecMessage(message);

        @Constants.HandleMessageResult int handleMessageResult =
                dispatchMessageToLocalDevice(message);
        // mAddressAllocated is false during address allocation, meaning there is no device to
        // handle the message, so it should be buffered, if possible.
        if (!mAddressAllocated
                && mCecMessageBuffer.bufferMessage(message)) {
            return Constants.HANDLED;
        }

        return handleMessageResult;
    }
```



## 流程图

![](/img/cec-1.svg)

