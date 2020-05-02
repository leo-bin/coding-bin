# id生成算法



## 如何生成全局唯一id

```xml
如何生成全局唯一id
```



## 参考代码

```java
package com.bins.algorithm.idgenerator;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @author leo-bin
 * @date 2020/4/24 10:08
 * @apiNote 全局唯一id生成器，雪花算法
 */
public class SnowFlake {

    /**
     * 初始时间截 (2017-01-01)
     */
    private static final long INITIAL_TIME_STAMP = 1483200000000L;

    /**
     * 机器id所占的位数
     */
    private static final long WORKER_ID_BITS = 5L;

    /**
     * 数据标识id所占的位数
     */
    private static final long DATACENTER_ID_BITS = 5L;

    /**
     * 支持的最大机器id，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数)
     */
    private static final long MAX_WORKER_ID = ~(-1L << WORKER_ID_BITS);

    /**
     * 支持的最大数据标识id，结果是31
     */
    private static final long MAX_DATACENTER_ID = ~(-1L << DATACENTER_ID_BITS);

    /**
     * 序列在id中占的位数
     */
    private final long SEQUENCE_BITS = 12L;

    /**
     * 机器ID的偏移量(12)
     */
    private final long WORKERID_OFFSET = SEQUENCE_BITS;

    /**
     * 数据中心ID的偏移量(12+5)
     */
    private final long DATACENTERID_OFFSET = WORKER_ID_BITS + SEQUENCE_BITS;

    /**
     * 时间截的偏移量(5+5+12)
     */
    private final long TIMESTAMP_OFFSET = SEQUENCE_BITS + WORKER_ID_BITS + DATACENTER_ID_BITS;

    /**
     * 生成序列的掩码，这里为4095 (0b111111111111=0xfff=4095)
     */
    private final long SEQUENCE_MASK = ~(-1L << SEQUENCE_BITS);

    /**
     * 工作节点ID(0~31)
     */
    private long workerId;

    /**
     * 数据中心ID(0~31)
     */
    private long dataCenterId;

    /**
     * 毫秒内序列(0~4095)
     */
    private long sequence = 0L;

    /**
     * 上次生成ID的时间截
     */
    private long lastTimestamp = -1L;


    /**
     * 构造函数
     *
     * @param workerId     工作ID (0~31)
     * @param dataCenterId 数据中心ID (0~31)
     */
    public SnowFlake(long workerId, long dataCenterId) {
        if (workerId > MAX_WORKER_ID || workerId < 0) {
            throw new IllegalArgumentException(String.format("WorkerID 不能大于 %d 或小于 0", MAX_WORKER_ID));
        }
        if (dataCenterId > MAX_DATACENTER_ID || dataCenterId < 0) {
            throw new IllegalArgumentException(String.format("DataCenterID 不能大于 %d 或小于 0", MAX_DATACENTER_ID));
        }
        this.workerId = workerId;
        this.dataCenterId = dataCenterId;
    }

    /**
     * 获得下一个ID (用同步锁保证线程安全)
     *
     * @return SnowflakeId
     */
    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();
        // 如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
        if (timestamp < lastTimestamp) {
            throw new RuntimeException("当前时间小于上一次记录的时间戳！");
        }
        // 如果是同一时间生成的，则进行毫秒内序列
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & SEQUENCE_MASK;
            // sequence等于0说明毫秒内序列已经增长到最大值
            if (sequence == 0) {
                // 阻塞到下一个毫秒,获得新的时间戳
                timestamp = tilNextMillis(lastTimestamp);
            }
        }
        // 时间戳改变，毫秒内序列重置
        else {
            sequence = 0L;
        }
        // 上次生成ID的时间截
        lastTimestamp = timestamp;
        // 移位并通过或运算拼到一起组成64位的ID
        return ((timestamp - INITIAL_TIME_STAMP) << TIMESTAMP_OFFSET) | (dataCenterId << DATACENTERID_OFFSET)
                | (workerId << WORKERID_OFFSET) | sequence;
    }
    
    /**
     * 阻塞到下一个毫秒，直到获得新的时间戳
     *
     * @param lastTimestamp 上次生成ID的时间截
     * @return 当前时间戳
     */
    private long tilNextMillis(long lastTimestamp) {
        long timestamp = System.currentTimeMillis();
        while (timestamp <= lastTimestamp) {
            timestamp = System.currentTimeMillis();
        }
        return timestamp;
    }
```



## 测试代码

```java
    /**
     * note:
     * SnowFlake所生成的ID一共分成四部分
     * 1.第一位
     * 占用1bit，其值始终是0，没有实际作用
     * 2.时间戳
     * 占用41bit，精确到毫秒，总共可以容纳约140年的时间
     * 3.工作机器id
     * 占用10bit，其中高位5bit是数据中心ID（datacenterId），低位5bit是工作节点ID（workerId），最多可以容纳1024个节点
     * 4.序列号
     * 占用12bit，这个值在同一毫秒同一节点上从0开始不断累加，最多可以累加到4095
     * <p>
     * SnowFlake算法在同一毫秒内最多可以生成多少个全局唯一ID呢？只需要做一个简单的乘法
     * 同一毫秒的ID数量 = 1024 X 4096 = 4194304
     * 这个数字在绝大多数并发场景下都是够用的
     */
    public static void main(String[] args) {
        final SnowFlake idGenerator = new SnowFlake(1, 1);
        // 线程池并行执行10000次ID生成
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 0; i < 10000; i++) {
            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    long id = idGenerator.nextId();
                    System.out.println(id);
                }
            });
        }
        executorService.shutdown();
    }
}
```



## 测试结果

![JxuPYt.jpg](https://s1.ax1x.com/2020/05/02/JxuPYt.jpg)



## 总结

```xml
本次介绍了id生成算法雪花算法还有其他的全局唯一id生成方法
```

