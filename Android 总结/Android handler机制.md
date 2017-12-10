## Android handler机制  
### 一 什么是Android Handler 机制？
很多的博客书籍，在介绍Android handler机制的时候，更多的是解释为，Android提供的，子线程用于和UI线程通讯，用于更新主线程UI。但是这种异步处理回调机制，是否仅仅只是用于线程间的通讯？我们能不能用这种回调思想，用于我们的业务中？是我们一个值得去思考的地方。
###二 Handler机制中几个重要的组成部分  
**1 Looper**  

looper主要用于和当前的线程绑定，保证一个线程只有一个looper对象，并且只有一个messageQueue。looper在Handler机制中，扮演着重要的作用。looper在初始化的时候，就生成了messageQueue，开启一个死循环去遍历这个messageQueue，当获取到一个message对象的时候，message.target.dispathmessage()方法，将message发送给对应的Handler处理  

**2 Message**  

Message是子线程和主线程联系的单位。我们在生成一个message对象的时候，可以通过
Message msg=new Message()方式去获取一个message对象，也可以通过
Message msg=handler.obtainMessage()方法获取，但是我们更加推荐的是使用后者去获取一个Message对象。从源码中  

    /**
     * Return a new Message instance from the global pool. Allows us to
     * avoid allocating new objects in many cases.
     */
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
我们可以看出，通过handler的obtain方法获取的message对象，是在一个可回收利用的对象池中获取Message的，这样可以避免我们每次都去new一个对象，造成浪费。  
在1中，我们还有一个问题是，Message是怎么知道自身对应的是哪个Handler呢？
enqueueMessage中首先为message.target赋值为this，也就是把当前的handler作为msg的target属性。最终会调用queue的enqueueMessage的方法，也就是说handler发出的消息，最终会保存到消息队列中去。而message.target.dispathmessage()方法，最后就是回调到我们的handler的onHandlerMessage方法中。  

**3 Handler**  

Handler的作用就是将message对象压入messageQueue中，然后通过looper不停的从消息队列中取出message，并作出对应的处理。因此，Handler有两个主要的功能，分别是：
1 定时执行messages 和 runnables；
2 在将一个action入队并在其他线程中执行；  
### 三 总结  
简单来说，Android handler机制就是在子线程中，handler把一个消息对象压入消息队列中，然后，在handler对应的线程中，通过looper循环队列，取出对应的message并作出相对应的处理，完成了两个线程间的通讯。 

### 四 引申  

前文说到，Android Handler机制我们可以按照这种思想来实现什么业务呢？  
1 即时通讯的消息处理。我们可以在发消息的时候，把一个消息体放到消息队列中，然后在后台中开启一个循环去遍历这个消息队列，然后调用发送消息接口把这个消息发送出去，这样，我们就可以在不堵塞主线程的情况下，完成消息的发送处理。  

2 图片框架处理。简单的来说，我们可以开启多个线程去处理我们的多图片业务，然后通过handler回调已经完成的图片处理给UI线程，展示图片
