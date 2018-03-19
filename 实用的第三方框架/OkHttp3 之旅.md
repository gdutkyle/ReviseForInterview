# OkHttp3 之旅   #
## 一 为什么推荐使用Okhttp3？   
首先，我并不觉得OkHttp是一个网络框架。okhttp对标的，应该是HttpClient或者HttpURLConnection，okhttp应该是一种新的网络请求方法，而网络框架，应该是基于上面几个网络访问方式进行封装的。像volley（基于httpClient和httpURLConnection）或者retrofit2（基于OkHttp3）。  
好吧，扯远了~~  
我们为什么要使用OkHttp3作为我们新的网络请求方式？我认为有以下几点： 
 
**1 OkHttp支持SPDY、http2.0和https**  

**2 OkHttp3内置ConnectionPool连接池，可以实现多路复用（在spdy不可用的情况下使用）**   
 
**3 利用GZip压缩内容**  

**4 具备超时重连机制，调用方不用自己去进行自定义连接动作**   

**5 具备缓存机制，可以避免重复的网络请求**  

**6 使用简单，封装的很好，方便调用（这个吧，不知道算不算，哈哈哈）** 

## 二 OKHttp3.0集成  
集成方式还是比较简单的，我们只需要在我们的app的build.gradle中添加依赖  

    compile 'com.squareup.okhttp3:okhttp:3.10.0'

同步调用：  

    private void sendRequestSync() throws IOException {
        OkHttpClient mHttpClient=new OkHttpClient();
        Request request= new Request.Builder().url(ENDPOINT).build();
        mHttpClient.newCall(request).execute();
    }

异步调用：  

     private void sendRequestAsyn(){
        OkHttpClient mHttpClient=new OkHttpClient();
        Request request= new Request.Builder().url(ENDPOINT).build();
        mHttpClient.newCall(request).enqueue(new Callback() {
            @Override 
            public void onFailure(Call call, IOException e) {
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                String json=response.body().string();
            }
        });
    }
## 三 OkHttp3请求流程分析 ##
由于同步执行的代码比较简单直观，我们这里用异步请求的方式进行分析。首先我们把整个网络请求分为4个部分  
###3.1 OkHttpClient的初始化###
     OkHttpClient mHttpClient=new OkHttpClient();   

OKHttpClient其实是call的一个工厂类，OkHttpClient是用来发起http请求，并拿到http的response的处理类。因为每一个HttpClient都维护有自己的连接池和线程池，所以不建议在每一次请求的时候都去初始化一个OkHttpcliet，而是在我们的项目中，只维护一个单例类就可以。  
我们有两种方法去初始化一个OkHttpClient:  

方法一：直接new OkHttpClient()，创建一个默认的OkHttpClient；  

方法二：通过builder去创建:  

    public final OkHttpClient client = new OkHttpClient.Builder()

    .addInterceptor(new HttpLoggingInterceptor())

    .cache(new Cache(cacheDir, cacheSize))

    .build();  
###3.2 异步方法执行  ###

    mHttpClient.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                String json=response.body().string();
            }
        });
在这里，我们首先去查看newcall(request)方法  

    @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);}  
所以，我们知道，最终的enqueue方法是在realCall类中调用的。我们前去查看源码  

    public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    client.dispatcher().enqueue(new AsyncCall(responseCallback));}
从这段代码的最后一个client.dispatcher()，好了到这里，我们就需要去看看Dispatcher类了。  
###3.3：Dispatcher类###
Dispatcher是OkHttp的异步请求策略类，在这里，OkHttp3定义了多个重要的参数，分别是  

    public final class Dispatcher {
		//最大的请求数量
  		private int maxRequests = 64;
  		
		//每一个host的最大请求数
		private int maxRequestsPerHost = 5;

 		private @Nullable Runnable idleCallback; 
		private @Nullable ExecutorService executorService;

		//当前准备执行的队列
  		private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

		//当前正在执行的队列，包括那些尚未结束的请求
  		private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  		//同步执行的队列，包括还没有结束的请求
  		private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  		public Dispatcher(ExecutorService executorService) {
    		this.executorService = executorService;
 		 }

  		public Dispatcher() {
  		 }
	}
dispatch中查看enqueue的处理方式  

    synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
     	 runningAsyncCalls.add(call);
      	 executorService().execute(call);
    	} else {
      	 readyAsyncCalls.add(call);
       }
     }
也就是，当前的运行队列小于最大值maxRequest并且当前运行的每个host的请求小于host最大请求数的时候，我们就把当前的call加入到执行队列中，否则就加入到等待队列中，等待okHttp执行。那么，我们是在哪里执行这些队列中的请求的呢？答案是RealCall
###3.4：RealCall  
RealCall里面的内部类AsyncCall封装了异步执行下的execute()方法。话不多少，我们直接上源码  

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
这段代码，我们只关注两个核心的方法，一个是**getResponseWithInterceptorChain()**，另一个是**client.dispatcher().finished(this);**  

####3.4.1：getResponseWithInterceptorChain()源码如下：  

    Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  	}
我们从execute()方法可以知道，我们真正发出网络请求，获取response方法就是在这个方法中。我们一步一步分析：  
#####第一：多拦截器的构造。  
getResponseWithInterceptorChain()方法首先定义了一个拦截器list，从代码中我们可以看到，添加的顺序分别为：  

**1 移动端自定义的拦截器**：client.interceptors()（为什么自定义的拦截器要第一个添加，后面有介绍）  

**2 retryAndFollowUpInterceptor** 失败重试的拦截器  

**3 BridgeInterceptor**：官方解释，这是一个连接我们应用和网络的桥梁。首先通过用户请求创建一个网咯请求，接着把这个请求发出处理，最后把网络返回的值转换成用户所需要的值  

**4 CacheInterceptor**：缓存拦截器  

**5 ConnectInterceptor**：网络请求拦截器。用来跟服务端进行连接  

**6 CallServerInterceptor**：这个是我们网络请求的链式调用的最后一个拦截器，用来将数据丢到网络中进行处理  
  
#####第二：RealInterceptorChain 初始化，链式调用正式开始  

RealInterceptorChain类我们主要看**process()**方法。  

    public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  	}  
前面的异常处理逻辑我们一律不看。我们主要看这一段  

     // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);  
我们在前RealCall中AsynCall的execute()方法中，传入RealInterceptorChain初始化的时候，index是0的，也就是说，上面的interceptors.get(index)；其实等于interceptors.get(0)=client.interceptors()，也即是我们移动端自定义的拦截器。最后通过  

    Response response = interceptor.intercept(next);
我们可以看到，最后回到的，就是各个自定义拦截器中的intercept()方法中（各位如果不理解，可以去看看官方demo里面HttpLoggingInterceptor.java的实现）。  

    if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }  
这段代码就是为了回调前面我们调用enqueue的时候的listener，返回当前的网络请求是成功的，还是失败的。这个简单，暂且不做分析。  
###3.4.3：client.dispatcher().finished(this);  
    private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }

    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
     }
    }
我们可以看到，无论是同步还是异步执行网络请求，最终都会调用到这个finish方法中。这个就是OkHttp维护当前连接池的方法，每当一个请求结束后，都会把当前的call从当前的运行队列中移除。然后再执行promoteCalls()，把等待运行队列中的下一个请求放入运行队列中。  
## 四 总结   
自此，OkHttp3.0的执行流程分析到此结束。但是OkHttp的内容远不止这么复杂，接下来我会在下一篇文章，对OkHttp的其他技术细节结合源码进行详细分析。