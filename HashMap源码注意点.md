### HashMap源码注意点
> HashMap内部是使用一个默认容量为16的数组来存储数据的，而数组中每一个元素却又是一个链表的头结点，所以，更准确的来说，HashMap内部存储结构是使用哈希表的拉链结构（数组+链表）
#### HashMap的一些特性：
+ 其中的每一个节点，我们称之为 Entry类型，可以理解为包含内容为 Key Value Hash值 和 next指向的下一个Entry
+  基于Map接口实现，允许null键/值，并不是同步的，也就是说不是线程安全的；而且不保证有序，也不保证顺序不随时间变化。
+  在不发生散列冲突的情况下，HashMap的查询速度是常量级别的 O(1)
+  当发生散列冲突的情况下，将元素存放在对应冲突的哈希表的头节点的链表下面；
+  那么HashMap为了优化当散列冲突过多的情况，某一哈希表下的链表过于冗长，在 java8 之后提出了，当链表的节点大于等于 8 的时候，会将这个链表转化为一个红黑树（二叉排序树）来进行数据存储；以便于减少散列冲突带来的查询时间的增加；
+  由于红黑树的逻辑结构复杂，从源码中，我得知当节点个数小于等于6的时候，红黑树就又会转换为一条链表；

源码观察：
```java
//先来看一下关键字段

 static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;//默认的初始容量，aka 16
 static final float DEFAULT_LOAD_FACTOR = 0.75f;//默认的填充因子（可以修改）
 static final int TREEIFY_THRESHOLD = 8;//桶上链表转数的最大值
 static final int UNTREEIFY_THRESHOLD = 6;//红黑树转链表的元素的最小值
 transient Node<K,V>[] table;//哈希数组，用来进行下标索引查找元素
 transient int size//此HashMap中所有元素的个数
```
+ 解决散列冲突问题后，对于大量的数据利用红黑树+定长的结构去存储显然不太现实；于是HashMap就会进行扩容为当前的容量的两倍（16—>32）；
+ 扩容的标准是由所设定的负载因子所决定的，源码上标识为 0.75f ；当所有Entry的数目大于 （当前容量 x 负载因子），就会进行扩容处理；

#### Get函数和Put函数的实现思路：
+ put函数大致的思路为：
1.	对key的hashCode()做hash，然后再计算index; 
2.	如果没碰撞直接放到bucket里； 	
3.	如果碰撞了，以链表的形式存在buckets后；
4.	如果碰撞导致链表过长(大于等于TREEIFY_THRESHOLD)，就把链表转换成 红黑树； 
5.	如果节点已经存在就替换old	value(保证key的唯一性) 
6. 如果bucket满了(超过load	factor*current	capacity)，就要resize

+ Get函数的大致思路
1.	bucket里的第一个节点，直接命中； 
2.		如果有冲突，则通过key.equals(k)去查找对应的entry
若为树，则在树中通过key.equals(k)查找，O(logn)；
若为链表，则在链表中通过key.equals(k)查找，O(n)

#### Hash（）函数的源码解释：
```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
你知道hash的实现吗？为什么要这样实现
+ 在Java	1.8的实现中，是通过hashCode()的高16位异或低16位实现的：	(h	= k.hashCode())	^	(h	>>>	16)	==（高16bit不变，低16bit和高16bit做了一个异 或）==
+ hash算法的处理机制是将一个键的hashcode的值与他的hashcode右移16位之后的值进行异或运算，高位的信息得以保存，地位的信息中掺杂了高位的信息，这样可以使得出来的数的随机性更大，更不容易散列冲突。
+ 主要是从速度、功效、质量来考虑的，这么做 可以在bucket的n比较小的时候，也能保证考虑到高低bit都参与到hash的计算中， 同时不会有太大的开销

##### 碰撞后，利用链表和树存储结构的查找时间复杂度比较：
1.	首先根据hashCode()做hash，然后确定bucket的index；==（计算下标：(n	-	1)	&	hash）==
2.	如果bucket的节点的key不是我们需要的，则通过keys.equals()在链中找。

##### 链地址法
在Java	8之前的实现中是用链表解决冲突的，在产生碰撞的情况下，进行get时，两 步的时间复杂度是O(1)+O(n)。因此，当碰撞很厉害的时候n很大，O(n)的速度显然 是影响速度的。
因此在Java	8中，利用红黑树替换链表，这样复杂度就变成了O(1)+O(logn)了，这 样在n很大的时候，能够比较理想的解决这个问题

#### HashTable
HashTable是基于陈旧的 Dictionary；不允许键值和Key 为 null；

HashMap作为HashTable的轻量级实现，但是HashTable是线程安全的，在源码中，HashTable的每一个方法都被Synchronize 关键字修饰；

#### 常见的HashMap的问题：
+ HashMap的工作原理： 
	+ 通过hash的方法，通过put和get存储和获取对象。
	+ 存储对象时，我们将K/V传给put 方法时，它调用hashCode计算hash从而得到bucket位置，进一步存储，会根据当前bucket的占用情况自动调整容量(超过Load	Facotr则resize为原来的2 倍)。
	+ 获取对象时，我们将K传给get，它调用hashCode计算hash从而得到bucket位 置，并进一步调用equals()方法确定键值对。如果发生碰撞的时候，Hashmap通过 链表将产生碰撞冲突的元素组织起来。
	+ 在Java	8中，如果一个bucket中碰撞冲突的 元素超过某个限制(默认是8)，则使用红黑树来替换链表，从而提高速度。 

+ 你知道get和put的原理吗？equals()和hashCode()的都有什么作用？ 
	+  通过对key的hashCode()进行hashing，并计算下标(	n-1	&	hash)，从而获得 buckets的位置。
	+  如果产生碰撞，则利用key.equals()方法去链表或树中去查找对应 的节点

+ 为什么要高低位互相异或再求hash？
	+ 在n	-	1为 15(0x1111)时，其实散列真正生效的只是低4bit的有效位，当然容易碰撞了。
	+ 因此，设计者想了一个顾全大局的方法(综合考虑了速度、作用、质量)，就是把高 16bit和低16bit异或了一下，增加了复杂度，保证再n比较小的时候的碰撞几率；

+ 为什么链表变成红黑树后，它会效率比较高？
	+ 红黑树是一棵自平衡的二叉树，也叫二叉搜索树；
	+ 特性： 左右子树的高度差绝对值不超过1
	+ 平衡的意思就是说它是是对称的，保证了中序遍历的出来的数据是由小到大递增，根节点相当于数据的中点；
	+ 类似于二分查找的思想，查询速度加快，可以达到 log( n ) ；

+ TreeMap ，LinkedHashMap与 HashMap 的区别？
	+ HashMap 不保证数据有序，LinkedHashMap 可以保证输出的时候，数据为插入的顺序；TreeMap 可以在输出的时候保持 key 从小到大顺序输出。
	+ TreeMap 的 桶结构是 红黑树，利用二叉搜索树的特性，中序遍历出来就保证了 key 是有序的；
	+ HashMap 的 桶结构通常为 链表，java8 之后，当桶里的节点数多于8 就转换为 红黑树；小于 6 就转化为 链表；
	+ LinkedHashMap 是 hash 表加双向链表，利用双向链表保持 节点的插入顺序
 

#####  HashMap和ConcurrentHashMap的区别联系？
[救急->ConcurrentHashMap](http://www.jasongj.com/java/concurrenthashmap/)
>ConcurrentHashMap是并发包（Concurrent）的重要成员；所以它是一种线程安全，并且支持高效并发的 HashMap；
> 与之相似的 HashTable 虽然也是线程同步，但是其使用的全局锁，不得不使得我们关注它的性能；

> 理想状态下：
> 支持16个线程执行并发写操作及任意数量的线程的读操作；

区别：
concurrent与 hashTable 一样 ，不允许key 和 value 的值为null；

线程安全



##### HashMap线程不安全主要体现在 resize （扩容的时候）容易发生死循环；
形成的原因：
+ 多并发情况下多个Map扩容时，多个线程对同一个捅进行操作，当某个线程的时间片用完等产生被挂起操作，第二个线程对整个Map扩容完毕，可这时，第一个线程被唤起执行，导致按照顺序执行的时候容易把该捅弄成环；

##### concurrentHashMap Java7和Java8的结构改变；
Java7：基于分段锁，借助多个 segment 数组，里面存放着多个小的 hashMap ；
在读写key 的时候 ，哈希值的高N位对Segment个数取模从而得到该Key应该属于哪个Segment
> Segment 类继承于 ReentrantLock 类，从而使得 Segment 对象能充当锁的角色

Java 8 : 摒弃了分段锁的方案，使用大数组提高了哈希碰撞下的寻址性能；（segment 的寻址麻烦）

### 关于内存优化角度思考的HashMap替换问题：
HashMap为什么需要根据需求进行内存优化的原因：
> Android作为对内存非常敏感的移动平台；
> 当我们面对的数百万条数据的时候，HashMap的扩容特性，导致不断的扩充空间的同时又要进行一次次的Hash操作，这个时候对于本来有限的内存空间造成了较大的消耗和浪费；

##### 所以在一些情况下我们可以使用 SparseArray 和 ArrayMap 来替换HashMap

##### SparseArray
在数据量在千级以内，Key必须为int类型的情况下；性能比HashMap更好，原因是避免了对Key的自动装箱（int -> Integer），内部通过两个数组来进行数据存储，分别存储Key和Value，同时对稀疏数组的数据采用了压缩的方式；
在进行存储和读取数据的时候，都是使用二分查找法，自然比HashMap遍历Entry数组更快；
由于在添加数据的时候，会按照key值的大小，按照从小到大的顺序排序，所以SparseArray存储的元素都是按元素的key值从小到大排列好的

##### ArrayMap
它是一个<key ， vakue> 映射的数据结构；
应用场景和SparseArray相似，
+ 千级别以内，数据结构类型为Map；

也是用两个数组分别存储 key 的hash 值和 value值。也会对key使用二分法进行从小到大排序，在添加、删除、查找数据的时候都是先使用二分查找法得到相应的index，然后通过index来进行添加、查找、删除等操作

#### 遍历 HashMap 的方法：
> HashMap 在遍历的时候，是按照哈希表的每个索引的链表从上往下遍历；
> 
> HashMap 的遍历方法由 4 种 ；
> 
> 其中 keySet() 方法 和 entrySet 比较常用，entrySet 由于只用遍历 map 一次 所以比较高效；
##### 使用 KeySet() 方法遍历：
通过取出 Map 中的 key 并把其存储成一个 set 集合，遍历 set 集合 ，通过 key 去获取 Map 里面的 Value 值；
```java
for (Integer integer : tmap.keySet()) {
            System.out.println(tmap.getClass().getName() + "  "
                    + integer + " " + tmap.get(integer));
        }
```

##### 使用 entrySet() 遍历：
该方法会返回一个以 Entry 为元素的 Set 集合 ，然后对该 Set 集合进行遍历：
```java
for (Map.Entry<Integer, String> entry : tmap.entrySet()) {
            System.out.println(tmap.getClass().getName() + 
                    entry.getKey() + "  " + entry.getValue());
        }
```

##### 使用 KeySet() 返回的集合的 iterator（迭代器） 遍历：
```java
Iterator<Integer> it = tmap.keySet().iterator();
        while (it.hasNext()) {
            Integer key = it.next();
            System.out.println(key + " : " + tmap.get(key));
        }
```
##### 使用map的values()方法遍历集合的values;
map.values()返回的是由map的值组成的Collection，这个方法只能遍历map的所有value，不能得到map的key;
```java
for (String value : map.values()) {
    System.out.println(value);
}
```



	



