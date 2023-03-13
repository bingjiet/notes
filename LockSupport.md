与 Object 的 wait/notify/notifyAll 对比
- wait-notify/notifyAll 必须配合 Object Monitor 使用，而 park unpark 不需要
- park & unpak 是以线程为单位去阻塞线程和唤醒线程，可以精确到唤醒哪个线程，而 notify 只能唤醒其中一个线程，notifyAll 唤醒所有等待线程
- park & unpark 先后可以翻转，而 wait/notify&notifyAll 不行


每个线程都有一个 parker 对象，由 _count，_cond, _mutex 三部分组成

只要 _count 为 1 就可以一直运行下去, unpark 方法会让 _count 变成1

 

当线程调用 unsafe.park 方法
会检查 counter 是否为0，如果为0，这时候会获取 mutex 互斥锁
线程进入 cond 进行阻塞
设置 counter = 0

unpark 
让 counter 变成 1，唤醒 cond 的线程，线程执行，最后把 counter 设为 0

先 unpark 后 park
让 counter 设置成 1，线程判断 counter 发现不为 0，则继续运行，把 counter 设成 0

线程状态

java 的六种状态

RUUNNABLE 包含可运行，运行，阻塞三种状态

- NEW
- RUNNABLE
- BLOCKED
- WAITING
- TIMEED_WAITING
- TERMINATED


wait
RUNNABLE -> WAITING
notify, notifyAll, interupt
- 竞争成功 WAITING -> RUNNABLE
- 竞争失败 WAITING -> BLOCKED

join
RUNNABLE -> WAITING
让调用这个方法的线程进入等待
线程结束，或者调用 interupt
WAITING -> RUNNABLE

park
RUN -> WAITING
unpark or interupt 
WAITING -> RUNNABLE


RUNNABLE -> TIMED_WAITING
wait(timeout), join(timeout), sleep(timeout)

获取锁失败
RUNNABLE -> BLOCKED

执行完毕
RUNNABLE -> TERMINATED

锁的活跃性
死锁
t1 获取 A 锁，然后获取 B 锁
t2 获取 B 锁，然后获取 A 锁

查找死锁
可以 jps 查找pid
然后 jstack pid

活锁
活锁出现在两个线程互相改变对方的结束条件，从而导致无法结束
时间交错开，一些随机的随眠时间，可以避免活锁

饥饿
让某些线程获取CPU的时间片很多

ReentranLock
与 syn 对比
- 可中断
- 可以设置超时时间
- 可以设置公平锁
- 支持多个条件变量
- 支持可重入
