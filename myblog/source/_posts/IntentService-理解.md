---
title: IntentService 理解
date: 2018-08-16 14:49:19
tags: Android
toc: true
---

# IntentService
**IntentService用于需要执行异步操作的Service,当异步操作执行完毕时会停止这个Service**

## 代码分析
```
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper; // 工作线程 looper
    private volatile ServiceHandler mServiceHandler; // 工作handler
    private String mName;
    private boolean mRedelivery; // 恢复类型

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj); // 消息不处理 直接发送过去
            stopSelf(msg.arg1); //上面的方法执行完毕,停止这个service
            // 由上可知,onHandleIntent方法中不能再写异步方法,否则可能直接停止service
        }
    }


    public IntentService(String name) {
        super();
        mName = name;
    }

    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        // thread中使用handler必须要有looper,HandlerThread就是简化创建thread和looper关系
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start(); // 创建循环looper,创建成功后会notifyAll();

        mServiceLooper = thread.getLooper(); // 获取looper,当looper==null时wait(),等待start创建成功后的notifyAll再返回looper
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
         // onStart时给handler发送msg
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        // 第一步调用onStart
        // 第二步设置恢复模式
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    @Override
    public void onDestroy() {
        // 关闭looper,相当于不循环取消息了.这一套流程也就停止了.
        mServiceLooper.quit();
    }

    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }

    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}

```

### IntentService$onCreate
1. 创建一个`HandlerThread`,并且`name`为`"IntentService[" + mName + "]"`
2. 调用`HandlerThread.start`,
    1.  执行了`HandlerThread`中的`run`方法
    2.  `looper`创建成功后会`notifyAll()`
    3.  执行循环`looper`
3. ServiceHandler(thread.getLooper())
    1. `getLooper`中,当`thread`活着 && `mLooper==null`时,`wait()`
    2. `wait()`是为了等待`looper`创建成功,执行`notifyAll()`,这样`mLooper`就有值了.

### IntentService$onStartCommand
1. 调用`onStart`
2. `handler.sendMessage(msg)`
3. `ServiceHandler$handleMessage`
4. `onDestroy`调用`looper.quit`

### ServiceHandler$handleMessage
`onHandleIntent`中不能做异步处理,否则会直接执行`stopSelf`

### onStartCommand的恢复模式
- `START_REDELIVER_INTENT`: 当service在启动时被杀死,将会被重新启动,知道service调用`stopSelf`
- `START_NOT_STICKY`: 当service在启动时被杀死,不做处理.
