---
layout:      post
title:      "Android短信接收流程"
subtitle:   "分析短信接收流程"
navcolor:   "invert"
date:       2019-12-02
author:     "Cheson"
#header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android
    - Sms
    - 短信
---

# rild 

接收来自modem的消息，在hardware/ril/reference-ril/reference-ril.c  

```c
static void onUnsolicited (const char *s, const char *sms_pdu)
{
	...
    } else if (strStartsWith(s, "+CMT:")) {
        RIL_onUnsolicitedResponse (
            RIL_UNSOL_RESPONSE_NEW_SMS,
            sms_pdu, strlen(sms_pdu));
    ...
}   
```

然后到hardware/ril/libril/ril.cpp  

```cpp
extern "C"
void RIL_onUnsolicitedResponse(int unsolResponse, void *data,
                                size_t datalen)
{
    ... ...
    unsolResponseIndex = unsolResponse - RIL_UNSOL_RESPONSE_BASE;    /* 找出消息在s_unsolResponses[]的索引 */
    ... ...
    
    // chendongqi: s_unsolResponses来自于ril_unsol_commands.h中的宏定义  
    switch (s_unsolResponses[unsolResponseIndex].wakeType) {         /* 禁止进入休眠 */
        case WAKE_PARTIAL:
            grabPartialWakeLock();
            shouldScheduleTimeout = true;
        break;
        ... ...
    }
    ... ...
    // chendongqi: 从onUnsolicited传下来的参数为RIL_UNSOL_RESPONSE_NEW_SMS
    // chendongqi: 前面找到了RIL_UNSOL_RESPONSE_NEW_SMS这个回调函数的index为unsolResponseIndex
    ret = s_unsolResponses[unsolResponseIndex]
                .responseFunction(p, data, datalen);
    ... ...
    
    ret = sendResponse(p);                                           /* 发送Parcel中的信息内容到服务端RILJ */
}
```

总而言之，根据RIL_UNSOL_RESPONSE_NEW_SMS这个消息，调用的是responseString    

```cpp
static int responseString(Parcel &p, void *response, size_t responselen) {
    /* one string only */
    startResponse;
    appendPrintBuf("%s%s", printBuf, (char*)response);
    closeResponse;

    writeStringToParcel(p, (const char *)response);

    return 0;
}
```

所做的就是把消息序列化放到Parcel中，然后通过sendResponse向framework传送消息  

```cpp
static int
sendResponse (Parcel &p) {
    printResponse;
    return sendResponseRaw(p.data(), p.dataSize());
}
```

```cpp
static int
sendResponseRaw (const void *data, size_t dataSize) {
    int fd = s_fdCommand;
    int ret;
    uint32_t header;

    if (s_fdCommand < 0) {
        return -1;
    }

    if (dataSize > MAX_COMMAND_BYTES) {
        RLOGE("RIL: packet larger than %u (%u)",
                MAX_COMMAND_BYTES, (unsigned int )dataSize);

        return -1;
    }

    pthread_mutex_lock(&s_writeMutex);

    header = htonl(dataSize);

    ret = blockingWrite(fd, (void *)&header, sizeof(header));

    if (ret < 0) {
        pthread_mutex_unlock(&s_writeMutex);
        return ret;
    }

    ret = blockingWrite(fd, data, dataSize);

    if (ret < 0) {
        pthread_mutex_unlock(&s_writeMutex);
        return ret;
    }

    pthread_mutex_unlock(&s_writeMutex);

    return 0;
}
```

这里fd来自s_fdCommand，它在listenCallback方法中连接了socket`s_fdCommand = accept(s_fdListen, (sockaddr *) &peeraddr, &socklen);`，这个s_fdListen在RIL_register中初始化了，`s_fdListen = android_get_control_socket(SOCKET_NAME_RIL);`，这个SOCKET_NAME_RIL就是“rild”  


# framework层  
与rild通讯的是framework中的framework/opt/telephony/src/java/com/android/internal/telephony/RIL.java，其中的RILReceiver是接收rild信息的地方，接收的方式是socket  

```java
    class RILReceiver implements Runnable {
        byte[] buffer;

        RILReceiver() {
            buffer = new byte[RIL_MAX_COMMAND_BYTES];
        }

        @Override
        public void
        run() {
            int retryCount = 0;

            try {for (;;) {
                LocalSocket s = null;
                LocalSocketAddress l;

                try {
                    s = new LocalSocket();
                    // chendongqi: private String SOCKET_NAME_RIL = "rild"
                    l = new LocalSocketAddress(SOCKET_NAME_RIL,
                            LocalSocketAddress.Namespace.RESERVED);
                    s.connect(l);
                } catch (IOException ex){
                    try {
                        if (s != null) {
                            s.close();
                        }
                    } catch (IOException ex2) {
                        //ignore failure to close after failure to connect
                    }

                    // don't print an error message after the the first time
                    // or after the 8th time

                    if (retryCount == 8) {
                        Rlog.e (RILJ_LOG_TAG,
                            "Couldn't find '" + SOCKET_NAME_RIL
                            + "' socket after " + retryCount
                            + " times, continuing to retry silently");
                    } else if (retryCount > 0 && retryCount < 8) {
                        Rlog.i (RILJ_LOG_TAG,
                            "Couldn't find '" + SOCKET_NAME_RIL
                            + "' socket; retrying after timeout");
                    }

                    try {
                        Thread.sleep(SOCKET_OPEN_RETRY_MILLIS);
                    } catch (InterruptedException er) {
                    }

                    retryCount++;
                    continue;
                }

                retryCount = 0;

                mSocket = s;
                Rlog.i(RILJ_LOG_TAG, "Connected to '" + SOCKET_NAME_RIL + "' socket");

                int length = 0;
                try {
                    // chendongqi: 读取到rild发来的inputStream
                    InputStream is = mSocket.getInputStream();

                    for (;;) {
                        Parcel p;

                        // chendongqi: 读取信息到buffer中
                        length = readRilMessage(is, buffer);

                        if (length < 0) {
                            // End-of-stream reached
                            break;
                        }

                        p = Parcel.obtain();
                        // chendongqi： 反序列化
                        p.unmarshall(buffer, 0, length);
                        p.setDataPosition(0);

                        //Rlog.v(RILJ_LOG_TAG, "Read packet: " + length + " bytes");

                        // chendongqi: 处理消息内容  
                        processResponse(p);
                        p.recycle();
                    }
                } catch (java.io.IOException ex) {
                    Rlog.i(RILJ_LOG_TAG, "'" + SOCKET_NAME_RIL + "' socket closed",
                          ex);
                } catch (Throwable tr) {
                    Rlog.e(RILJ_LOG_TAG, "Uncaught exception read length=" + length +
                        "Exception:" + tr.toString());
                }

                Rlog.i(RILJ_LOG_TAG, "Disconnected from '" + SOCKET_NAME_RIL
                      + "' socket");

                setRadioState (RadioState.RADIO_UNAVAILABLE);

                try {
                    mSocket.close();
                } catch (IOException ex) {
                }

                mSocket = null;
                RILRequest.resetSerial();

                // Clear request list on close
                clearRequestList(RADIO_NOT_AVAILABLE, false);
            }} catch (Throwable tr) {
                Rlog.e(RILJ_LOG_TAG,"Uncaught exception", tr);
            }

            /* We're disconnected so we don't know the ril version */
            notifyRegistrantsRilConnectionChanged(-1);
        }
    }
```

处理短信的流程会转到processUnsolicited中  

```java
    protected void
    processUnsolicited (Parcel p) {
        int response;
        Object ret;

        response = p.readInt();

        try {switch(response) {
/*
 cat libs/telephony/ril_unsol_commands.h \
 | egrep "^ *{RIL_" \
 | sed -re 's/\{([^,]+),[^,]+,([^}]+).+/case \1: \2(rr, p); break;/'
*/

            case RIL_UNSOL_RESPONSE_RADIO_STATE_CHANGED: ret =  responseVoid(p); break;
            case RIL_UNSOL_RESPONSE_CALL_STATE_CHANGED: ret =  responseVoid(p); break;
            case RIL_UNSOL_RESPONSE_VOICE_NETWORK_STATE_CHANGED: ret =  responseVoid(p); break;
            // chendongqi: RIL_UNSOL_RESPONSE_NEW_SMS--GMS短信接收消息
            case RIL_UNSOL_RESPONSE_NEW_SMS: ret =  responseString(p); break;
            case RIL_UNSOL_RESPONSE_NEW_SMS_STATUS_REPORT: ret =  responseString(p); break;
            case RIL_UNSOL_RESPONSE_NEW_SMS_ON_SIM: ret =  responseInts(p); break;
            case RIL_UNSOL_ON_USSD: ret =  /*responseStrings(p)*/ responseUnsolUssdStrings(p);/* SPRD: */ break;
            case RIL_UNSOL_NITZ_TIME_RECEIVED: ret =  responseString(p); break;
            case RIL_UNSOL_SIGNAL_STRENGTH: ret = responseSignalStrength(p); break;
            case RIL_UNSOL_DATA_CALL_LIST_CHANGED: ret = responseDataCallList(p);break;
            case RIL_UNSOL_SUPP_SVC_NOTIFICATION: ret = responseSuppServiceNotification(p); break;
            case RIL_UNSOL_STK_SESSION_END: ret = responseVoid(p); break;
            case RIL_UNSOL_STK_PROACTIVE_COMMAND: ret = responseString(p); break;
            case RIL_UNSOL_STK_EVENT_NOTIFY: ret = responseString(p); break;
            case RIL_UNSOL_STK_CALL_SETUP: ret = responseInts(p); break;
            case RIL_UNSOL_SIM_SMS_STORAGE_FULL: ret =  responseVoid(p); break;
            case RIL_UNSOL_SIM_REFRESH: ret =  responseSimRefresh(p); break;
            case RIL_UNSOL_CALL_RING: ret =  responseCallRing(p); break;
            case RIL_UNSOL_RESTRICTED_STATE_CHANGED: ret = responseInts(p); break;
            case RIL_UNSOL_RESPONSE_SIM_STATUS_CHANGED:  ret =  responseVoid(p); break;
            // chendongqi: CDMA短信接收
            case RIL_UNSOL_RESPONSE_CDMA_NEW_SMS:  ret =  responseCdmaSms(p); break;
            case RIL_UNSOL_RESPONSE_NEW_BROADCAST_SMS:  ret =  responseRaw(p); break;
            case RIL_UNSOL_CDMA_RUIM_SMS_STORAGE_FULL:  ret =  responseVoid(p); break;
            case RIL_UNSOL_ENTER_EMERGENCY_CALLBACK_MODE: ret = responseVoid(p); break;
            case RIL_UNSOL_CDMA_CALL_WAITING: ret = responseCdmaCallWaiting(p); break;
            case RIL_UNSOL_CDMA_OTA_PROVISION_STATUS: ret = responseInts(p); break;
            case RIL_UNSOL_CDMA_INFO_REC: ret = responseCdmaInformationRecord(p); break;
            case RIL_UNSOL_OEM_HOOK_RAW: ret = responseRaw(p); break;
            case RIL_UNSOL_RINGBACK_TONE: ret = responseInts(p); break;
            case RIL_UNSOL_RESEND_INCALL_MUTE: ret = responseVoid(p); break;
            case RIL_UNSOL_CDMA_SUBSCRIPTION_SOURCE_CHANGED: ret = responseInts(p); break;
            case RIL_UNSOl_CDMA_PRL_CHANGED: ret = responseInts(p); break;
            case RIL_UNSOL_EXIT_EMERGENCY_CALLBACK_MODE: ret = responseVoid(p); break;
            case RIL_UNSOL_RIL_CONNECTED: ret = responseInts(p); break;
            case RIL_UNSOL_VOICE_RADIO_TECH_CHANGED: ret =  responseInts(p); break;
            case RIL_UNSOL_CELL_INFO_LIST: ret = responseCellInfoList(p); break;
            case RIL_UNSOL_RESPONSE_IMS_NETWORK_STATE_CHANGED: ret =  responseVoid(p); break;

            default:
                throw new RuntimeException("Unrecognized unsol response: " + response);
            //break; (implied)
        }} catch (Throwable tr) {
            Rlog.e(RILJ_LOG_TAG, "Exception processing unsol response: " + response +
                "Exception:" + tr.toString());
            return;
        }

        switch(response) {
            case RIL_UNSOL_RESPONSE_RADIO_STATE_CHANGED:
                /* has bonus radio state int */
                RadioState newState = getRadioStateFromInt(p.readInt());
                if (RILJ_LOGD) unsljLogMore(response, newState.toString());

                switchToRadioState(newState);
            break;
            case RIL_UNSOL_RESPONSE_IMS_NETWORK_STATE_CHANGED:
                if (RILJ_LOGD) unsljLog(response);
                ret = responseInts(p);
                mImsNetworkStateChangedRegistrants
                    .notifyRegistrants(new AsyncResult(null, ret, null));
            break;
            case RIL_UNSOL_RESPONSE_CALL_STATE_CHANGED:
                if (RILJ_LOGD) unsljLog(response);
                /* SPRD: kill game under lowmem when call income @{ */
                if (mThread == null && OptConfig.LC_RAM_SUPPORT) {
                    mThread = new Thread(new Runnable() {
                        public void run() {
                            try {
                                if (RILJ_LOGD) riljLog("ActivityManagerNative.moveTaskToBack() start ");
                                ActivityManagerNative.getDefault().killStopFrontApp(
                                        ActivityManager.KILL_STOP_FRONT_APP);
                                if (RILJ_LOGD) riljLog("ActivityManagerNative.moveTaskToBack() end ");
                                Thread.sleep(3000);
                            } catch (RemoteException e) {
                                e.printStackTrace();
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                            mThread = null;
                        }
                    });
                    mThread.start();
                }
                /* @} */
                mCallStateRegistrants.notifyRegistrants(new AsyncResult(null, null, null));
            break;
            case RIL_UNSOL_RESPONSE_VOICE_NETWORK_STATE_CHANGED:
                if (RILJ_LOGD) unsljLog(response);

                mVoiceNetworkStateRegistrants
                    .notifyRegistrants(new AsyncResult(null, null, null));
            break;
            // chendongqi: GSM SMS处理
            case RIL_UNSOL_RESPONSE_NEW_SMS: {
                if (RILJ_LOGD) unsljLog(response);

                // FIXME this should move up a layer
                String a[] = new String[2];

                a[1] = (String)ret;

                SmsMessage sms;

                sms = SmsMessage.newFromCMT(a);//chendongqi: 构造短信内容  
                if (mGsmSmsRegistrant != null) {
                    // chendongqi: 继续通知，mGsmSmsRegistrant来自于RIL的基类BaseCommands的setOnNewGsmSms
                    mGsmSmsRegistrant
                        .notifyRegistrant(new AsyncResult(null, sms, null));
                }
            break;
            }
            ...
        }
    }
```

接下来的处理发生在InBoundSmsHandler.java中，这个继承了StateMachine，短信处理在下面   

```java
    class DeliveringState extends State {
        @Override
        public void enter() {
            if (DBG) log("entering Delivering state");
        }

        @Override
        public void exit() {
            if (DBG) log("leaving Delivering state");
        }

        @Override
        public boolean processMessage(Message msg) {
            switch (msg.what) {
                case EVENT_NEW_SMS:
                    // handle new SMS from RIL
                    // chendongqi: handleNewSms
                    handleNewSms((AsyncResult) msg.obj);
                    sendMessage(EVENT_RETURN_TO_IDLE);
                    return HANDLED;

                ...
            }
        }
    }
```

```java
    void handleNewSms(AsyncResult ar) {
        if (ar.exception != null) {
            loge("Exception processing incoming SMS: " + ar.exception);
            return;
        }

        int result;
        SmsMessage class2Sms = null; //for bug 818500
        try {
            SmsMessage sms = (SmsMessage) ar.result;
            class2Sms = sms;//for bug 818500
            // chendongqi: 下一步dispatchMessageRadioSpecific
            result = dispatchMessage(sms.mWrappedSmsMessage);
        } catch (RuntimeException ex) {
            loge("Exception dispatching message", ex);
            result = Intents.RESULT_SMS_GENERIC_ERROR;
        }
		...
    }
```

dispatchMessageRadioSpecific是一个抽象方法，实现在GsmInboundSmsHandler.java中    

```java
    @Override
    protected int dispatchMessageRadioSpecific(SmsMessageBase smsb) {
        ...
        // chendongqi: 再回到InboundSmsHandler.java中
        return dispatchNormalMessage(smsb);
    }
```

```java
    protected int dispatchNormalMessage(SmsMessageBase sms) {
        SmsHeader smsHeader = sms.getUserDataHeader();
        InboundSmsTracker tracker;
		
        // chendongqi: 短信头为空的时候
        if ((smsHeader == null) || (smsHeader.concatRef == null)) {
            // Message is not concatenated.
            int destPort = -1;
            if (smsHeader != null && smsHeader.portAddrs != null) {
                // The message was sent to a port.
                destPort = smsHeader.portAddrs.destPort;
                if (DBG) log("destination port: " + destPort);
            }

            /*
            * SPRD:
            *   To intercept Sms, and put the Sms to blackList SQlite
            *
            * @orig
            * tracker = new InboundSmsTracker(sms.getPdu(), sms.getTimestampMillis(), destPort,
            *        is3gpp2(), false);
            *
            * @{
            */

            tracker = new InboundSmsTracker(sms.getPdu(), sms.getTimestampMillis(), destPort,
                    is3gpp2(), false,sms.getOriginatingAddress());
            /*
            * @}
            */
        } else {
            // Create a tracker for this message segment.
            SmsHeader.ConcatRef concatRef = smsHeader.concatRef;
            SmsHeader.PortAddrs portAddrs = smsHeader.portAddrs;
            int destPort = (portAddrs != null ? portAddrs.destPort : -1);

            tracker = new InboundSmsTracker(sms.getPdu(), sms.getTimestampMillis(), destPort,
                    is3gpp2(), sms.getOriginatingAddress(), concatRef.refNumber,
                    concatRef.seqNumber, concatRef.msgCount, false);
        }

        if (VDBG) log("created tracker: " + tracker);
        // chendongiq:将短信插入到数据库中，并发送通知  
        return addTrackerToRawTableAndSendMessage(tracker);
    }
```

```java
    protected int addTrackerToRawTableAndSendMessage(InboundSmsTracker tracker) {
        switch(addTrackerToRawTable(tracker)) {// chendongqi: 添加数据
        // chendongiq:添加成功之后，处理发广播消息  
        case Intents.RESULT_SMS_HANDLED:
            sendMessage(EVENT_BROADCAST_SMS, tracker);
            return Intents.RESULT_SMS_HANDLED;

        case Intents.RESULT_SMS_DUPLICATED:
            return Intents.RESULT_SMS_HANDLED;

        case Intents.RESULT_SMS_GENERIC_ERROR:
        default:
            return Intents.RESULT_SMS_GENERIC_ERROR;
        }
    }
```

接下来通过一些状态机的消息处理，会交给processMessagePart函数处理   

```java
    boolean processMessagePart(InboundSmsTracker tracker) {
		...
        Intent intent;
        if (destPort == -1) {
            // chendongqi: new Intent
            intent = new Intent(Intents.SMS_DELIVER_ACTION);

            // Direct the intent to only the default SMS app. If we can't find a default SMS app
            // then sent it to all broadcast receivers.
            ComponentName componentName = SmsApplication.getDefaultSmsApplication(mContext, true);
            if (componentName != null) {
                // Deliver SMS message only to this receiver
                intent.setComponent(componentName);
                log("Delivering SMS to: " + componentName.getPackageName() +
                        " " + componentName.getClassName());
            }
        } else {
            Uri uri = Uri.parse("sms://localhost:" + destPort);
            intent = new Intent(Intents.DATA_SMS_RECEIVED_ACTION, uri);
        }

        // chendongqi：添加短信内容
        intent.putExtra("pdus", pdus);
        intent.putExtra("format", tracker.getFormat());
        intent.putExtra(Phone.PHONE_ID, mPhoneId);//SPRD: add for dsds

        /*
        * SPRD:
        * To intercept Sms, and put the Sms to blackList SQlite
        * @{
        */
        String addressNumber = tracker.getAddress();
        Log.v("BlackList intercept :", "addressNumber="+addressNumber);
        if(BlackListUtils.checkIsBlackNumber(mContext,addressNumber)){
            SmsMessage[] messages = Intents.getMessagesFromIntent(intent);
            /* SPRD:add for Bug811969. @{ */
            String body = "";
            for (int i = 0; i < messages.length; i++) {
                if (messages[i] != null) {
                    body += messages[i].getMessageBody();
                }
            }
            /* @ } */
            Log.v("BlackList intercept :", "body="+body);
            BlackListUtils.putToSmsBlackList(mContext,addressNumber,body,(new Date()).getTime());
            deleteFromRawTable(tracker.getDeleteWhere(), tracker.getDeleteWhereArgs());
            return false;
        }
        /*
        * @}
        */
		
        // chendongqi: 发送广播
        dispatchIntent(intent, android.Manifest.permission.RECEIVE_SMS,
                AppOpsManager.OP_RECEIVE_SMS, resultReceiver);
        return true;
    }
```

然后自己接收这个广播，接收之后再次发送action为SMS_RECEIVED_ACTION的广播    

```java
    /**
     * Handler for an {@link InboundSmsTracker} broadcast. Deletes PDUs from the raw table and
     * logs the broadcast duration (as an error if the other receivers were especially slow).
     */
    private final class SmsBroadcastReceiver extends BroadcastReceiver {
        private final String mDeleteWhere;
        private final String[] mDeleteWhereArgs;
        private long mBroadcastTimeNano;

        SmsBroadcastReceiver(InboundSmsTracker tracker) {
            mDeleteWhere = tracker.getDeleteWhere();
            mDeleteWhereArgs = tracker.getDeleteWhereArgs();
            mBroadcastTimeNano = System.nanoTime();
        }

        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            if (action.equals(Intents.SMS_DELIVER_ACTION)) {
                // Now dispatch the notification only intent
                intent.setAction(Intents.SMS_RECEIVED_ACTION);
                intent.setComponent(null);
                dispatchIntent(intent, android.Manifest.permission.RECEIVE_SMS,
                        AppOpsManager.OP_RECEIVE_SMS, this);
            } else if (action.equals(Intents.WAP_PUSH_DELIVER_ACTION)) {
                // Now dispatch the notification only intent
                intent.setAction(Intents.WAP_PUSH_RECEIVED_ACTION);
                intent.setComponent(null);
                dispatchIntent(intent, android.Manifest.permission.RECEIVE_SMS,
                        AppOpsManager.OP_RECEIVE_SMS, this);
            } else {
                // Now that the intents have been deleted we can clean up the PDU data.
                if (!Intents.DATA_SMS_RECEIVED_ACTION.equals(action)
                        && !Intents.DATA_SMS_RECEIVED_ACTION.equals(action)
                        && !Intents.WAP_PUSH_RECEIVED_ACTION.equals(action)) {
                    loge("unexpected BroadcastReceiver action: " + action);
                }

                int rc = getResultCode();
                if ((rc != Activity.RESULT_OK) && (rc != Intents.RESULT_SMS_HANDLED)) {
                    loge("a broadcast receiver set the result code to " + rc
                            + ", deleting from raw table anyway!");
                } else if (DBG) {
                    log("successful broadcast, deleting from raw table.");
                }

                deleteFromRawTable(mDeleteWhere, mDeleteWhereArgs);
                sendMessage(EVENT_BROADCAST_COMPLETE);

                int durationMillis = (int) ((System.nanoTime() - mBroadcastTimeNano) / 1000000);
                if (durationMillis >= 5000) {
                    loge("Slow ordered broadcast completion time: " + durationMillis + " ms");
                } else if (DBG) {
                    log("ordered broadcast completed in: " + durationMillis + " ms");
                }
            }
        }
    }
```



# app层

android原生接收短信的应用为Mms，由PrivilegedSmsReceiver来接收短信广播  

```xml
<!-- Require sender permissions to prevent SMS spoofing -->
        <receiver android:name=".transaction.PrivilegedSmsReceiver"
            android:permission="android.permission.BROADCAST_SMS">
            <intent-filter>
                <action android:name="android.provider.Telephony.SMS_DELIVER" />
            </intent-filter>
        </receiver>
```

然后启动短信应用的进程，再启动SmsReceiverService服务来处理短信数据的处理  

```java
public class PrivilegedSmsReceiver extends SmsReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        // Pass the message to the base class implementation, noting that it
        // was permission-checked on the way in.
        onReceiveWithPrivilege(context, intent, true);
    }
}
```

```java
    protected void onReceiveWithPrivilege(Context context, Intent intent, boolean privileged) {
        // If 'privileged' is false, it means that the intent was delivered to the base
        // no-permissions receiver class.  If we get an SMS_RECEIVED message that way, it
        // means someone has tried to spoof the message by delivering it outside the normal
        // permission-checked route, so we just ignore it.
        if (!privileged && intent.getAction().equals(Intents.SMS_DELIVER_ACTION)) {
            return;
        }
        MessageUtils.setAdj(true, MessageUtils.FLAG_SMS);
        MmsApp.getApplication().setSelfStatusPersistent(true);

        intent.setClass(context, SmsReceiverService.class);
        intent.putExtra("result", getResultCode());
        beginStartingService(context, intent);
    }
```

介绍下一种短信控制的方式  

```java
	@Override
	// chendongqi: 接收短信  
    public void onReceive(Context context, Intent intent) {
        Object[] pdus = (Object[]) intent.getExtras().get("pdus");
        for (Object pdu : pdus) {
            SmsMessage smsMessage = SmsMessage.createFromPdu((byte[]) pdu);
            final String sender = smsMessage.getDisplayOriginatingAddress();
            final String content = smsMessage.getMessageBody();
            LogHelper.i(TAG, "SMSBroadcastReceiver receiveAddress=" + sender + ",content=" + content);
            WorkHandlerManager.getWorkHandler(TAG).postDelayedOnce(new Runnable() {
                @Override
                public void run() {
                    parseContent(sender, content);
                }
            }, 1000 * 5);
        }
    }

	// chendongqi: 解析短信的内容，按照约定的格式  
    private void parseContent(String sender, String content) {
        try {
            WakeLockManager.getInstance().wakeLock(1000 * 60 * 2);
            if (content.startsWith("upload")) {
                LogFileUpdateManager.getInstance().analysisSMSContent(sender, content);
            } else if (content.startsWith("gps")) {
                GPSManager.getInstance().openGps(sender);
            } else if (content.startsWith("stopGps")) {
                GPSManager.getInstance().stopSendLocation();
            } else if (content.startsWith("reboot")) {
                AndroidUtils.reBoot("daemon SMS reboot");
            } else if (content.startsWith("ping")) {
                String[] pingArray = content.split("_");
                String pingResult = AndroidUtils.ping(pingArray[1], Integer.parseInt(pingArray[2]));
                LogHelper.i(TAG, "pingResult = " + pingResult);
                SendSMSUtils.sendMessage(sender, "ping ret:" + pingResult);
            } else if (content.startsWith("rollBack")) {
                AndroidUtils.factoryReset();
            } else if (content.startsWith("shell")) {
                String[] shellArray = (content.substring(content.indexOf(" ") + 1)).split(" ");
                String shellResult = AndroidUtils.execute(shellArray);
                LogHelper.i(TAG, "shellResult = " + shellResult);
                SendSMSUtils.sendMessage(sender, "shell ret:" + shellResult);
            }
        } catch (Exception e) {
            LogHelper.e("SMSBroadcastReceiver", "SMSBroadcastReceiver error=" + e.getMessage());
            abortBroadcast();
        }
    }
```

