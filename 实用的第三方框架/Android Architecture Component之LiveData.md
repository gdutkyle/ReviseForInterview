#Android Architecture Component之LiveData 分析#
### 一 什么是LiveData
LiveData是一个可观察数据变化的类，但是与其他观察类不同的是，LiveData是可以感知到所持有类的生命周期的。所以我们一般用它来和Activity、Fragment或者service一起使用。这里有个很有好处的地方就是，LiveData的onChange方法，只会在activity在前台的时候产生回调，后面我们可以用例子来说明。
### 二 LiveData的好处
那么使用LiveData有什么好处呢？从LiveData的相关特性，我们可以归纳出以下几点：  

**1 确保UI和数据状态保持一致**  
LiveData遵循观察者模式，当生命周期发生改变的时候，LiveData会通知观察者对象。因此我们可以通过统一在观察者对象数据中更新UI，而不是通过app数据的变更去改变UI。在我们这些对象发生变更的时候，我们的UI也会跟着改变。  

**2 不会有内存泄漏**  
所有的观察者都是绑定到owner的生命周期内的，也就是说，如果owner被finish的时候，所有的观察者也会被自动销毁。炒鸡智能方便有木有？  

**3 不会因为Activity被销毁而导致崩溃**  
这种情况多数发生在我们的异步回调函数中，比如有时候我们回去通过异步线程来获取网络数据后，刷新我们的界面。这个时候，当我们的activity被销毁的时候，我们的异步线程回调才刚回来，这就导致了我们在一个废弃的activity中更新他的UI，这就会导致异常崩溃的发生。 

**4 不再需要手动去控制生命周期**  
UI组件只会在对象不是stop或者resume的时候进行观察，LiveData会自动管理这一切。在正确的时候，给与观察者以回调。  

**5 永远保持最新的数据状态**  
当观察的对象不是处于激活状态的时候，LiveData并不会去刷新数据，而当观察的对象重新激活的时候，那么LiveData将会刷新最新的数据。最典型的是，如果你的Activity是后台的时候，再次期间所有的数据变更，liveData都会选择性的忽略，等到你重新唤醒Activity的时候，LiveData马上就会刷新他的值，让他和最后一个setValue（value）保持一致。  

### 三 开始源码之旅  
接下来，我们从一段简单的源码开始  

    viewModel.loadFiveUsers().observe(MainActivity.this, new Observer<List<User>>() {
            @Override
            public void onChanged(@Nullable List<User> users) {
                StringBuilder sb = new StringBuilder();
                for (int i = 0; i < users.size(); i++) {
                    User user = users.get(i);
                    sb.append("====================\n");
                    sb.append("uid:" + user.getUid() + "\n " + user.getUserName() + "\n" + user.getLastName() + "\n");
                    sb.append("\n");
                }
                tv_values.setText(sb.toString());
            }
        });
  我们这里暂且忽略viewModel部门，我们只需要知道，viewModel.loadFiveUsers()返回的是一个LiveData<T>类型的数据就可以了。简单的说一下这段代码，就是实时获取当前数据库中的前5条数据，并且把这5条数据通过TextView显示出来。业务场景其实非常的简单。好了，接下来开始拆轮子  
  
第一步：LiveData.observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer)Owner和observe是如何建立联系的  
我们进入源码的世界  

     @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            // ignore
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && !existing.isAttachedTo(owner)) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        owner.getLifecycle().addObserver(wrapper);
    }
首先，LiveData强调的是，observe方法必须是在UI线程中进行调用的。我们从源码可知  
1 如果当前的owner是Destroyed状态的，那么我们就直接忽略掉了，这个是显而易见的。  
2 LiveData会先将当前的owner和observe结合在一起，生成一个wrapper对象，然后尝试observe和wrapper当成key和value 添加进mObservers中，这个mObservers其实就是一个map对象。如果发现这个key对应的value已经有值了，并且这value并不是我们即将put进去的value，那么就会抛错  
3 加入不存在的话，那么wrapper将会被添加进当前owner的生命周期中，进行监听。

第二步：setValue(T value) 
LiveData提供了两个setValue的方法，一个是同步情况的调用  

    @MainThread
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }  
另一个是异步情况的调用  

    protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
       postTask = mPendingData == NOT_SET;
        mPendingData = value;
       }
       if (!postTask) {
           return;
       }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }  
其实最后都是调用了 setValue((T) newValue);方法。我们来研究下这个方法，可以看到，他会先判断当前是否在主线程调用，如果不是的话，那么就会抛出错误。然后就会把当前LiveData的版本号mVersion++,把mData设置成最新的value。其实整个过程最麻烦的是dispatchingValue(null)这个方法。我们进入这个方法进行查看  

    private void dispatchingValue(@Nullable ObserverWrapper initiator) {
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                considerNotify(initiator);
                initiator = null;
            } else {
                for (Iterator<Map.Entry<Observer<T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                    considerNotify(iterator.next().getValue());
                    if (mDispatchInvalidated) {
                        break;
                    }
                }
            }
        } while (mDispatchInvalidated);
        mDispatchingValue = false;
    }

其实我们可以看到，这个个方法就是现在已有的观察者进行循环，检查到如果是当前的观察者，那么就调用considerNotify(iterator.next().getValue())方法。也即是找到最新的mData，进行onchanged(mData)回调给观察者。  

     private void considerNotify(ObserverWrapper observer) {
        if (!observer.mActive) {
            return;
        }
        // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
        //
        // we still first check observer.active to keep it as the entrance for events. So even if
        // the observer moved to an active state, if we've not received that event, we better not
        // notify for a more predictable notification order.
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
        if (observer.mLastVersion >= mVersion) {
            return;
        }
        observer.mLastVersion = mVersion;
        //noinspection unchecked
        observer.mObserver.onChanged((T) mData);
    }

其实这个方法，会调用到 

     @Override
      boolean shouldBeActive() {
          return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
      }
这个方法就是会去判断当前的owner的状态是否是active状态，进行是否分发的动作。  

### 四 生命周期中LiveData的回调情况
**1 关于：永远保持最新的数据状态**  

我们主动setValue（T value）方法后，liveData调用的onChanged（T value）流程已经分析完毕了，那么前面我们所说的，**当观察的对象不是处于激活状态的时候，LiveData并不会去刷新数据，而当观察的对象重新激活的时候，那么LiveData将会刷新最新的数据。最典型的是，如果你的Activity是后台的时候，再次期间所有的数据变更，liveData都会选择性的忽略，等到你重新唤醒Activity的时候，LiveData马上就会刷新他的值，让他和最后一个setValue（value）保持一致**。这个是怎么实现的呢？  
![LifecycleBoundObserver 生命周期回调](https://i.imgur.com/8SlRzsq.png)  
其实就是如上图所说的，我们的activity在后台或者重新回到前台的时候，都会回调到上面的onStateChanged（）方法。当Activity是后台的时候，shouldBeActive返回的是false，那么  

    void activeStateChanged(boolean newActive) {
            if (newActive == mActive) {
                return;
            }
            // immediately set active state, so we'd never dispatch anything to inactive
            // owner
            mActive = newActive;
            boolean wasInactive = LiveData.this.mActiveCount == 0;
            LiveData.this.mActiveCount += mActive ? 1 : -1;
            if (wasInactive && mActive) {
                onActive();
            }
            if (LiveData.this.mActiveCount == 0 && !mActive) {
                onInactive();
            }
            if (mActive) {
                dispatchingValue(this);
            }
        }
中的dispatchingValue（this）就不会进行分发，从而不会触发observe的onchanged(T mData) 方法。

**2 Activity finish的时候，移除监听**  

上面的图中我们可以看到，LiveData还会去判断当前的owner是否已经被销毁了，如果被销毁了，还会调用removeObserver(mObserver);方法，这也就是我们不需要去关心内存泄漏，或者在Activity finish情况下，还去尝试更新UI的问题。  

自此，LiveData的完整分析已经完成。

