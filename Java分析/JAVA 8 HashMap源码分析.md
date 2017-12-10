# JAVA 8 HashMap 源码分析
-----
## 一 什么是HashMap？  
HashMap 继承了AbstractMap,实现了Map接口，HashMap是根据key-value来存储数据的，HashMap其实就是一个数组+链表+红黑树组成的。因为HashMap在存储数据的时候，会先去计算key的hash值，因为一个相同的key值，所得的hash值应该是唯一的，所以我们可以很快的去定位到value的值。HashMap允许key和value都为null，但是key只能有一次为空，为什么呢？我们在下文中会进行分析。HashMap是非线程安全的，所以非常不建议在多线程中去操作hashMap，以为这样会导致数据不统一。如果一定要用到类似结构，可以使用SynchronizedMap。

## 二 从源码入手，开始出发。
### **Step 1：HashMap 的基本数据单元 Node**   

    /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

       ......
    }
从源码注释我们可以看出，这个是hash的基本节点。其基本数据单元是Node，我们来看Node的构造方法  

     Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
Node节点包含了当前节点的hash值也就是HashMap的下标。key-value，和当前下一个节点node的指向。这就可以说明，这个Node也是一个单链表。 
 
### **Step 2：几个重要变量**  

    /**
     * The next size value at which to resize (capacity * load factor).
     *O
     * @serial
     */
    int threshold; 
        /**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     */
    transient int modCount;/**
     * The load factor for the hash table.
     *
     * @serial
     */
    final float loadFactor;
    /**
     * The number of key-value mappings contained in this map.
     */
    transient int size;  
好了，其实这几个变量我们从它们各自的注释中，可以很容易的看出它们的作用：  
1 threshold：HashMap的阈值。threshold=capacity*loadFactor，其实hashMap有两个重要的所谓的容量，一个是HashMap的总容量(capacity，这个java8 给我们的默认初始值=16)，另一个就是HashMap的阈值threshold。当我们的数据了超过threshold的时候，hashMap就要考虑去进行重新扩容了  
2 modCount：modCount是记录了我们当前对HashMap进行了多少次的操作，这些操作是会对当前HashMap的内部结构产生变化的，比如增加、删除、扩容，如果只是单纯的修改value，是不会产生modCount变化的。那么，mod的作用是什么呢？其实是因为我们的HashMpa是线程不安全的，HashMap在进行使用遍历器进行数据操作或者查询的时候，回去将当前的modCount暂存起来mc，在操作完数据后，再去判断mc==modCount，如果两个值不相等，说明这段时间有进行的数据操作，那么HashMap就会抛出异常：ConcurrentModificationException（）。  
3 loadFactor：负载因子。HashMap的负载因子默认为0.75，这个是根据空间和时间效率平衡出来的一个结果，我们也可以自己修改这个值。如果在空间效率要求比较高，但是时间效率要求比较低的场景中，我们可以适当的提高loadFactor的值，如果在空间要求不高，但是时间要求比较高的情况下，我们可以适当的降低loadFactor的值
4 size：当前HashMap的node长度，也就是当前hashMap我们put了多少个值进去（这个解释是不是简单粗暴？）跟我们前面所说的什么capacity，threshold没什么关系  

### **Step 3：重要方法分析**  
 **1 hash(Object key)方法**


    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
计算key的Hash值是整个hashMap增删改查的最基本步奏。Hash()方法要求要有很高的离散性能，因为这样，我们就可以只通过hash值，就可以快速的定位当前key的value，不用再去便利链表或者红黑树。  
第一步：我们看方法实现，首先会去判断key是不是为空，如果key为空的话，那么就返回当前key的hash值为0。好了，这也就解答了我们最开始的一个问题：“HashMap允许key和value都为null，但是key只能有一次为空”。因为以后每设置一次key为null，实际上就是覆盖当前key为0的value
第二步：获取当前key的hashCode，hashCode()的方法是object实现的，在native中实现，这里不做分析  
第三步：将当前的hashCode右移16位，然后和当前的hashCode进行异或。
经过以上几个步奏，我们就可以得到当前key值所在的hash值，这个有什么用呢？我们在下文继续分析 
 
**2 put(K key, V value)方法**  

     /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
put()方法的注释是，将特定的key和value进行关联，如果HashMap已经存在了这个key，那么value将会进行替换  


     final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }  
好了，又要开始痛苦的撸源码了。  
**第一步**：如果table为空或者table的长度为0，就要开始扩容了。我们来看下扩容的算法resize()  

    /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

resize()的方法是用来初始化或者扩大table的size的（double）如果table是null，那么就以默认值进行初始化，还有，以为我们扩容的时候，是以2倍的方法进行扩容的，所以原来hashmap中个元素有可能会待在原来的index，或者是原来2的倍数的index上。好了，简单概括一下resize()机制  
由源码进行分析，首先，我们会把老的hashMap的一些重要属性暂存起来，然后判断  
1 oldcap>0? 如果>0并且oldCap已经比HashMap默认的最大capacity还要大，那么就把新的阀值threshold赋值为最大的阀值。如果不是，那么将新的阀值设置为老的阀值的两倍，并且将新的容量设置为老的容量额两倍，即  
newthreshold=2*oldthreshold；newcap=oldcap*2；  
2 oldcap<0 &&oldthre>0，那么newcap=oldThre；否则，newcap赋值为默认初始化值16，newThre默认为12，也即是16*0.75
这个时候，newThre和newCap已经赋值完毕，接下来就开始进行扩容后的元素搬迁了  
首先，开始循环遍历老的tab，找出每一个元素e，如果  

    if (e.next == null)
       newTab[e.hash & (newCap - 1)] = e;
也就是说，如果e没有和他发生碰撞的元素，那么，我们就可以简单的根据根据hash算法，找到它在新表中的位置，
e.hash & (newCap - 1)，就是前面求hash的重要原因了  
好了，如果当前节点e后面还有值，那么，我们就要判断这个值是不是在红黑树上，如果是的话，那么就要执行红黑树的插入或者翻转动作。  
如果不是在红黑树上，那就是在hashMap的单链表上，那么，如果e的hash&&oldcap==0，就是说明节点e需要待在原来的位置上，佛则，就是在原索引位置+oldcap的位置上  
好了，这个时候就把HashMap的resize()扩容机制分析完毕了。**接着回归Step2**  
**Step3 第三步**，寻找插入位置  
如果通过indexhash()方法，得到当前hash在table中没有元素，那么我们就可以直接把这个节点插入到这个位置  
如果这个位置已经有元素了，那么就要开始解决hash冲突了。
hashMap解决hash冲突有两种方法，一张是开放定位法，一种是链地址发。JAVA HashMap采用的是第二种方法。我们回到上面的源码，看具体实现  
第一步：比对key，如果两个key一样，说明这个位置的hashmap操作其实不是一个插入的操作，而是一个替换的操作。  
第二步：如果当前发生冲突的节点位于红黑树上，那么我们就执行红黑树的插入动作，让新的节点插入到制定的位置。  
第三步：如果冲突节点不在红黑树上，那么它就在链表上，那么我们就将这个节点插在单链表的第一个位置  
插入的最后，还是要继续判断插入后，会不会使当前map超过了制定的容量，如果超过了，就要继续扩容  
至此，HashMap的put流程已经分析完毕，可能还有很多看起来很晕的地方，建议大家自己再去撸一遍源码，可以更加清晰的知道整个工作流程。
 
**3 remove(Object key)方法**  
HashMap删除方法。我们下面来看下remove的源码分析  

    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
remove方法还是比较简单明了的，首先，在我们的hashMap不为空的情况下，我们根据key值去indexHash,找到当前key在hashMap对应的下标。再去HashMap中将对应的node取出来，如果当前Node的key和目标key一样，那么就执行删除操作，否则的话，就跟put流程一样，判断当前Node是不是在红黑树上，如果在红黑树上，那么就去执行红黑树的删除动作，否则的话，就去便利单链表，直到找到key一样的node，将这个node删除掉。  
**3 get(Object key)方法**  
get方法其实和remove的流程其实差不多，这里不再进行分析，有需要的朋友可以自己去查看源码。  

## 三 总结  
自此，HashMap的源码分析到此结束了，我们需要特别注意一下几点：  
1 hashMap是线程不安全的，如果需要在多线程下执行同一个hashMap,ConcurrentHashMap();  
2 JAVA 8 在处理Hash冲突的时候，引用了红黑树。我们知道，红黑树的插入动作虽然很麻烦，但是他的时间复杂度其实可以控制在n（Log n）上，所以，性能会十分的客观。而且，老的使用数据扩容的方法，当数据量大的时候，会时时间性能呈现指数增长的过程  
3 扩容是一个很耗费性能的过程，所以大家在使用的时候，可以最先就预估map的大小，避免hashMap需要频繁的resize()  
##四 参考  
http://www.importnew.com/20386.html

