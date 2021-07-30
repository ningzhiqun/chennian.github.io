

- [1. 简介](#title_1)
- [2. 内部结构——双向链表](#title_2)
- [3. LinkedList 概述](#title_3)
  - [3.1 LinkedList 构造方法](#title_3_1)
  - [3.2 LinkedList 增删改查](#title_3_2)
  - [3.3 LinkedList 遍历](#title_3_3)



### <p id="title_1">1. 简介</p>

LinkedList是一个实现了List接口和Deque接口的双端链表。 LinkedList底层的链表结构使它支持高效的插入和删除操作，另外它实现了Deque接口，使得LinkedList类也具有队列的特性; LinkedList不是线程安全的，如果想使LinkedList变成线程安全的，可以调用静态类Collections类中的synchronizedList方法：

```
List list=Collections.synchronizedList(new LinkedList(...));
```

### <p id="">2. 内部结构——双向链表</p>

![img](https://gitee.com/SnailClimb/JavaGuide/raw/master/docs/java/collection/images/linkedlist/LinkedList%E5%86%85%E9%83%A8%E7%BB%93%E6%9E%84.png)

看完了图之后，我们再看LinkedList类中的一个**内部私有类Node**就很好理解了：

```
private static class Node<E> {
        E item;//节点值
        Node<E> next;//后继节点
        Node<E> prev;//前驱节点

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

这个类就代表双端链表的节点Node。这个类有三个属性，分别是前驱节点，本节点的值，后继结点。

### <p id="title_3">3. LinkedList 概述</p>



> 图中蓝色实线箭头是指继承关系 ，绿色虚线箭头是指接口实现关系。

1、`LinkedList` 继承自 `AbstrackSequentialList` 并实现了 `List` 接口以及 `Deque` 双向队列接口，因此 LinkedList 不但拥有 List 相关的操作方法，也有队列的相关操作方法。

2、`LinkedList` 和 `ArrayList` 一样实现了序列化接口 `Serializable` 和 `Cloneable` 接口使其拥有了序列化和克隆的特性。

**`LinkedList` 一些主要特性：**

1. **`LinkedList` 集合底层实现的数据结构为双向链表**
2. **`LinkedList` 集合中元素允许为 null**
3. **`LinkedList` 允许存入重复的数据**
4. **`LinkedList` 中元素存放顺序为存入顺序。**
5. **`LinkedList` 是非线程安全的，如果想保证线程安全的前提下操作 `LinkedList`，可以使用 `List list = Collections.synchronizedList(new LinkedList(...));` 来生成一个线程安全的 `LinkedList`**

#### <p id="title_3_1">3.1 LinkedList 构造方法</p>

LinkedList 有两个构造函数：

```java
/**
 * 空构造方法：
 */
 public LinkedList() {
 }

/**
 * 用已有的集合创建链表的构造方法：
 */
public LinkedList(Collection<? extends E> c) {
   this();
   addAll(c);
}
```

#### <p id="title_3_2">3.2 LinkedList 增删改查</p>

##### 新增

**add(E e)** 方法：将元素添加到链表尾部

```
public boolean add(E e) {
        linkLast(e);//这里就只调用了这一个方法
        return true;
    }
   /**
     * 链接使e作为最后一个元素。
     */
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;//新建节点
        if (l == null)
            first = newNode;
        else
            l.next = newNode;//指向后继元素也就是指向下一个元素
        size++;
        modCount++;
    }
```

**add(int index,E e)**：在指定位置添加元素

```
public void add(int index, E element) {
        checkPositionIndex(index); //检查索引是否处于[0-size]之间

        if (index == size)//添加在链表尾部
            linkLast(element);
        else//添加在链表中间
            linkBefore(element, node(index));
    }
```

linkBefore方法需要给定两个参数，一个插入节点的值，一个指定的node，所以我们又调用了Node(index)去找到index对应的node

##### 查找

**get(int index)：** 根据指定索引返回数据

```
public E get(int index) {
        //检查index范围是否在size之内
        checkElementIndex(index);
        //调用Node(index)去找到index对应的node然后返回它的值
        return node(index).item;
    }
```

**获取头节点（index=0）数据方法:**

```
public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }
public E element() {
        return getFirst();
    }
public E peek() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }

public E peekFirst() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
     }
```

**区别：** getFirst(),element(),peek(),peekFirst() 这四个获取头结点方法的区别在于对链表为空时的处理，是抛出异常还是返回null，其中**getFirst()** 和**element()** 方法将会在链表为空时，抛出异常

element()方法的内部就是使用getFirst()实现的。它们会在链表为空时，抛出NoSuchElementException
**获取尾节点（index=-1）数据方法:**

```
 public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }
 public E peekLast() {
        final Node<E> l = last;
        return (l == null) ? null : l.item;
    }
```

**两者区别：** **getLast()** 方法在链表为空时，会抛出**NoSuchElementException**，而**peekLast()** 则不会，只是会返回 **null**。

#####  删除方法

**remove()** ,**removeFirst(),pop():** 删除头节点

```
public E pop() {
        return removeFirst();
    }
public E remove() {
        return removeFirst();
    }
public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }
```

**removeLast(),pollLast():** 删除尾节点

```
public E removeLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return unlinkLast(l);
    }
public E pollLast() {
        final Node<E> l = last;
        return (l == null) ? null : unlinkLast(l);
    }
```

**区别：** removeLast()在链表为空时将抛出NoSuchElementException，而pollLast()方法返回null。

**remove(Object o):** 删除指定元素

```
public boolean remove(Object o) {
        //如果删除对象为null
        if (o == null) {
            //从头开始遍历
            for (Node<E> x = first; x != null; x = x.next) {
                //找到元素
                if (x.item == null) {
                   //从链表中移除找到的元素
                    unlink(x);
                    return true;
                }
            }
        } else {
            //从头开始遍历
            for (Node<E> x = first; x != null; x = x.next) {
                //找到元素
                if (o.equals(x.item)) {
                    //从链表中移除找到的元素
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }
```

当删除指定对象时，只需调用remove(Object o)即可，不过该方法一次只会删除一个匹配的对象，如果删除了匹配对象，返回true，否则false。

unlink(Node x) 方法：

```
E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;//得到后继节点
        final Node<E> prev = x.prev;//得到前驱节点

        //删除前驱指针
        if (prev == null) {
            first = next;//如果删除的节点是头节点,令头节点指向该节点的后继节点
        } else {
            prev.next = next;//将前驱节点的后继节点指向后继节点
            x.prev = null;
        }

        //删除后继指针
        if (next == null) {
            last = prev;//如果删除的节点是尾节点,令尾节点指向该节点的前驱节点
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

**remove(int index)**：删除指定位置的元素

```
public E remove(int index) {
        //检查index范围
        checkElementIndex(index);
        //将节点删除
        return unlink(node(index));
    }
```

#### <p id="title_3_3">3.3 LinkedList 遍历</p>

在 `ArrayList` 分析的时候，我们就知道 `List` 的实现类，有4中遍历方式：for 循环，高级 for 循环，`Iterator` 迭代器方法， `ListIterator` 迭代方法。由于 `ArrayList` 源码分析的时候比较详细看了源码，对于不同数据结构的 `LinkedList` 我们只看下他们的不同之处.

`LinkedList` 没有单独 `Iterator` 实现类，它的 `iterator` 和 `listIterator` 方法均返回 `ListItr`的一个对象。 LinkedList 作为双向链表数据结构，过去上个元素和下个元素很方便。

下边我们来看下 `ListItr` 的源码：

```
private class ListItr implements ListIterator<E> {
   // 上一个遍历的节点
   private Node<E> lastReturned;
   // 下一次遍历返回的节点
   private Node<E> next;
   // cursor 指针下一次遍历返回的节点
   private int nextIndex;
   // 期望的操作数
   private int expectedModCount = modCount;
    
   // 根据参数 index 确定生成的迭代器 cursor 的位置
   ListItr(int index) {
       // assert isPositionIndex(index);
       // 如果 index == size 则 next 为 null 否则寻找 index 位置的节点
       next = (index == size) ? null : node(index);
       nextIndex = index;
   }

   // 判断指针是否还可以移动
   public boolean hasNext() {
       return nextIndex < size;
   }
    
  // 返回下一个带遍历的元素
  public E next() {
       // 检查操作数是否合法
       checkForComodification();
       // 如果 hasNext 返回 false 抛出异常，所以我们在调用 next 前应先调用 hasNext 检查
       if (!hasNext())
           throw new NoSuchElementException();
        // 移动 lastReturned 指针
       lastReturned = next;
        // 移动 next 指针
       next = next.next;
       // 移动 nextIndex cursor
       nextIndex++;
       // 返回移动后 lastReturned
       return lastReturned.item;
   }

  // 当前游标位置是否还有前一个元素
   public boolean hasPrevious() {
       return nextIndex > 0;
   }
  
  // 当前游标位置的前一个元素
   public E previous() {
       checkForComodification();
       if (!hasPrevious())
           throw new NoSuchElementException();
        // 等同于 lastReturned = next；next = (next == null) ? last : next.prev;
        // 发生在 index = size 时
       lastReturned = next = (next == null) ? last : next.prev;
       nextIndex--;
       return lastReturned.item;
   }
    
   public int nextIndex() {
       return nextIndex;
   }

   public int previousIndex() {
       return nextIndex - 1;
   }
    
    // 删除链表当前节点也就是调用 next/previous 返回的这节点，也就 lastReturned
   public void remove() {
       checkForComodification();
       if (lastReturned == null)
           throw new IllegalStateException();

       Node<E> lastNext = lastReturned.next;
       //调用LinkedList 的删除节点的方法
       unlink(lastReturned);
       if (next == lastReturned)
           next = lastNext;
       else
           nextIndex--;
       //上一次所操作的 节点置位空    
       lastReturned = null;
       expectedModCount++;
   }

    // 设置当前遍历的节点的值
   public void set(E e) {
       if (lastReturned == null)
           throw new IllegalStateException();
       checkForComodification();
       lastReturned.item = e;
   }
    // 在 next 节点位置插入及节点
   public void add(E e) {
       checkForComodification();
       lastReturned = null;
       if (next == null)
           linkLast(e);
       else
           linkBefore(e, next);
       nextIndex++;
       expectedModCount++;
   }
    //简单哈操作数是否合法
   final void checkForComodification() {
       if (modCount != expectedModCount)
           throw new ConcurrentModificationException();
   }
}
```

