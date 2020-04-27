# Java 8 LinkedList 源码简单分析

## LinkedList

```txt
LinkedList 是集合 List 的实现类，也是双端队列 Deque 的实现。

所以 LinkedList 具有集合和双端队列的双重特性。

LinkedList 是通过链表实现的，相比 ArrayList 的数组实现，它在插入和删除更加高效。
```

## 问题
```txt
关于实现 LinkedList 的数据结构是否为循环的双向链表，上网搜了有很多文章都说是循环的？在 Java 8 中我看了源码是不循环的。
```
```java
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
```
```txt
删除第一个节点后，next.prev = null; 并未指向 last。应该是在 Java 8 中对 LinkedList 进行改造了。 
```

## LinkedList源码分析

> 基本数据结构
```java
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
`
这是一个标准的双向链表的结构。
`

> 源码
```java
    /**
     * 内部方法 
     * 在链表头部添加节点
     */
    private void linkFirst(E e) {
        final Node<E> f = first;
        // 新建一个 node 节点,将 prev -> null， next -> f
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        // first == null 认为当前链表是没有节点的，
        // 将 last 同时指向 newNode
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }

    /**
     * 内部方法
     * 在链表尾部添加节点
     */
    void linkLast(E e) {
        final Node<E> l = last;
        // 新建一个 node 节点，将 prev -> l, next -> null
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        // last == null 认为当前链表是没有节点的，
        // first -> newNode 这样 first 头也有了
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }

    /**
     * 内部方法
     * 在非空节点之前插入元素
     */
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        // newNode 的 newNode.prev -> succ.prev 
        // newNode.next -> succ
        final Node<E> newNode = new Node<>(pred, e, succ);
        // newNode 取代了 succ 节点的位置， 所以 succ.prev -> newNode
        succ.prev = newNode;
        // pred == null 代表 succ 已经是 first 节点
        // 在其前面插入，也就是在 first 位置插入节点
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }

    /**
     * 内部方法
     * 删除非空的第一个节点
     */
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        // 下面两部为了释放对象的引用，方便 GC
        f.item = null;
        f.next = null; // help GC
        first = next;
        // 被删除的节点是链表唯一一个节点
        // 将 last 置空
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }

    /**
     * 内部方法     
     * 删除非空最后一个节点
     */
    private E unlinkLast(Node<E> l) {
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }

    /**
     * 内部方法    
     * 删除一个非空节点
     */
    E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
```