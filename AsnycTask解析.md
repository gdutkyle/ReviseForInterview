#Android AsyncTask 解析  
###一 什么是AsnycTask  
<p>AsyncTask enables proper and easy use of the UI thread. This class allows you to perform background operations and publish results on the UI thread without having to manipulate threads and/or handlers.</p>  

这句话的意思是，我们在UI线程可以很简单、适用的调用AsyncTask。这个方法可以使我们不用调用Thread+handler的方式，就可以轻松的在后台线程做操作，然后把结果回调给UI线程  
<p>AsyncTask is designed to be a helper class around {@link Thread} and {@link Handler}
 * and does not constitute a generic threading framework. AsyncTasks should ideally be
 * used for short operations (a few seconds at the most.) If you need to keep threads
 * running for long periods of time, it is highly recommended you use the various APIs
 * provided by the <code>java.util.concurrent</code> package such as {@link Executor},
 * {@link ThreadPoolExecutor} and {@link FutureTask}.</p>  

同时，google也声明了，AsyncTask设计的初衷就是一个用来连接Threa和Handler的帮助类，而不是用于设计一个通用的线程框架，同时，AsnycTask应该用于那些后台操作在极短，最长时间不超过几秒的操作，如果我们需要在后台进行长时间的操作，更推荐使用{@link Executor},{@link ThreadPoolExecutor} and {@link FutureTask}（也即是自定义线程池）去定义自己的后台行为  

###二 几个重要方法和变量  

    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }

其中execute（Params...params）方法，调用的最终是回归到executeOnExecutor(Executor exec，Params... params)方法中，其中，传入的参数是sDefaultExecutor中，我们回归源码，看到这个sDefaultExecutor是什么？  


     /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */`    
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;`
好了，也即是说，sDefaultExecutor其实是一个一个串行执行的executor，如果你在使用execute(Params... params)的时候，执行的代码是：  

    new AsyncTask<Void, Void, Void>() {
            @Override
            protected Void doInBackground(Void... params) {
                Log.i(MainActivity.this.getClass().getSimpleName(),"execute time "+System.currentTimeMillis());
                return null;
            }

        }.execute();
        new AsyncTask<Void, Void, Void>() {
            @Override
            protected Void doInBackground(Void... params) {
                Log.i(MainActivity.this.getClass().getSimpleName(),"execute time "+System.currentTimeMillis());
                return null;
            }

        }.execute();  
那么，你的执行结果是这样的： 

    10-22 17:35:33.279 25394-25463/com.example.hao_wu.myapplication I/MainActivity: execute time 1508664933279
    10-22 17:35:33.280 25394-25466/com.example.hao_wu.myapplication I/MainActivity: execute time 1508664933280
所以，如果你需要一个并发执行的AsyncTask，那么你需要在execute执行的时候，指定需要调用的executor （AsyncTask.THREAD_POOL_EXECUTOR)  

        new AsyncTask<String, Void, Void>() {
            @Override
            protected Void doInBackground(String... params) {
                Log.i(MainActivity.this.getClass().getSimpleName(), "execute time " + System.currentTimeMillis());
                return null;
            }

        }.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, "");
        new AsyncTask<String, Void, Void>() {
            @Override
            protected Void doInBackground(String... params) {
                Log.i(MainActivity.this.getClass().getSimpleName(),"execute time "+System.currentTimeMillis());
                return null;
            }
  
        }.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR,""); 
  
这个时候，我们来看一下线程的执行情况：
  
    10-22 17:45:49.045 2254-2393/com.example.hao_wu.myapplication I/MainActivity: execute time 1508665549045
    10-22 17:45:49.045 2254-2395/com.example.hao_wu.myapplication I/MainActivity: execute time 1508665549045
综上所述，AsnycTask 提供了两个线程池，一个SERIAL_EXECUTOR，用于控制AsyncTask线程的串行执行，也可以说，AsnycTask用这个线程池来实现排队队列；另一个是AsyncTask.THREAD_POOL_EXECUTOR，用于真正的执行线程.
###三 言归正传  
AsyncTask为我们提供四个核心的回调方法，我们来看一下当我们初始化一个AsnycTask时，会给我们回调什么方法： 
  
        new AsyncTask<Void, Void, Void>() {

            @Override
            protected void onPreExecute() {
                super.onPreExecute();
            }
            @Override
            protected Void doInBackground(Void... params) {
                return null;
            }

            @Override
            protected void onPostExecute(Void aVoid) {
                super.onPostExecute(aVoid);
            }

            @Override
            protected void onProgressUpdate(Void... values) {
                super.onProgressUpdate(values);
            }
        }.execute();  
上面的回调方法中，我们着重看一下doInbackground（）方法，通过翻看源码我们可以看到， 

    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };

        ...
    }  
也即是说，当我们在线程开始执行的时候，会回调doInBackground方法给调用方，当方法结束的时候，AsnycTask通过postResult（result）方法，把结果通过handler回调给UI线程   

     private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
 AsnycTask定义了一个InnerHandler，去接受这个message  

    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }

而后呢，AsnycTask在finish方法中，把线程结束的信息回调给 onPostExecute（result）

    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
自此，AsnycTask中源码的调用流程就已经分析完了，以下是几个应该注意的地方：  

1 AsnycTask应该在UI线程中被初始化和使用，因为创建Handler对象时需要当前线程的Looper，所以我们为了以后能够通过sHandler将执行环境从后台线程切换到主线程，我们必须使用主线程的Looper，因此必须在主线程中创建sHandler。 

2 AsnycTask定义了两个线程池线程数，分别是：

    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    // We want at least 2 threads and at most 4 threads in the core pool,
    // preferring to have 1 less than the CPU count to avoid saturating
    // the CPU with background work
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
官方的解释是：
我们至少需要2个线程，最多需要4个线程在核心池中，采用比cpu数少一个的方式，避免后台工作的CPU饱和  
另外，定义的两个线程池数量的作用是：  
1、如果线程池的当前大小还没有达到基本大小(CORE_POOL_SIZE < MAXIMUM_POOL_SIZE)，那么就新增加一个线程处理新提交的任务；  
2、如果当前大小已经达到了基本大小，就将新提交的任务提交到阻塞队列排队，等候处理；   
3、如果队列容量已达上限，并且当前大小CORE_POOL_SIZE没有达到MAXIMUM_POOL_SIZE，那么就新增线程来处理任务；  
4、如果队列已满，并且当前线程数目也已经达到上限，那么意味着线程池的处理能力已经达到了极限，此时需要拒绝新增加的任务
