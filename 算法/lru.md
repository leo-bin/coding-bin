

## 什么是LRU？

```xml
1.Latest Recently Unuesd（最近最少未使用）。
2.如果一个数据在最近一段时间内都没有被访问，那么就认为这个数据在以后被访问的概率也很小了。
3.所以当内存满了的时候，这个数据就会最先被淘汰掉。
```



## 如何实现？

```xml
1.最朴素的思想就是用数组+时间戳的方式，不过这样做效率较低。
2.因此，我们可以用双向链表+哈希表（HashMap）实现（链表用来表示位置，哈希表用来存储和查找）。
3或者也可以利用Java中现成的数据结构LinkedHashMap来实现一个简单的LRU算法。
```



## 一  使用现有的数据结构实现

```java
package com.bins.algorithm.lru;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * @author leo-bin
 * @date 2020/2/23 19:50
 * @apiNote 使用LinkedHashMap实现
 */
public class LRUCache<K, V> extends LinkedHashMap<K, V> {

    /**
     * 初始缓存大小
     */
    private final int CACHE_SIZE;

    /**
     * 初始化缓存，并设置大小
     *
     * @param cacheSize 缓存大小
     */
    public LRUCache(int cacheSize) {
        //初始化一个LinkedHashMap，true表示按照访问顺序来进行排序，最近访问的数据放在尾部，最老访问的放在头部
        super((int) Math.ceil((cacheSize / 0.75) + 1), 0.75f, true);
        this.CACHE_SIZE = cacheSize;
    }

    /**
     * 是否移除最久没被访问的那个节点
     *
     * @return 是否满足删除条件
     */
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        //判断现在map中的数量是否达到了指定的缓存数
        //在调用put方法的时候，会调用这个方法，如果达到了缓存的大小，删掉最长时间没被访问的entry
        return size() > CACHE_SIZE;
    }
}
```



## 测试结果

![](https://gitee.com/leobins/coding-bin/blob/master/算法/img/lruV1测试结果.bmp)

```java
package com.bins.algorithm.lru;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * @author leo-bin
 * @date 2020/2/23 21:04
 * @apiNote LRU测试
 */
public class LRUTest {
    private static final Logger log = LoggerFactory.getLogger(LRUTest.class);

    /**
     * 初始化一个容量大小为10的LRU缓存
     */
    private static LRUCache<String, Integer> lruCache = new LRUCache<>(10);


    public static void main(String[] args) {
        //往缓存中加10个数，加入之后，缓存现在已经满了
        for (int i = 0; i < 8; i++) {
            lruCache.put("k" + i, i);
        }
        //先看看现在缓存中的情况
        log.info("all cache:{}", lruCache);

        //先访问一次k3
        lruCache.get("k3");
        log.info("get k3之后：{}", lruCache);

        //在访问一次k4
        lruCache.get("k4");
        log.info("get k4之后：{}", lruCache);

        //往容量已经满了的缓存中再加一个数k10,看看谁被淘汰了
        lruCache.put("k10", 10);
        log.info("put k10之后：{}", lruCache);
    }
}
```



## 二  使用HashMap+双向链表实现



```xml
原理：
```



```java
package com.bins.algorithm.lru;

import java.util.HashMap;

/**
 * @author leo-bin
 * @date 2020/4/7 19:44
 * @apiNote 使用HashMap+双链表+synchronized实现(线程安全)
 */
public class LRUCacheV2 {

    /**
     * 缓存容量
     */
    private final int cacheSize;
    private HashMap<String, Node> map;
    private Node head;
    private Node tail;

    public LRUCacheV2(int cacheSize) {
        this.cacheSize = cacheSize;
        this.map = new HashMap<>(16);
    }

    /**
     * 内部节点类
     */
    public class Node {
        private String key;
        private volatile String value;
        private Node pre;
        private Node next;

        public Node(String key, String value) {
            this.key = key;
            this.value = value;
        }
    }

    /**
     * get操作
     * 时间复杂度：O(1)
     */
    public String get(String key) {
        Node node = map.get(key);
        if (node == null) {
            return null;
        } else {
            refreshNode(node);
            return node.value;
        }
    }

    /**
     * put操作
     * 时间复杂度：O(1)
     */
    public void put(String key, String value) {
        Node node = map.get(key);
        synchronized (this) {
            if (node == null) {
                //缓存已满，需要删除最近最少使用的节点，其实就是头结点
                if (map.size() >= cacheSize) {
                    Node oldNode = removeNode(head);
                    map.remove(oldNode.key);
                }
                //没满或者删掉之后，放map中
                node = new Node(key, value);
                map.put(key, node);
                addNode(node);
            }
            //node已经存在，修改访问频率，并修改map中的value
            else {
                node.value = value;
                map.put(key, node);
                refreshNode(node);
            }
        }
    }

    /**
     * 刷新当前的链表
     */
    private void refreshNode(Node node) {
        if (node == tail) {
            return;
        }
        if (node != null) {
            removeNode(node);
            addNode(node);
        }
    }

    /**
     * 往链表尾部增加一个节点
     */
    private void addNode(Node node) {
        //1.head为空，那就将head指向node
        if (head == null) {
            head = node;
        }
        //2.tail不为空说明链表中有元素了，改变指向
        if (tail != null) {
            tail.next = node;
            node.pre = tail;
            node.next = null;
        }
        //3.tail无论如何都要先指向node
        tail = node;
    }

    /**
     * 随机删除链表中的一个节点
     */
    private Node removeNode(Node node) {
        //鲁棒
        if (node == null) {
            return node;
        }
        //1.要删除的节点是唯一节点
        if (node == head && node == tail) {
            head = null;
            tail = null;
        }
        //2.要删除的节点是head节点
        else if (node == head) {
            head = node.next;
            node.next.pre = null;
            node.next = null;
        }
        //3.要删除的节点是tail节点
        else if (node == tail) {
            tail = node.pre;
            node.pre.next = null;
            node.pre = null;
        }
        //4.要删除的节点是中间节点
        else {
            node.pre.next = node.next;
            node.next.pre = node.pre;
            node.next = null;
            node.pre = null;
        }
        return node;
    }
```



## 测试结果

![](https://gitee.com/leobins/coding-bin/blob/master/算法/img/lruV2测试结果.bmp)

```java
    public static void main(String[] args) {
        LRUCacheV2 lruCache = new LRUCacheV2(4);
        //初始化缓存，加四个元素进去
        lruCache.put("1", "User1");
        lruCache.put("2", "User2");
        lruCache.put("3", "User3");
        lruCache.put("4", "User4");

        //get一下其中一个元素User1
        System.out.println(lruCache.get("1"));

        //再加一个元素，加入之后User2应该被删掉了
        lruCache.put("5", "User5");

        //验证下User2是否被删掉了
        System.out.println("User2=" + lruCache.get("2"));
    }
```



## 总结

```xml
这里写测试结论和小结。。。
```

