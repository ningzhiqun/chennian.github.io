- [1. Java 集合框架总结](#title_1)
  - [1.1 集合概述](#title_1_1)
    - [1.1.1 java 集合概览](#title_1_1_1)
    - [1.1.2 List、Set、Map 数据结构总结](#title_1_1_2)
    - [1.1.3 List、Set、Map 三者的区别 ](#title_1_1_3)
    - [1.1.4 为什么要使用集合？](#title_1_1_4)
    - [1.1.5 怎么选择合适的集合？ ](#title_1_1_5)
    - [1.1.6 comparable 和 Comparator 的区别 ](#title_1_1_6)
  - [1.2 collection 子接口 List](#title_1_2)
    - [1.2.1 Arraylist 与 Vector 的区别 ](#title_1_2_1)
    - [1.2.2 Arraylist 与 Linkedlist 的区别 ](#title_1_2_2)
  - [1.3 collection 子接口 Set](#title_1_3)
    - [1.3.1 无序性和不可重复性的含义是什么？ ](#title_1_3_1)
    - [1.3.2 比较 HashSet、LinkedHashSet、TreeSet 三者的异同 ](#title_1_3_2)
  - [1.4 Map 接口](#title_1_4)
    - [1.4.1 HashMap 和 HashTable 的区别](#title_1_4_1)
    - [1.4.2 HashMap 和 HashSet 的区别](#title_1_4_2)
    - [1.4.3 HashMap 和 TreeMap 的区别](#title_1_4_5)
  - [1.5 Collections 工具类](#title_1_5)
    - [1.5.1 排序操作](#title_1_5_1)
    - [1.5.2 查找、替换操作](#title_1_5_2)



# <p id="title_1">1. Java 集合框架总结</p>

### <p id="title_1_1">1.1 集合概述</p>

#### <p id="title_1_1_1">1.1.1 java 集合概览</p>

通过下图可以清晰的看出来，在 Java 中除了以 `Map` 结尾的类之外， 其他类都实现了 `Collection` 接口。

并且，以 `Map` 结尾的类都实现了 `Map` 接口。

![java-collection-hierarchy](/images/java-collection-hierarchy.png)



#### <p id="title_1_1_2">1.1.2 List、Set、Map 数据结构总结</p>

###### 1.1.2.1 List

- `Arraylist`： `Object[]`数组
- `Vector`：`Object[]`数组
- `LinkedList`： 双向链表(JDK1.6 之前为循环链表，JDK1.7 取消了循环)

###### 1.1.2.2 Set

- `HashSet`（无序，唯一）: 基于 `HashMap` 实现的，底层采用 `HashMap` 来保存元素
- `LinkedHashSet`：`LinkedHashSet` 是 `HashSet` 的子类，并且其内部是通过 `LinkedHashMap` 来实现的。有点类似于我们之前说的 `LinkedHashMap` 其内部是基于 `HashMap` 实现一样，不过还是有一点点区别的
- `TreeSet`（有序，唯一）： 红黑树(自平衡的排序二叉树)

###### 1.1.2.3 Map

- `HashMap`： JDK1.8 之前 `HashMap` 由数组+链表组成的，数组是 `HashMap` 的主体，链表则是主要为了解决哈希冲突而存在的（“拉链法”解决冲突）。JDK1.8 以后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间
- `LinkedHashMap`： `LinkedHashMap` 继承自 `HashMap`，所以它的底层仍然是基于拉链式散列结构即由数组和链表或红黑树组成。另外，`LinkedHashMap` 在上面结构的基础上，增加了一条双向链表，使得上面的结构可以保持键值对的插入顺序。同时通过对链表进行相应的操作，实现了访问顺序相关逻辑。
- `Hashtable`： 数组+链表组成的，数组是 `Hashtable` 的主体，链表则是主要为了解决哈希冲突而存在的
- `TreeMap`： 红黑树（自平衡的排序二叉树）



#### <p id="title_1_1_3">1.1.3 List、Set、Map 三者的区别</p>

- `List`(对付顺序的好帮手)： 存储的元素是有序的、可重复的。
- `Set`(注重独一无二的性质): 存储的元素是无序的、不可重复的。
- `Map`(用 Key 来搜索的专家): 使用键值对（key-value）存储，类似于数学上的函数 y=f(x)，“x”代表 key，"y"代表 value，Key 是无序的、不可重复的，value 是无序的、可重复的，每个键最多映射到一个值。



#### <p id="title_1_1_4">1.1.4 为什么要使用集合？</p>

我们知道，在java中有数组的概念，数组可以用来存放一组数据。但是，数组是固定长度的，这样在使用的时候就会有很多的不方便，比如说资源的浪费。这个时候，我们就希望有一种可以动态改变大小的数组，那就是集合的作用了。Java 所有的集合类都位于 java.util 包下，提供了一个表示和操作对象集合的统一构架，包含大量集合接口，以及这些接口的实现类和操作它们的算法。



#### <p id="title_1_1_5">1.1.5 怎么选择合适的集合？</p>

Java 为您提供了多种收集实现供您选择。 通常，您将始终为您的编程任务寻找性能最佳的集合，在大多数情况下为ArrayList，HashSet或HashMap。但是请注意，如果您需要某些特殊功能（例如排序或排序），则可能需要进行特殊的实现。 该 Java 集合教程不包括WeakHashMap等很少使用的类，因为它们是为非常特定或特殊任务设计的，因此在 99% 的情况下都不应该选择它们。

###### 1.1.5.1 选择正确的 Java Map 接口

HashMap –如果迭代时项目的顺序对您不重要，请使用此实现。与TreeMap和LinkedHashMap相比，HashMap具有更好的性能。

TreeMap – 已排序和排序，但比HashMap慢。TreeMap根据其比较器具有键的升序

LinkedHashMap – 在插入过程中按键对项目排序

###### 1.1.5.2 选择正确的 Java List 接口

ArrayList –插入期间对项目进行排序。与对LinkedLists的搜索操作相比，对ArrayLists的搜索操作更快

LinkedList – 已快速添加到列表的开头，并通过迭代从内部快速删除

###### 1.1.5.3 选择正确的 Java Set 接口

HashSet – 如果迭代时项目的顺序对您不重要，请使用此实现。与TreeSet和LinkedHashSet相比，HashSet具有更好的性能

LinkedHashSet – 在插入过程中排序元素

TreeSet – 根据其比较器，按键的升序排序



#### <p id="title_1_1_6">1.1.6 comparable 和 Comparator 的区别</p>

1、如果实现类没有实现Comparable接口，又想对两个类进行比较（或者实现类实现了Comparable接口，但是对compareTo方法内的比较算法不满意），那么可以实现Comparator接口，自定义一个比较器，写比较算法

2、实现Comparable接口的方式比实现Comparator接口的耦合性要强一些，如果要修改比较算法，要修改Comparable接口的实现类，而实现Comparator的类是在外部进行比较的，不需要对实现类有任何修改。从这个角度说，其实有些不太好，尤其在我们将实现类的.class文件打成一个.jar文件提供给开发者使用的时候。实际上实现Comparator接口的方式后面会写到就是一种典型的**策略模式**。



### <p id="title_1_2">1.2 collection 子接口 List</p>

#### <p id="title_1_2_1">1.2.1 Arraylist 与 Vector 的区别</p>

- `ArrayList` 是 `List` 的主要实现类，底层使用 `Object[ ]`存储，适用于频繁的查找工作，线程不安全 ；
- `Vector` 是 `List` 的古老实现类，底层使用`Object[ ]` 存储，线程安全的。

#### <p id="title_1_2_2">1.2.2 Arraylist 与 Linkedlist 的区别</p>

1. **是否保证线程安全：** `ArrayList` 和 `LinkedList` 都是不同步的，也就是不保证线程安全；
2. **底层数据结构：** `Arraylist` 底层使用的是 **`Object` 数组**；`LinkedList` 底层使用的是 **双向链表** 数据结构（JDK1.6 之前为循环链表，JDK1.7 取消了循环。注意双向链表和双向循环链表的区别，下面有介绍到！）
3. 插入和删除是否受元素位置的影响：
   - `ArrayList` 采用数组存储，所以插入和删除元素的时间复杂度受元素位置的影响。 比如：执行`add(E e)`方法的时候， `ArrayList` 会默认在将指定的元素追加到此列表的末尾，这种情况时间复杂度就是 O(1)。但是如果要在指定位置 i 插入和删除元素的话（`add(int index, E element)`）时间复杂度就为 O(n-i)。因为在进行上述操作的时候集合中第 i 和第 i 个元素之后的(n-i)个元素都要执行向后位/向前移一位的操作。
   - `LinkedList` 采用链表存储，所以，如果是在头尾插入或者删除元素不受元素位置的影响（`add(E e)`、`addFirst(E e)`、`addLast(E e)`、`removeFirst()` 、 `removeLast()`），近似 O(1)，如果是要在指定位置 `i` 插入和删除元素的话（`add(int index, E element)`，`remove(Object o)`） 时间复杂度近似为 O(n) ，因为需要先移动到指定位置再插入。
4. **是否支持快速随机访问：** `LinkedList` 不支持高效的随机元素访问，而 `ArrayList` 支持。快速随机访问就是通过元素的序号快速获取元素对象(对应于`get(int index)`方法)。
5. **内存空间占用：** ArrayList 的空 间浪费主要体现在在 list 列表的结尾会预留一定的容量空间，而 LinkedList 的空间花费则体现在它的每一个元素都需要消耗比 ArrayList 更多的空间（因为要存放直接后继和直接前驱以及数据）。

### <p id="title_1_3">1.3 collection 子接口 Set</p>

#### <p id="title_1_3_1">1.3.1 无序性和不可重复性的含义是什么？</p>

1、什么是无序性？无序性不等于随机性 ，无序性是指存储的数据在底层数组中并非按照数组索引的顺序添加 ，而是根据数据的哈希值决定的。

2、什么是不可重复性？不可重复性是指添加的元素按照 equals()判断时 ，返回 false，需要同时重写 equals()方法和 HashCode()方法。

#### <p id="title_1_3_2">1.3.2 比较 HashSet、LinkedHashSet、TreeSet 三者的异同</p>

`HashSet` 是 `Set` 接口的主要实现类 ，`HashSet` 的底层是 `HashMap`，线程不安全的，可以存储 null 值；

`LinkedHashSet` 是 `HashSet` 的子类，能够按照添加的顺序遍历；

`TreeSet` 底层使用红黑树，能够按照添加元素的顺序进行遍历，排序的方式有自然排序和定制排序。

### <p id="title_1_4">1.4 Map 接口</p>

#### <p id="title_1_4_1">1.4.1 HashMap 和 HashTable 的区别</p>

1. **线程是否安全：** `HashMap` 是非线程安全的，`HashTable` 是线程安全的,因为 `HashTable` 内部的方法基本都经过`synchronized` 修饰。（如果你要保证线程安全的话就使用 `ConcurrentHashMap` 吧！）；
2. **效率：** 因为线程安全的问题，`HashMap` 要比 `HashTable` 效率高一点。另外，`HashTable` 基本被淘汰，不要在代码中使用它；
3. **对 Null key 和 Null value 的支持：** `HashMap` 可以存储 null 的 key 和 value，但 null 作为键只能有一个，null 作为值可以有多个；HashTable 不允许有 null 键和 null 值，否则会抛出 `NullPointerException`。
4. **初始容量大小和每次扩充容量大小的不同 ：** ① 创建时如果不指定容量初始值，`Hashtable` 默认的初始大小为 11，之后每次扩充，容量变为原来的 2n+1。`HashMap` 默认的初始化大小为 16。之后每次扩充，容量变为原来的 2 倍。② 创建时如果给定了容量初始值，那么 Hashtable 会直接使用你给定的大小，而 `HashMap` 会将其扩充为 2 的幂次方大小（`HashMap` 中的`tableSizeFor()`方法保证，下面给出了源代码）。也就是说 `HashMap` 总是使用 2 的幂作为哈希表的大小,后面会介绍到为什么是 2 的幂次方。
5. **底层数据结构：** JDK1.8 以后的 `HashMap` 在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间。Hashtable 没有这样的机制。

#### <p id="title_1_4_2">1.4.2 HashMap 和 HashSet 的区别</p>

如果你看过 `HashSet` 源码的话就应该知道：`HashSet` 底层就是基于 `HashMap` 实现的。（`HashSet` 的源码非常非常少，因为除了 `clone()`、`writeObject()`、`readObject()`是 `HashSet` 自己不得不实现之外，其他方法都是直接调用 `HashMap` 中的方法。

| `HashMap`                              |                                                    `HashSet` |
| -------------------------------------- | -----------------------------------------------------------: |
| 实现了 `Map` 接口                      |                                              实现 `Set` 接口 |
| 存储键值对                             |                                                   仅存储对象 |
| 调用 `put()`向 map 中添加元素          |                          调用 `add()`方法向 `Set` 中添加元素 |
| `HashMap` 使用键（Key）计算 `hashcode` | `HashSet` 使用成员对象来计算 `hashcode` 值，对于两个对象来说 `hashcode` 可能相同，所以`equals()`方法用来判断对象的相等性 |

#### <p id="title_1_4_3">1.4.3 HashMap 和 TreeMap 的区别</p>

`TreeMap` 和`HashMap` 都继承自`AbstractMap` ，但是需要注意的是`TreeMap`它还实现了`NavigableMap`接口和`SortedMap` 接口。

**相比于`HashMap`来说 `TreeMap` 主要多了对集合中的元素根据键排序的能力以及对集合内元素的搜索的能力。**

### <p id="title_1_5">1.5 Collections 工具类</p>

#### <p id="title_1_5_1">1.5.1 排序操作</p>

```
void reverse(List list)//反转
void shuffle(List list)//随机排序
void sort(List list)//按自然排序的升序排序
void sort(List list, Comparator c)//定制排序，由Comparator控制排序逻辑
void swap(List list, int i , int j)//交换两个索引位置的元素
void rotate(List list, int distance)//旋转。当distance为正数时，将list后distance个元素整体移到前面。当distance为负数时，将 list的前distance个元素整体移到后面
```

#### <p id="title_1_5_2">1.5.2 查找、替换操作</p>

```
int binarySearch(List list, Object key)//对List进行二分查找，返回索引，注意List必须是有序的
int max(Collection coll)//根据元素的自然顺序，返回最大的元素。 类比int min(Collection coll)
int max(Collection coll, Comparator c)//根据定制排序，返回最大元素，排序规则由Comparatator类控制。类比int min(Collection coll, Comparator c)
void fill(List list, Object obj)//用指定的元素代替指定list中的所有元素
int frequency(Collection c, Object o)//统计元素出现次数
int indexOfSubList(List list, List target)//统计target在list中第一次出现的索引，找不到则返回-1，类比int lastIndexOfSubList(List source, list target)
boolean replaceAll(List list, Object oldVal, Object newVal)//用新元素替换旧元素
```
