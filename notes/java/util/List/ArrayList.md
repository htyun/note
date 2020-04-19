# Java 8 ArrayList 源码简单分析

## 前言

```text
本人从学校毕业也快三年了，加上学校工作室和实习也算工作了快五年的码农了。平时比较喜欢专研一些新的技术或者是主流技术，前两年是什么都想学，导致接触面广，但是在深度上有些欠缺，所以后来思考一番后，觉得需要专注某一个领域，万金油很多，某个领域专业型的人才还是不多，不知道我能否达到那个层次，但是我还是想那方向去努力。

虽然之前会去阅读源码和 Spring 官方文档，但是都是一扫而过，没有记录下来。倒是记录了一些中间件服务器部署的操作过程。这还是第一次如此比较深入的分析源码，也是第一次写出来。各位发现有错的或者写的不好的，欢迎指出来，共同学习共同成长。
```

## extends And implements

> 继承 **AbstractList** 实现集合属性和基本的操作
>
> 实现 **List** 接口，覆盖掉 **AbstractList** 中的一些实现
>
> 实现 **RandomAccess** 接口，实现此接口有随机读写功能
>
> 实现 **Cloneable** 接口，克隆功能
>
> 实现 **java.io.Serializable** 接口，序列化

## 初始化filed

> **private static long serialVersionUID;** 序列化id，保证反序列化也是此类
>
> **private static final int DEFAULT_CAPACITY = 10;** 默认的数组长度
>
> **private static final Object[] EMPTY_ELEMENTDATA = {};** 空的数组实例
>
> **private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};** 空的数组实例
>
> > **EMPTY_ELEMENTDATA** 和 **DEFAULTCAPACITY_EMPTY_ELEMENTDATA** 的区别 在于操作 **elementData** 的时候知道对象是有参构造方法还是无参构造方法初始化的
> >
> > **DEFAULTCAPACITY_EMPTY_ELEMENTDATA**  标记的 **elementData** 会使用 **DEFAULT_CAPACITY** 变量进行初始化，或者扩容判断操作
>
> **transient Object[] elementData;** 存储元素的实例数组
>
> > 如果是使用 **DEFAULTCAPACITY_EMPTY_ELEMENTDATA** 初始化实例数组，则使用 **DEFAULT_CAPACITY** 默认值来初始化数组大小
>
> **private int size;** ArrayList 的大小( ArrayList 包含的元素的数量)
>
> **private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;** 要分配的数组的最大大小,分配过大的数组大小可能会导致 OutOfMemoryError

```tex
注意：这里的所有数组都是 Object 类型的
```

## 构造方法

```java
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```

- `无参构造方法 elementData 用 DEFAULTCAPACITY_EMPTY_ELEMENTDATA 是标记作用，主要是为了能够用到 DEFAULT_CAPACITY  值。下面会讲到`

```java
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
```

- `有参构造方法，initialCapacity == 0 时，就会用 EMPTY_ELEMENTDATA 初始化 elementData 。EMPTY_ELEMENTDATA 作为标记，认为 ArrayList 对象是通过有参构造方法实例化`

```java
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

- `c.toArray() 是复制出来的对象。`
- `这里有一个转换，length != 0 将 elementData 转为 Object[]，同样通过有参构造方法实例化 ArrayList ，如果 length == 0 使用 EMPTY_ELEMENTDATA 赋值标记`

## ArrayList 函数

### **trimToSize**

```java
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
```

- `听名字就知道是缩减 size，和 String.trim() 类似，这里是将 elementData 长度缩小到最小有效长度。`
- `modCount 是用来记录被操作次数，EMPTY_ELEMENTDATA 就和上面提到的是作为标记`

### **ensureCapacity**

```java
    // 确保 ArrayList 能够容纳所存储的数据
    // 这个方法目前只在 com.sun.corba.se.impl.orbutil.DenseIntMapImpl 看到有调用 list.ensureCapacity( index + 1 );
    public void ensureCapacity(int minCapacity) {
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            // 其他的则可以直接通过 minCapacity > minExpand 条件
            ? 0
            // DEFAULTCAPACITY_EMPTY_ELEMENTDATA 标记的 elementData，将使用默认大小进行初始化
            : DEFAULT_CAPACITY;

        // 最小容量大于最小扩展（除非是当前 elementData 是使用 DEFAULTCAPACITY_EMPTY_ELEMENTDATA 标记，说明它现在还是空的数组）
        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // minCapacity 需要承载数据的容量的大小 > 当前数组大小，则进行扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    // 扩容
    // 根据入参方向看出 minCapacity 通常接近于 size
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        // 将容量扩大为 <= 1.5 倍
        // 简单的例子，如果当前二进制容量是 1100 >> 1 为 0110，是为原来的 1/2，但是如果二进制的最后一位为1， 1101 >> 1 = 0110， 最后一位丢失，所以会比原来的 1/2 小
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // oldCapacity 已经很大的时候，再扩大就会超出有符号int的最大正数值，最左位变成 1 就为负数
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // 最大承载值矫正
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0)
            throw new OutOfMemoryError();
        // 最大只能达到 Integer.MAX_VALUE，也就是 0x7FFFFFFF, 二进制为 ‭0111 1111 1111 1111 1111 1111 1111 1111，有符号的最大int值‬。MAX_ARRAY_SIZE 上面有列出来 Integer.MAX_VALUE - 8
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

- `以上是 ArrayList 扩容的过程`

### **size** and **isEmpty**

```java
    // 返回当前数组元素个数
    // size 是使用基本数据类型定义的，默认值是 0
    public int size() {
        return size;
    }

    public boolean isEmpty() {
        return size == 0;
    }
```

### contains

```java
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }

    public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```

- `这里是会返回 -1的，使用的时候要注意，lastIndexOf 和 indexOf 刚好便利的顺序是相反的`

### copy

```java
    public Object clone() {
        try {
            ArrayList<?> v = (ArrayList<?>) super.clone();
            v.elementData = Arrays.copyOf(elementData, size);
            v.modCount = 0;
            return v;
        } catch (CloneNotSupportedException e) {
            // 正常来说这个是不会发生的，只有可能是底层 copy 时出现问题了
            throw new InternalError(e);
        }
    }

    // copyOf 使用的 System.arraycopy 进行数组复制
    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }

    public <T> T[] toArray(T[] a) {
        if (a.length < size)
            // 注意这里，使用 Arrays.copyOf ,a.length < size 时 Arrays.copy 会新建一个数组对象，所以这里返回的对象不是原来的 a, 且会丢弃掉大于 a.length 之后的数据
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        // 这里返回的是 a
        System.arraycopy(elementData, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }
```

- `以上是 ArrayList 复制数据的过程`

### get

```java
    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }

    // 这里并没有判断 index < 0 的情况
    // 因为在 elementData() 方法中如果 index 是 < 0 会抛出 ArrayIndexOutOfBoundsException
    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    E elementData(int index) {
        return (E) elementData[index];
    }
```

### ArrayList 数据操作

#### set

```java
// set

    // 简单的数组操作，返回原来这个坑位的元素
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```

#### add

```java
// add

    public boolean add(E e) {
        // 先进行扩容操作，如果有操作会 增加 modCount
        ensureCapacityInternal(size + 1);
        elementData[size++] = e;
        return true;
    }

    public void add(int index, E element) {
        rangeCheckForAdd(index);
        // 先进行扩容操作，如果有操作会 增加 modCount
        ensureCapacityInternal(size + 1);
        // 将数组从 index 位置开始后移，将 index 这个坑空出来
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }

    // addAll 重载方法
    public boolean addAll(Collection<? extends E> c) {
        // 记住，toArray 时copy不同的对象的，所以 c 改变它的 elementData 坑的指向，是不会影响到当前 list 的
        Object[] a = c.toArray();
        // 被加进来的集合的长度
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }

    // addAll 重载方法，此方法就比 addAll(Collection) 多了一步，从 index 后移 c.size 位，就是将 size - index 个元素往后移 c.size 位
    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);

        int numMoved = size - index;
        if (numMoved > 0)
            // 从 index 到 size 有 numMoved 个元素，复制到新的数组，新数组从 index + c.size 开始插入
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);
        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }

    // 这里对 index 进行 < 0 判断是因为 add(int, E) 方法会先进行数组复制，如果 index < 0 会有 ArrayIndexOutOfBoundsException，导致数组复制失败
    private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }

    private void ensureCapacityInternal(int minCapacity) {
        // minCapacity 是认为当前数组最小能够承载数据的最小值
        // 调用 add() 方法，DEFAULTCAPACITY_EMPTY_ELEMENTDATA 标记的 elementData 会使用 DEFAULT_CAPACITY 默认值来进行扩容
        // 这里为什么要用 Math.max 得到最大值呢，都已经是一个空数组了，minCapacity 应该会 < DEFAULT_CAPACITY.
        // 因为当前这个方法在 addAll() 也有调用，this.elementData 被 DEFAULTCAPACITY_EMPTY_ELEMENTDATA 标记，但是 addAll() 传入的对象 length 是有可能 > DEFAULT_CAPACITY
        // 非 DEFAULTCAPACITY_EMPTY_ELEMENTDATA 标记的 elementData 是通过有参构造方法初始化的，有初始容量，所以不使用 DEFAULT_CAPACITY
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        // 扩容
        ensureExplicitCapacity(minCapacity);
    }
```

#### remove

```java
// remove

    // remove 有两个重载方法，参数为 int 返回的是被删除的对象
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        // 需要移动坑的元素个数
        int numMoved = size - index - 1;
        if (numMoved > 0)
            // index 这个坑位后面的元素全部往前移
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        // 置为 null 可以释放内存空间，GC可以回收这个占着茅坑的对象
        elementData[--size] = null;

        return oldValue;
    }

    // remove 有两个重载方法，参数为 Object 返回的是 boolean 类型
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    // 删除本集合中存在 c 集合中的元素
    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    }

    // 保留本集合中存在 c 集合中的元素
    public boolean retainAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, true);
    }

    // complement 决定是删除还是保留
    private boolean batchRemove(Collection<?> c, boolean complement) {
        // 定义局部数组可以减少本对象的寻址
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        // 标记是否发生改变
        boolean modified = false;
        try {
            // 这里将需要的元素保留，并且向前移位，r 位移动到 w 位， 这里 r >= w
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            // 源码的注释是：保持与AbstractCollection的行为兼容性，即使c.contains()抛出

            // 我的看法是会出现多个线程操作的情况，导致 r != size
            // 这里有一个问题，结合上面来看
            // 如果在进行移位的时候，ArrayList remove 了元素，那么上面的循环里面就会报 ArrayIndexOutOfBoundsException 错误，所以这里的 r == size 的
            // 如果在进行移位的时候，ArrayList add 元素，此时elementData 和 size 都是在同步变化的
            // 所以要想进入这个判断里面运行，就是在上面循环移位结束之后，在进入 if 判断之前，ArrayList 发生了 add 操作
            // r < size
            if (r != size) {
                // 从 r 开始，就是原来的 size 位开始往前移动 size - r 个 add 的元素，目标数组从上面保留元素的最大位 w 开始写入，移动 size - r 个元素
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                // w 则增加 size - r 个 add 的元素
                w += size - r;
            }
            // w == size 表示没有需要去除的元素
            if (w != size) {
                for (int i = w; i < size; i++)
                    // 让 JVM GC 掉这个坑，释放内存
                    elementData[i] = null;
                // 改变了 size - w 次
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }

    // fastRemove(int) 和 remove(int) 功能是差不多的，只是 fastRemove(int) 没有返回类型
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        // 同样的，置为 null 可以释放内存空间，GC可以回收这个占着茅坑的对象
        elementData[--size] = null;
    }
```

#### clear

```java
// clear

    public void clear() {
        modCount++;

        // 这里也是，置为 null 可以释放内存空间，GC可以回收这个占着茅坑的对象
        for (int i = 0; i < size; i++)
            elementData[i] = null;
        // 并把size置为 0 ，此时 isEmpty 就是 true
        size = 0;
    }
```

- `以上是 ArrayList 对数据进行增删改的过程`
- `modCount 主要是用于判断当前数组被数据操作时，没有其他的线程同时操作，具体到下面来讲`

### Iterator

- `讨论 ArrayList 中的 Iterator 和 modCount 用途`

```java
    public Iterator<E> iterator() {
        return new Itr();
    }

    private class Itr implements Iterator<E> {
        // 游标，当前位置
        int cursor;
        // 最后一个返回元素的索引;-1表示没有
        int lastRet = -1;
        // 预计 modCount，用来标记操作
        int expectedModCount = modCount;

        // 这个应该不用说了吧，大家都知道，嘿嘿
        public boolean hasNext() {
            return cursor != size;
        }

        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            // 游标移动到下一个位置
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        // 如果 modCount != expectedModCount，在迭代的时候其他的线程也修改了 elementData
        // 可能会导致数组越界问题
        // 如果发成了排序，也会造成并发修改异常
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }

```

- `ArrayList 的迭代器实现还是比较简单的，主要就是数组的遍历，使用 modCount 控制在迭代期间多线程操作`
- `最后看看Java 8 新增的 ArrayList forEach() 和 ArrayList 迭代器中的 forEachRemaining() 的区别`

#### forEach and forEachRemaining

```java
// Arraylist.forEach()
    @Override
    public void forEach(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        final int expectedModCount = modCount;
        @SuppressWarnings("unchecked")
        final E[] elementData = (E[]) this.elementData;
        final int size = this.size;
        for (int i=0; modCount == expectedModCount && i < size; i++) {
            action.accept(elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

// ArrayList.Itr.forEachRemaining()
        @Override
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // 在迭代结束时更新一次，以减少堆写流量
            cursor = i;
            lastRet = i - 1;
            checkForComodification();
        }
```

- `forEach() 是 Iterable 引入的 default 方法,forEachRemaining() 是 Iterator 引入的 default 方法`
- `forEach() 是从 i = 0 开始进行 for 循环遍历的`
- `forEachRemaining() 可以是使用 while 循环遍历，结合 iterator 迭代中的 cursor 游标，对从游标位置开始的元素进行遍历操作`

## 总结

``` tex
以上是我对 ArrayList 常用的一些方法和内部操作进行的分析

简单的分析了 ArrayList 扩容和删除操作，还要一些操作对 elementData 对象的变更

ArrayList 基于数组的数据结构形式，可快速读取，但是在删除和写入上需要复杂的判断，会对速度一定影响，看完分析可以看出，它并不是一个线程安全的集合

ArrayList set() 和 add() 、addAll() 是没有对 null 过滤的，可以存储 null

在以上贴出的代码中，equals() 是 Object 提供的。ArrayList 没有重写 equals() 和 hashCode()，使用的是 java.util.AbstractList 中的
```
