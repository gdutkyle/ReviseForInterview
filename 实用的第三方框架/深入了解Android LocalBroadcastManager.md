# 深入了解Android LocalBroadcastManager  
## 一 前言  
想写这篇文章，最大的原因就是前面写了一篇Android Handler机制分析[https://www.jianshu.com/p/08ee800fe70e](https://www.jianshu.com/p/08ee800fe70e)。然后有人问我，你了解安卓的LocalBroadcastManager吗？好吧，一脸懵逼的进来，一脸懵逼的出去。回去我们的工程中，发现LocalBroadcastManager在我们自己的项目中就有使用了，而且我还经常用到。。。痛定思痛，写下这篇文章，深入的剖析一下LocalBroadcastManager  
## 二 什么是LocalBroadcastManager  
LocalBroadcastManager 主要在你的进程中，用来注册和发送一个Intent。相对于全局的广播，LocalBroadcastManager有很多的优势：  
**1 安全性**   

1.1 使用localbroadcastmanager只会在你自己的app中传播数据，其他的app无法监听到你的广播，所以你无须担心隐私消息外漏   

2.2 其他app不能发送广播到的app中，因此你不用担心安全漏洞的问题 

**2 高效性：**localbroadcastmanager比全局的广播更加的高效   
## 三 LocalBroadcastManager的使用  
###第一步：自定义BroadcastReceiver  

      public static class MyBroadCardReveiver extends BroadcastReceiver {

        @Override
        public void onReceive(Context context, Intent intent) {
            String intentFilter = intent.getAction();
            if (intentFilter.equals(MainActivity.INTENT_FILTER_REFRESH_UI)) {
                Toast.makeText(context, "收到了广播", Toast.LENGTH_LONG).show();
            }
        }
    }
###第二步：注册Receiver  

    IntentFilter intentFilter = new IntentFilter();
    intentFilter.addAction(INTENT_FILTER_REFRESH_UI);
    LocalBroadcastManager.getInstance(this).registerReceiver(myBroadCardReveiver, intentFilter);  

LocalBroadcastManager 的注册需要传入自定义的**BroadcastReceiver** 和 **IntentFilter**。IntentFilter主要用来注册当前需要监听的action是什么。这个我们后面再讲  
###第三步：在取消监听Receiver：unregisterReceiver(myBroadCardReveiver)  
LocalBroadcastManager 的注册和监听一般是成对存在的。如果我们在Activity的onCreate()方法中注册监听Broadcast，那么我们就需要在onDestory()中进行反注册。如果我们不unregister，将会带来内存泄漏的问题 
 
    @Override
    protected void onDestroy() {
        LocalBroadcastManager.getInstance(this).unregisterReceiver(myBroadCardReveiver);
        super.onDestroy();
    }  
###第四步：发送localBroadcast  

     Intent intent=new Intent();
     intent.setAction(INTENT_FILTER_REFRESH_UI);
     LocalBroadcastManager.getInstance(MainActivity.this).sendBroadcast(intent);  
好了至此为止，我们已经知道了如何在我们的项目中使用localbroadcastmanager。接下来我们将进行源码的剖析  
## 四 源码剖析  
###4.1 localBroadcastmanager的初始化  

    public static LocalBroadcastManager getInstance(Context context) {
        synchronized (mLock) {
            if (mInstance == null) {
                mInstance = new LocalBroadcastManager(context.getApplicationContext());
            }
            return mInstance;
        }
    }

    private LocalBroadcastManager(Context context) {
        mAppContext = context;
        mHandler = new Handler(context.getMainLooper()) {

            @Override``
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    case MSG_EXEC_PENDING_BROADCASTS:
                        executePendingBroadcasts();
                        break;
                    default:
                        super.handleMessage(msg);
                }
            }
        };
    }
好吧，LocalBroadcastManager其实是一个**单例设计**，根据后面的初始化我们知道，其实我们在外面传入的context，到了localbroadcastmanager内部，用的是当前context所在的应用的上下文。这样是因为不造成内存泄漏，LocalBroadcastManager只持有当前app的上下文。同时我们看到，LocalBroadcastManager 在初始化的时候，还初始化了一个handler，这个handler是把结果抛向主线程的。所以，我们可以在LocalBroadcastManager接收回调的时候，进行UI的更新操作。  
###4.2 localbroadcastmanager的注册  

    public void registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
        synchronized (mReceivers) {
            ReceiverRecord entry = new ReceiverRecord(filter, receiver);
            ArrayList<ReceiverRecord> filters = mReceivers.get(receiver);
            if (filters == null) {
                filters = new ArrayList<>(1);
                mReceivers.put(receiver, filters);
            }
            filters.add(entry);
            for (int i=0; i<filter.countActions(); i++) {
                String action = filter.getAction(i);
                ArrayList<ReceiverRecord> entries = mActions.get(action);
                if (entries == null) {
                    entries = new ArrayList<ReceiverRecord>(1);
                    mActions.put(action, entries);
                }
                entries.add(entry);
            }
        }
    }
registerReceiver方法是为那些匹配了intentFilter的action注册的监听。  
我们可以看到，这个方法有两个重要的变量，一个是**mReceiver**：mReceivers是一个list，它是以Receiver当做key，filters当成value保存进list中。第二个是**mActions**：mActions保存着filter中添加进去的action，也就是我们上面的代码  

    intentFilter.addAction(INTENT_FILTER_REFRESH_UI);  
###4.3 localbroadcastmanager的反注册  

    public void unregisterReceiver(BroadcastReceiver receiver) {
        synchronized (mReceivers) {
            final ArrayList<ReceiverRecord> filters = mReceivers.remove(receiver);
            if (filters == null) {
                return;
            }
            for (int i=filters.size()-1; i>=0; i--) {
                final ReceiverRecord filter = filters.get(i);
                filter.dead = true;
                for (int j=0; j<filter.filter.countActions(); j++) {
                    final String action = filter.filter.getAction(j);
                    final ArrayList<ReceiverRecord> receivers = mActions.get(action);
                    if (receivers != null) {
                        for (int k=receivers.size()-1; k>=0; k--) {
                            final ReceiverRecord rec = receivers.get(k);
                            if (rec.receiver == receiver) {
                                rec.dead = true;
                                receivers.remove(k);
                            }
                        }
                        if (receivers.size() <= 0) {
                            mActions.remove(action);
                        }
                    }
                }
            }
        }
    }
从上面代码中，我们可以看到，LocalBroadcastManager尝试从mReceiver中移除当前的监听，如果这个监听没有被注册进去，那么就直接返回，否则的话，就进一步去那mAction中被添加进去的action，把他们一个一个找到，然后移除出去。  
###4.4 localbroadcastmanager发送广播  

    public boolean sendBroadcast(Intent intent) {
        synchronized (mReceivers) {
            final String action = intent.getAction();
            final String type = intent.resolveTypeIfNeeded(
                    mAppContext.getContentResolver());
            final Uri data = intent.getData();
            final String scheme = intent.getScheme();
            final Set<String> categories = intent.getCategories();

            final boolean debug = DEBUG ||
                    ((intent.getFlags() & Intent.FLAG_DEBUG_LOG_RESOLUTION) != 0);
            if (debug) Log.v(
                    TAG, "Resolving type " + type + " scheme " + scheme
                    + " of intent " + intent);

            ArrayList<ReceiverRecord> entries = mActions.get(intent.getAction());
            if (entries != null) {
                ArrayList<ReceiverRecord> receivers = null;
                for (int i=0; i<entries.size(); i++) {
                    ReceiverRecord receiver = entries.get(i);
                if (receivers != null) {
                    for (int i=0; i<receivers.size(); i++) {
                        receivers.get(i).broadcasting = false;
                    }
                    mPendingBroadcasts.add(new BroadcastRecord(intent, receivers));
                    if (!mHandler.hasMessages(MSG_EXEC_PENDING_BROADCASTS)) {
                        mHandler.sendEmptyMessage(MSG_EXEC_PENDING_BROADCASTS);
                    }
                    return true;
                }
            }
        }
        return false;
    }

sendBroadcast()方法是是一个异步的操作。LocalBroadcastManager会马上返回这个值，但是真正的操作要等待异步回调到handler我们才能真正的收到广播的回调。  

      if (!mHandler.hasMessages(MSG_EXEC_PENDING_BROADCASTS)) {
          mHandler.sendEmptyMessage(MSG_EXEC_PENDING_BROADCASTS);
      }
  
所以又回到了我们在localbroadcastmanager初始化方法，也即是初始化handler中。通过代码，我们可以看到，handler最后调用了executePendingBroadcasts()方法中  

     private void executePendingBroadcasts() {
        while (true) {
            final BroadcastRecord[] brs;
            synchronized (mReceivers) {
                final int N = mPendingBroadcasts.size();
                if (N <= 0) {
                    return;
                }
                brs = new BroadcastRecord[N];
                mPendingBroadcasts.toArray(brs);
                mPendingBroadcasts.clear();
            }
            for (int i=0; i<brs.length; i++) {
                final BroadcastRecord br = brs[i];
                final int nbr = br.receivers.size();
                for (int j=0; j<nbr; j++) {
                    final ReceiverRecord rec = br.receivers.get(j);
                    if (!rec.dead) {
                        rec.receiver.onReceive(mAppContext, br.intent);
                    }
                }
            }
        }
    }

最后的 `rec.receiver.onReceive(mAppContext, br.intent);`               表明广播已经回到了我们注册的Broadcast的onReceive()方法中。  
## 五：结语  
LocalBroadcastManager其实是一个很简单的工具方法，但是其中的设计各有精妙的地方，很值得我们去借鉴。