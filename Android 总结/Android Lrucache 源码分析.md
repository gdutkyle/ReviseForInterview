# Android Lrucache 源码分析 #
___
# 一 什么是LruCache？ #  
我们来看官方的解释，Lrucache是一个持有有限数量强引用对象的缓存。每当一个新的对象到来的时候，lrucache会把它移动到队列的头部。当对象到来的时候，如果这个队列已经满了，那么在队列尾部的对象就会被弹出，然后变成可以被GC回收的对象。  
lrucache是线程安全的，并且Lrucache不允许value或者key是null，否则会返回the key was not in the cache。  
# 二 开始进入正题 #  
**1 Lrucache初始化** 

     public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }

从这里我们可以看到，Lrucache包含了一个LinkHashMap，为什么不用hashMap？因为在开头我们说到，Lrucache是一个线程安全的cache，二hashMap是线程不安全的，所以只能采用LinkHashMap。  
**2 Lrucache的Put流程**  

    public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;
            size += safeSizeOf(key, value);
            previous = map.put(key, value);
            if (previous != null) {
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, value);
        }

        trimToSize(maxSize);
        return previous;
    }
lrucache的put流程就是简单的把一个value移动到队列的开头，value和key是一一匹配的。我们接下来分析：  
1 判断key或者value是否为空，为空的话，那么就跑出异常  
2 定义一个value，赋值为previous，这个值表示当前key，是不是有值在cache中了  
3 通过map.put(key,value)，如果有值，那么previous返回的是原先的值  
4 如果previous不为空，那么就给出一个entryRemoved的回调  
5 调用trimToSize（maxisize）方法，按照情况判断是否需要弹出元素  
**3 trimToSize（maxisize）方法**  

     public void trimToSize(int maxSize) {
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize) {
                    break;
                }

                Map.Entry<K, V> toEvict = map.eldest();
                if (toEvict == null) {
                    break;
                }

                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }
这方法的执行步奏是，如果当前的size比maxsize还要小，那么就继续填加元素。否则，就找出map的最后一个元素，如果他不为空的时候，就把元素弹出，并且把当前的map的size重新复制  

    private int safeSizeOf(K key, V value) {
        int result = sizeOf(key, value);
        if (result < 0) {
            throw new IllegalStateException("Negative size: " + key + "=" + value);
        }
        return result;
    }
 
    protected int sizeOf(K key, V value) {
        return 1;
    }
在默认情况下，sizeof（）方法默认返回的是1，表示当前size是元素的值，maxsize是最大的元素值  
**4 lrucache的get()流程**  

    public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            missCount++;
        }

        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

        synchronized (this) {
            createCount++;
            mapValue = map.put(key, createdValue);

            if (mapValue != null) {
                // There was a conflict so undo that last put
                map.put(key, mapValue);
            } else {
                size += safeSizeOf(key, createdValue);
            }
        }

        if (mapValue != null) {
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
            trimToSize(maxSize);
            return createdValue;
        }
    }

这个方法的大致作用是，通过get方法会根据key返回对应的value，如果这个value被返回，那么这个元素将会被移动到队列的头部。当value没有被缓存或者没有被创建的时候，就会返回null。  
好了第一步：

     mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
如果通过map可以找到对应的值，那么就直接返回map中的value；否则，调用create（key）方法。create（key）方法可能会花费很多的时间，同时，在create方法执行的时候，map可能已经发生了变化。所以当一个key已经被放进map中的时候，我们以map中的value为准，释放掉create（key）方法中创建的值。  

那么什么时候会去调用oncreate（）方法呢？其实oncreate（）方法是需要我们去重写的。如果没有重写oncreate方法，那么LruCache默认返回的是0，整个流程就返回了。  
好了，我们现在假设我们重写的onCreate（）方法。流程往下走。我们将createVaule put进map中，发现如果有冲突了，那么说明我们这个key在我们创建的时候，别的线程已经赋值了，那么我们就回撤这个行为，将mapvalue重新put进去map中，也就是上面所说的以map中的value为准。如果put进去map成功了，那么我们就需要走上面的put内容，重新走trimToSize（）方法。  
**4 lrucache的remove()流程** 
其实这个流程比较简单就是简单的map的remove,话不多说上源码：

     public final V remove(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V previous;
        synchronized (this) {
            previous = map.remove(key);
            if (previous != null) {
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, null);
        }

        return previous;
    }
根据keyremove对应的value，同时把size-safeSizeOf（key，previous）的值。如果为空，那么返回null。  
# 三 备注 #
至此整个LruCache算法介绍完毕。LruCache算法更多的用于设计图片加载框架，这个我们以后再谈。。。