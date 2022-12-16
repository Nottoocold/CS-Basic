# JUC

<a href="https://pdai.tech/md/java/thread/java-thread-x-lock-AbstractQueuedSynchronizer.html">参考文章</a>

## AQS简介

**AQS是一个用来构建锁和同步器的框架**，使用AQS能简单且高效地构造出应用广泛的大量的同步器，比如我们提到的ReentrantLock，Semaphore，其他的诸如ReentrantReadWriteLock，SynchronousQueue，FutureTask等等皆是基于AQS的。当然，我们自己也能利用AQS非常轻松容易地构造出符合我们自己需求的同步器。

## AQS核心思想

AQS核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制AQS是用CLH队列锁实现的，即将暂时获取不到锁的线程加入到队列中。

AQS使用一个int型变量**记录同步状态**，通过内置的FIFO队列来完成获取资源线程的排队工作。使用CAS操作对该变量进行修改。

```java
private volatile int state;
```

### AQS对资源的共享方式

定义了两种资源共享的方式：

- Exclusive(独占)，只有一个线程能执行，如ReentrantLock。又可分为公平锁和非公平锁：
  
  - 公平锁，按照排队顺序
  
  - 非公平锁，不按照排队顺序，谁抢到就是谁的。

- Share(共享)，如Semaphore/CountDownLatch。Semaphore、CountDownLatCh、 CyclicBarrier、ReadWriteLock。

不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源 state 的获取与释放方式即可，至于具体线程等待队列的维护(如获取资源失败入队/唤醒出队等)，AQS已经在上层已经帮我们实现好了。

## AQS类的内部类Node

```java
static final class Node {
    // 模式，分为共享与独占
    // 共享模式
    static final Node SHARED = new Node();    
    // 独占模式
    static final Node EXCLUSIVE = null;

    // 节点状态
    /*
        CANCELLED, 1, 表示当前线程被取消
        SIGNAL, -1, 表示当前线程的nextWaiter需要被unpark
        CONDITION, -2, 表示当前线程处于等待条件状态，即处于ConditionQueue中
        PROPAGATE， -3, 表示当前场景下后续的acquireShared可以执行
        值为0，表示当前节点在sync队列中，等待着获取锁  
    */
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;

    volatile int waitStatus; // 结点状态
    volatile Node prev; // 前驱结点
    volatile Node next; // 后继结点
    volatile Thread thread; // 结点封装的当前线程
    Node nextWaiter; // 下一个等待者
}
```

## AQS内部类ConditionObject

```java
public class ConditionObject implements Condition {
        Node firstWaiter;
        Node lastWaiter;

        /**
         * Adds a new waiter to wait queue.
         * @return its new wait node
         */
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters(); // 清除不是CONDITION的结点
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }

        /**
         * Removes and transfers nodes until hit non-cancelled one or
         * null. Split out from signal in part to encourage compilers
         * to inline the case of no waiters.
         * @param first (non-null) the first node on condition queue
         */
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
            // 当将first结点转至sync队列失败且fisrtWaiter不是null
        }

        // doSignalAll同理

        // 将所有不为CONDITION的结点移出Condition Queue
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null; // 跟踪尾结点
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null) //当前处于队头且需要清除
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }


}
```