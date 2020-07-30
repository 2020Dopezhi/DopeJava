[TOC]
# 一、修改HashTable
HashTable是线程安全的HashMap，里面的方法都是用synchronized修饰过的，跟HashMap有以下不同之处：
- 初始容量不同，HashMap是16，HashTable是11
- 扩容倍数不同，HashMap扩容是1.5倍，而HashTable是2倍
- HashMap可以存储值为null的元素，而HashTable不可以
- HashMap的迭代器是基于快速失败(modCount)机制，fail-fast，而HashTable的迭代器不是快速失败的。


# 二、Colletions.synchronizedMap
除HashTable外，利用Colletions.synchronizedMap也可以得到线程安全的map，synchronizedMap是Collections的一个静态内部类，里面的方法都是用synchronized代码块去修饰的，因此也是线程安全的。  

![image](http://note.youdao.com/yws/res/3857/A284DF17E30E48358E0F95CD6CC778C5)  

# 三、1.7的ConcurrentHashMap
在JDK 1.7的ConcurrentHashMap,底层的数据结果是数组+链表，但是引入了分段锁（segement）的概念。   

分段锁是基于ReentranLock实现的，基于AQS。

一个ConcurrentHashMap有多个分段锁，分段锁管理着不同的HashEntry，想要对HashEntry进行操作，必须获取HashEntry所在的segement分段锁。  

![image](http://note.youdao.com/yws/res/3866/1C97AA6138E242898C6837998B39E468) 

## 1.7 put
首先会通过key的hashCode获取key对应的那个segement锁（Hash第一次找到segement锁），然后会在第二次Hash获取到对应HashEntry的下标，再进行Put操作：  
```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
  // 将当前 Segment 中的 table 通过 key 的 hashcode 定位到 HashEntry
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
// 遍历该 HashEntry，如果不为空则判断传入的 key 和当前遍历的 key 是否相等，相等则覆盖旧的 value。
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
         // 不为空则需要新建一个 HashEntry 并加入到 Segment 中，同时会先判断是否需要扩容。
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
       //释放锁
        unlock();
    }
    return oldValue;
}
```

## 1.7 get
1.7中的get逻辑比较简单，**get操作是不用加锁的**，只需要将key通过hash之后找到对应的segement，再通过一次hash定位到具体的位置上。  

**因为value是用volatile修饰的，因此能够保证每次获取到的value值都是最新的。**

## 1.7 size
尝试用不加锁的形式多次计算Concurrent的size，最多三次，比较前后统计出来的size次数，如果前后统计的size相等的话，那么就说明当前没有元素加入，返回size。  

如果统计三次发现前后结果不一致，会获取所有的分段锁进行size统计，最后返回size。  

```java
public int size() {
  final Segment<K,V>[] segments = this.segments;
  int size;
  boolean overflow; // true if size overflows 32 bits
  long sum;         // sum of modCounts
  long last = 0L;   // previous sum
  int retries = -1; // first iteration isn't retry
  try {
    for (;;) {
    //死循环中最多统计三次，三次以上前后结果不相同
    //获取所有的分段锁统计size
      if (retries++ == RETRIES_BEFORE_LOCK) {
        for (int j = 0; j < segments.length; ++j)
          ensureSegment(j).lock(); // force creation
      }
      sum = 0L;
      size = 0;
      overflow = false;
      for (int j = 0; j < segments.length; ++j) {
        Segment<K,V> seg = segmentAt(segments, j);
        if (seg != null) {
          sum += seg.modCount;
          int c = seg.count;
          if (c < 0 || (size += c) < 0)
            overflow = true;
        }
      }
      //如果前后次数相等，说明没有元素加入，返回结果
      if (sum == last)
        break;
      last = sum;
    }
  } finally {
    if (retries > RETRIES_BEFORE_LOCK) {
      for (int j = 0; j < segments.length; ++j)
        segmentAt(segments, j).unlock();
    }
  }
  return overflow ? Integer.MAX_VALUE : size;
}
```


# 四、1.8的ConcurrentHashMap
在JDK1.8中，完全摒弃了分段锁segement的概念，使用了CAS+Synchronized来保证其线程安全，底层的数据结构跟正常的HashMap一样，都是数组+链表+红黑树的结构，之前的HashEntry也被改成了Node实现。  

## 1.8 put
主要步骤如下：  
> 1.根据Key算出HashCode。  
2.判断是否需要初始化。  
> 3.根据之前计算的HashCode定位到下标，通过CAS自旋的方式进行写入，如果自旋失败则往下执行。  
4.如果不满足，会对当前的Node结点进行加synchronized锁，进行put操作。  
5.添加完成之后会判断是否大于8，如果大于8要转换成红黑树。


![image](http://note.youdao.com/yws/res/3912/54A7AD96E45D4141A79042BFE7B38168)  

可以看到，相对于1.7 segement来说，synchronized锁范围更小，锁住也就只是当前结点的node，并不影响其他结点，锁颗粒度小，并发度更高。此外，synchronized是基于JVM的，JVM对synchronized有一个锁升级的过程，效率比较高。  

### CAS 与 ABA
CAS(Compare and swap)，主要关联有三个值：
- 当前变量的值
- 期望值
- 新值

只有当当前变量的值 == 期望值时，那么就把当前变量的值设置为新值。  

**有可能出现ABA问题，也就是另外一个线程快速的把当前变量从A修改成B再修改成A，对于原线程来说，当前变量的值还是A，但是实际上是已经被修改过的。**  

解决ABA问题可以用到版本号、时间戳等（具体可看atomic原子类）


## 1.8 get
跟1.7 一样，value是用volatile修饰的，能够第一时间从主存拿到最新的值。


## 1.8 size
1.8的size就有点复杂了，size是等于baseCount和counterCell数组各子项的和。   

size()和mappingCount()都可以返回size，但是size返回类型是int,而mappingCount返回的是long，推荐使用mappingcount，不会因为size方法限制最大值。
```java
//size 和 mappingCount都可以获取size
(int) concurrentHashMap.size();
(long) concurrentHashMap.mappingCount();

//主要方法sumCount()
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
关键：baseCount、CounterCell[]两个辅助变量，sumCount方法就是通过计算baseCount和counterCell的值得到value，实际上在put的时候会调用addCount方法。

```java
private final void addCount(long x, int check) {
CounterCell[] as; long b, s;
//通过CAS修改baseCount
if ((as = counterCells) != null ||
    !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
    //CAS修改baseCount失败，CAS修改counterCells
    CounterCell a; long v; int m;
    boolean uncontended = true;
    if (as == null || (m = as.length - 1) < 0 ||
        (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
        !(uncontended =
          U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
         //CAS修改counterCells失败，进入fullAddCount()，死循环直到成功。
        fullAddCount(x, uncontended);
        return;
    }
    if (check <= 1)
        return;
    s = sumCount();
}
```
首先会通过CAS修改baseCount，CAS修改baseCount失败，再通过CAS修改counterCells数组中的值，如果失败就会fullAddCount()一直死循环直到成功。

# 五、快速失败(fail-fast) & 安全失败(fail—safe)
快速失败是针对迭代器遍历时，因为modCount != expectmodCount然后带来的concurrentModifycation异常，哪怕是单线程也会。  

经典例子：
```java
Iterator it = list.iterator();
while(it.hasNext()){
    Integer number = iterator.next();
    if (number % 2 == 0) {
        // 抛出ConcurrentModificationException异常
        list.remove(number);
    }
    
    //it.remove() 就不会
}

```

安全失败，JUC下的容器都是安全失败的，因为他们在用迭代器遍历时，都不是直接遍历，是copy一份原内容进行遍历的。

# 六、参考资料
https://mp.weixin.qq.com/s/AixdbEiXf3KfE724kg2YIw
https://juejin.im/post/5be62527f265da617369cdc8
https://zhuanlan.zhihu.com/p/40627259