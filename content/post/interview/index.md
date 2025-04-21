---
title: 美团面经
date: 2025-04-21
description: 美团面试题目和手撕算法题
tags: 
categories:
    - interview

---

## 一面

算法：翻转链表
解法：一个数组存储然后反向遍历构成新链表【空间复杂度 O(n) 】；链表下一个指针指向前一个

1. JVM 的内存结构
2. 怎么判断一个对象是否可以被回收？
3. 垃圾收集器有哪些？
4. 垃圾回收算法有什么？
5. redis用在项目的哪里？
6. redis的底层逻辑是什么？
7. 说一下多路复用是什么？
8. 线程池有哪些核心参数？
9. 怎么设置线程池的核心参数？
10. 介绍SpringIOC的原理
11. 介绍SpringAOP的原理
12. Spring、SpringMVC、SpringBoot的区别是什么？
13. 介绍mysql的索引
14. mysql的隔离级别是什么？
15. 访问百度整个流程是什么？
16. 如果访问一个网站一直转圈圈你会怎么排查？
17. tcp 和 udp的区别
18. tcp 粘包怎么解决？
19. 平时怎么学习新技术？
20. 会大模型吗？

## 二面

1. **项目：**
   - 文献表的mysql字段怎么设计？
   - 用到什么索引？
   - redis的用意是什么？
   - 为什么要用es？（不是因为es比mysql更快）

2. **mysql事务并发题：说说 Q1-Q4 是怎么执行的？Q4 查询的时候 b 是多少？**

    表：
    |a |b |c |
    |-|-|-|
    |1 |1 |1 |

    > 隔离级别：可重复读

    ```sql
    t1 start
    t2 start
    t1: UPDATE 表 SET b=b+1 WHERE a=1; // Q1
    t2: UPDATE 表 SET b=b+1 WHERE a=1; // Q2
    t1 submit
    t2: UPDATE 表 SET b=b+1 WHERE a=1; // Q3
    t3 start
    t3: SELECT * FROM 表 WHERE b=4; // Q4
    t2 submit
    t3 submit
    ```

    - 分析：
        - 可重复读是采用 MVCC 的方式来解决并发读写的问题（读取事务开始前的快照）。
        - UPDATE 会读取当前最新值（即使是在 REPEATABLE READ 下），但 SELECT 会读取快照。
    - Q1：t1 成功获取写锁并更新 b=2，但是还没提交。
    - Q2：t2 获取写锁失败，因此进入阻塞状态，等待 t1 释放锁。在 t1 提交后成功获取写锁并更新 b=3，但是还没提交。
    - Q3：t2 这时已经获取写锁，直接继续进行更新 b=4，但是还没提交。
    - Q4：t2 还未提交，t3 是 REPEATABLE READ 隔离级别（MVCC），会读取 t3 事务开始时的快照，也就是 t1 提交后的版本 b=2。b=4 是 t2 未提交的修改，t3 看不到，因此 Q4 查询不到任何数据。

3. **mysql索引题：以下是各表执行频率高达90%的查询语句，为每个表的字段建立联合索引**

    ```sql
    SELECT * FROM 表 WHERE a>1 AND b=2 // (b,a)
    SELECT * FROM 表 WHERE a=1 AND b=2 AND c>3 // (a,b,c), (b,a,c)
    SELECT * FROM 表 WHERE a=1 ORDER BY b // (a,b)
    SELECT * FROM 表 WHERE a=1 AND b=2 ORDER BY c // (a,b,c), (b,a,c)
    SELECT * FROM 表 WHERE a=1 AND b IN(1,2,3) // (a,b)
    SELECT * FROM 表 WHERE a=1 AND b=2 AND c IN(1,2,3) // (a,b,c), (b,a,c)
    SELECT * FROM 表 WHERE a=1 AND b IN(1,2,3) ORDER BY c // (a,b), (a,c)
    ```

    1. b 等值查询，a 范围查询
    
    2. a 和 b 等值查询，c 范围查询
    
    3. a 等值查询，b 排序（由于是 b+ 数结构，b 建立索引可以让叶子结点自动排好序，遍历所有叶子结点的这个链表就是排好序的）
    
    4. a 和 b 等值查询，c 排序
    
    5. IN 查询是 OR，虽然 OR 不走索引，但是会这条语句会变成：(a=1 AND b=1) OR (a=1 AND b=2) OR (a=1 AND b=3)，所以 a 是等值查询，b 属于范围查询，可以建索引
    
    6. a 和 b 等值查询，c IN（范围查询）
    
    7. 如果是建 (a,b,c) 就会导致 b 范围查询之后 c 索引失效；如果是建 (a,c,b) 就会导致 c 排序之后 b 索引失效；所以只能选其中一个
    
    **分析：联合索引设计原则**
      - 最左前缀原则：MySQL 使用索引时，必须从最左列开始，不能跳过中间列。
      - 等值查询优先：= 条件比 >、IN、ORDER BY 优先级更高，应该放在索引左侧。
      - 范围查询放最后：>、<、BETWEEN、IN 等范围查询应尽量放在索引右侧，避免影响后续列的索引使用。
      - 排序优化：如果查询包含 ORDER BY，尽量让排序字段在索引中，并且顺序与索引一致。
      - 覆盖索引：如果查询只涉及索引列，可以避免回表，提高性能。
      - mysql优化器：优化器倾向于选择区分度高的列作为索引的前缀（调整查询条件的执行顺序）；如果 ORDER BY 的列在索引中且顺序一致可以避免 filesort

4. **算法题：利用数组构建一个大小为 n 的双向队列，要求实现以下 6 个方法，且支持高并发下读多写少的情况：**
   - 入队
   - 出队
   - 查询队尾元素
   - 查询队头元素
   - 返回队列是否为空
   - 返回队列是否为满
  
    分析：
    - 使用读写锁 ReentrantReadWriteLock：对于读是共享锁，写是排它锁
    - 使用泛型提高复用性
    - 成员变量用 private 修饰，方法用 public 修饰
    - 灵活使用异常
    - 辅助垃圾回收
    - 这是一个容器类，不能用 static 静态属性修饰
    - 只涉及设置临界区，不涉及同步机制

```java
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ConcurrentArrayDeque<T> {
    private final T[] array;
    private int head; // 队头指针
    private int tail; // 队尾指针
    private int count; // 元素数量
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock(); // 锁
    private final ReentrantReadWriteLock.ReadLock readLock = lock.readLock(); // 读锁
    private final ReentrantReadWriteLock.WriteLock writeLock = lock.writeLock(); // 写锁

    public ConcurrentArrayDeque(int capacity) { // 构造方法
        if (capacity <= 0) throw new IllegalArgumentException("Capacity must be positive");
        this.array = (T[]) new Object[capacity]; // 强制转换
        this.head = 0;
        this.tail = 0;
        this.count = 0;
    }

    // 入队 (队尾)：写操作
    public boolean offerLast(T item) {
        if (item == null) throw new NullPointerException();
        writeLock.lock();
        try {
            if (isFull()) return false;
            array[tail] = item;
            tail = (tail + 1) % array.length;
            count++;
            return true;
        } finally {
            writeLock.unlock();
        }
    }

    // 出队 (队头)：写操作
    public T pollFirst() {
        writeLock.lock();
        try {
            if (isEmpty()) return null;
            T item = array[head];
            array[head] = null; // 帮助GC
            head = (head + 1) % array.length;
            count--;
            return item;
        } finally {
            writeLock.unlock();
        }
    }

    // 查询队头元素：读操作
    public T peekFirst() {
        readLock.lock();
        try {
            if (isEmpty()) return null;
            return array[head];
        } finally {
            readLock.unlock();
        }
    }

    // 查询队尾元素：读操作
    public T peekLast() {
        readLock.lock();
        try {
            if (isEmpty()) return null;
            return array[(tail - 1 + array.length) % array.length];
        } finally {
            readLock.unlock();
        }
    }

    // 队列是否为空：读操作
    public boolean isEmpty() {
        readLock.lock();
        try {
            return count == 0;
        } finally {
            readLock.unlock();
        }
    }

    // 队列是否为满：读操作
    public boolean isFull() {
        readLock.lock();
        try {
            return count == array.length;
        } finally {
            readLock.unlock();
        }
    }

    // 返回队列当前大小：读操作
    public int size() {
        readLock.lock();
        try {
            return count;
        } finally {
            readLock.unlock();
        }
    }
}

class Test {
    public static void main() {
        ConcurrentArrayDeque<Integer> deque = new ConcurrentArrayDeque<>(5);
        // 生产者线程
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                boolean success = deque.offerLast(i);
                System.out.println("Offer " + i + ": " + success);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
        // 消费者线程
        new Thread(() -> {
            while (true) {
                Integer item = deque.pollFirst();
                if (item != null) System.out.println("Poll: " + item);
                try {
                    Thread.sleep(150);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
```