# JAVA ArrayList 源码分析  
## 前言  
前面我们已经分析了hashMap，在这里也简单的介绍一下我们在java开发中常用的另一个数据结构：ArrayList。ArrayList比HashMap要来的简单的多，HashMap底层是数组+链表+红黑树，而ArrayList的底层其实就是一个数组而已，核心算法其实就是如何扩容，也即是如何维持这个数组的大小。  
## ArrayList 初始化  
ArrayList提供了三个实例化的方法，分别是  
### 1 ArrayList()  

    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
无参的构造方法，是初始化一个空的数组，我们可以看到`private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}` 表明DEFAULTCAPACITY_EMPTY_ELEMENTDATA是一个空的数组，这个数组是为了当ArrayList的第一个元素被添加的时候，可以和`private static final Object[] EMPTY_ELEMENTDATA = {};`做出区分。  

### 2 ArrayList(int initialCapacity)  

    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
这个构造方法是我们指定了初始化容量的大小。我们通过代码可以看到，当我们传入的初始化值大于0的时候，以传入的值未容量大小，如果传入的值=0，那么我们就初始化一个空的数组  
### 3 ArrayList(Collection<? extends E> c)  

    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }  
ArrayList还支持初始化的时候，就传入集合参数。当传入的集合的大小>0的时候，就初始化一个和传入的集合类一样大的数组，并把传入的参数的值赋给ArrayList中。如果传入的集合参数是一个空值，那么，还是初始化一个空的数组。  

## ArrayList的添加方法  

    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
add()方法是把每次添加进去的元素，都放到数组的最后一个位置。其中，add()方法最核心的就是扩容处理。我们跟进去代码  

     private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
我们可以得到两个重要信息：  

**1 当我们初始化的数组是个空数组的时候，最小化容量值minCapacity其实就是默认的容量值DEFAULT_CAPACITY=10，而如果我们初始化的时候，就已经指定了ArrayList的大小，那么minCapacity就等于指定的大小+1；**  

**2 通过这个最小的容量值和当前数组元素个数相比较，如果已经大于数组的实际容量的话，那么就要考虑扩容了，否则`elementData[size++] = e`就会抛出数组越界的问题。**  

### 扩容  

    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }  
扩容的步奏是：  

1 获取当前数组的长度，并且赋值成oldCapacity；  

2 生成newCapacity，大小等于oldCapacity + (oldCapacity >> 1);，也即是原来大小的1.5倍； 
 
3 比较。如果扩容后的数据没有最小的容量大，那么最小容量minCapacity就是我们当前的扩容后的大小；如果扩容后的大小比MAX_ARRAY_SIZE还要大，那么最小容量就是Integer的最大值；  

4 拷贝。把老的数组里的数据拷贝到扩容后的数组中  

上面的步奏就完成了整个扩容的过程，也即是ArrayList的add()方法里面的核心。  

## ArrayList的get()方法  

     public E get(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        return (E) elementData[index];
    }
get方法比较简答，返回返回数组指定位置的数据即可，不做分析  
 
## ArrayList的remove()方法  

       public E remove(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        modCount++;
        E oldValue = (E) elementData[index];

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }  
remove方法也即是删除指定位置的元素。首先把删除的位置后面的元素进行迁移，然后将删除的位置的元素置空，等待GC的回收`elementData[--size] = null;`  

## 结语  
至此，ArrayList源码分析到此结束，ArrayList的其他操作，也都是基于这几个基本的思想进行操作，不做赘述。ArrayList确实比hashMap要简单许多
