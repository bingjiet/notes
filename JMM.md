JMM

定义了主存，工作内存的抽象概念，底层对应着 CPU 寄存器、缓存、硬件内存、CPU 指令优化等

分为主存，工作内存，线程会从主存中复制共享变量的副本到工作内存中，线程执行的时候，操作的是工作内存的数据，这时候会出现缓存一致性问题，通过 LOCK 指令，锁总线或者缓存锁实现缓存一致性的问题。

MESI 缓存一致性协议，解决缓存一致性的问题

代表缓存行的状态

- Modified 对缓存行进行修改
- Exclusive 独占，占用这个缓存行，但是还没做修改，如果这个时候有其他 CPU 过来拿这个缓存行，直接给他即可，而 modify 需要把缓存行的数据回刷到主存中
- Shared 共享，各个 CPU 都能读取到，不做任何的修改
- Invalid 失效，当其中一个 CPU 进行修改缓存行的时候，会发出通知，其他 CPU 感知到，如果有给缓存行的数据就将其设置成无效

简单的来说，就是当其中一个CPU改变了工作内存中共享变量的值的时候，通过总线嗅探的机制，使其他CPU工作内存中该共享变量失效，使其到主存中获取最新的值。



perfbook

- read 
- read response 
- invalidate 
- invalidate Acknowledge 
- Read invalidate
- writeback



当其中一个 CPU 修改了共享变量的值的时候，对发出一个消息告诉其他 CPU 该共享变量失效，其他 CPU 会返回 ACK 响应，然后这个 CPU 把更新的值刷新到主存中，其他 CPU 到主存中获取新的值。

这里 CPU 等待 ACK 响应是同步的，因此如果等待多个 ACK 的响应，效率就会变低，后面引入了 store buffer 提高效率，引入了 store buffer 后，CPU 在写共享变量的时候，这个值先写入 store buffer，然后通知其他 CPU 缓存失效的信息，CPU 执行异步的操作 。如下面的代码，一开始读取 a = 0 到工作内存中，CPU 0 对 a 进行写操作，a = 10 这个值写入了 store buffer 中，并且通知其他 CPU 该变量的缓存失效，这个时候 CPU 0 是异步执行，因此还没收到 ACK 的时候，就执行了后面的代码，这个时候一共有两个缓存，一个是工作内存的缓存 a，一个 store buffer 中的 a，进行计算的时候用的是工作内存中的缓存 a ，从而导致计算的结果等于 10，这个时候 ACK 过来，工作内存的缓存被更新，再更新到主存中，结果仍然是错误的，因为发生了类似重排序的操作，这个时候加入一个写屏障，

```java
private int a = 0;

public void m1() {
	int a = 10;
	int b = a + 10;
}
```



因为 ACK 可能不会及时返回，那么写数据可以先写在 Store Buffe 里面，然后 CPU 就去执行下一条指令，这里就会出现指令重排序，因为内存原因出现的指令重排序，因为数据只是写在 Store Buffer 里面，并没有真实的写入，然后就执行下一条指令，相当于下一条指令重排在写入之前



CPU 0 对 a = 1 的写，先放入了 Store Buffer 中，发送 read invalidate 的消息，告诉 CPU 1 要将你的缓存行设置成无效，接着执行下面的代码，发现缓存行没有 a 的值，只能问 CPU 1 要，这个时候 a 为 0，因此结果是错误的

```java
// CPU 0 执行以下代码，a 在 CPU 1 上，初始值为 0
a = 1;
b = a + 1;
assert(b == 2);
```



CPU 0 执行，缓存行没有 a，写入 store buffer 里面，同时直接更改 b = 1

CPU 1 执行，缓存行没有 b，那么就问 CPU 0 拿，结束循环，这个时候 a 在缓存行中，a = 0 ，结果是错的

可以理解成，没有写入到缓存行，就执行了 b =1 ，导致 CPU1 的结果错误

```java
// CPU 0 执行 foo(), 拥有 b 的缓存行，初始值0 
void foo() {
    a = 1;
    
    b = 1;
}

// CPU1 执行 bar(), 拥有 a 的缓存行，初始值0
void bar() {
    while(b ==0) continue;
    assert(a == 1);
}
```



加了写屏障后，仍然会将 a = 1 写到 store buffer 中，然后看到有一个写屏障，就将下面的指令也写入到 store buffer 中，即当前的 store buffer 里面的存的是 a = 1, b = 1，并且是有序的

这个时候 CPU 1 问 CPU 0 的缓存行拿数据只能读到 b = 0，因为 CPU 1是读不到 store buffer 的值的

```java
// CPU 0 执行 foo(), 拥有 b 的缓存行，初始值0 
void foo() {
    a = 1;
    // 写屏障
    smp_wmb();
    b = 1;
}

// CPU1 执行 bar(), 拥有 a 的缓存行，初始值0
void bar() {
    while(b ==0) continue;
    assert(a == 1);
}
```





store buffer 是有限的，当写满了之后，CPU仍然会在这里阻塞等待，当有了smp_wmb() 屏障后，更容易让 store buffer 写满，因为有了这个屏障，之后的指令都是写入 store buffer 中，引入 invalidate queue，当有 invalidate 请求的时候，便将请求丢到队列中去，然后立即回复 ACK，但是这个时候并没有处理这个 invalidate 



CPU 0 a = 1 写入请求放到 store buffer 中，因为有内存屏障，所以 b = 1 都写入了 store buffer 并且发出一个invalidate 请求，这个时候 CPU 1 回复了 ACK ，此时 CPU 0 就将 store buffer 的值写入到缓存行中，这个时候 CPU1 执行，向 CPU 0 的缓存行读取这个 b 的值，退出循环，此时因为还没有真正处理这个 invalidate  的消息，因为这个时候的 a 的状态不是 invalidate，因此使用的还是 a 缓存行的旧数据，结果就是错误，这个相当于重排序了，将 assert(a == 1) 排到了 while(b) ，没有看到 b 值新的变化

```java
// CPU 0 执行 foo, a share, b exclusive
void foo() {
    a = 1;
    smp_wmb();
    b = 1;
}

// CPU 1 执行， a 处于 share
void bar() {
    while(b ==0) continue;
    assert(a == 1);
} 
```



加入了读屏障，必须强制处理 invalidate 队列中所有的消息，这个时候读到缓存行的 a 就是一个 invalidate 状态，那么就去读最新值

```java
// CPU 0 执行 foo, a share, b exclusive
void foo() {
    a = 1;
    smp_wmb();
    b = 1;
}

// CPU 1 执行， a 处于 share
void bar() {
    while(b ==0) continue;
    smp_rmb(); // 读屏障
    assert(a == 1);
} 
```



Java 内存模式，实际上就一套规范，抽象出来线程，工作内存，Save和Load操作，和主内存的概念

JVM 的内存屏障

读屏障，后面的代码不能重排序到读屏障前面

写屏障，写屏障前的代码不能重排序到写屏障后

- LoadLoad 读屏障，在两个读取操作上加这个屏障，后一次读操作不能重排到前一次读操作
- StoreStore 写屏障，在两个写操作上加这个屏障，后一次的写操作不能重排序到前一次写操作前
- LoadStore 后一次写操作不会重排序到前面的写操作
- StoreLoad 写和读都要管，对应CPU的操作就是 store 要同步更新到 store buffer ，invalidate queue 的值一定强制处理，后一次的读取不会重排序到前一个写操作，这个其实就是确保读取到的值是最新值



volatitle 

- 可见性 加了之后，CPU 线程都能读到这个共享变量的最新值

volatitle store 在后

- Normal Store, Volatile Store 用 StoreStore
- Volatile Store, Volatile Store 用 StoreStore
- Normal load, Volatile Store 用 loadStore
- Volatile load, Volatile Store 用 loadStore

volatitle load 在前

- Volatile Load, Volatile Load   LoadLoad
- Volatile Load, Normal Load   LoadLoad
- Volatile Load, Normal Store LoadStore

特殊情况

- Volatile Store, Volatile Load StoreLoad 后面读最新值，肯定不能重排序到修改前

只要后面是 Volatile Store  ，不管前面是什么，都要加一个 xxxxStore 屏障

只要前面是 Volatile Load，不管后面是什么，都要加一个 LoadXXXX屏障

如果前面是写，后面是读，就加一个 StoreLoad，注意都是 volatitle 读 和 volatitle 写



注意 X86 是没有 invalidate queue 的

实际上就是加了 volatitle 之后，会转成一个不同CPU实现的内存屏障



在写之前加屏障，在读之后加屏障

操作数据实际上还是在缓存上操作，刷新到主存的时机不确定

可见性维护

- 读的方面
  - 不使用寄存器
  - 禁止 volatitle 向后重排
- 写方面
  - 禁止写volatitle 向前重排







JMM

- 原子性，保证指令不会受到线程上下文切换的影响
- 可见性，保证指令不会受 cpu 缓存的影响
- 有序性，保证指令不会受 cpu 指令并行优化的影响

volatile
可以用来修饰成员变量和静态变量，避免线程从工作内存中读取共享变量，必须到主存中读取共享变量，线程操作volatile变量都是操作主内存的

syn
也可以保证可见性

指令重排
jvm 在不影响结果的前提下，有可能进行指令重排序，在单线程下结果会是正确的，但是在多线程下，可能会出现结果错误的情况
当下一条指令的计算依赖于上一条指令的的时候，无法进行重排序
单线程环境下 m2 方法的发生重排序，是不会影响结果
但是在多线程环境下，m2 方法如果发生指令重排序, flag = true 就会先执行，同时如果这个时候发生线程上下文切换，m1 的结果就会变成 0

```java
int num = 0;
boolean flag = false;
int result
public void m1() {
  if(flag) {
    result = num + num;
  } else {
    result = 1;
  }
}

//发生指令重排序
public void m2() {
  num = 2;
  flag = true
}
```
volatile 的底层原理是内存屏障，
- 对 volatile 变量的写指令后会加入写屏障
- 对 volatile 变量的读指令前会加入读屏障

保证可见性
写屏障保证在该屏障之前的，对共享变量的改动，都同步到主存中
读屏障保证在该屏障之后的，对共享变量的读取，加载的是主存中的值

保证有序性
写屏障会确保指令重排序时，不会将写屏障之前的指令排在写屏障之后
读屏障会确保指令重排序时，不会将读屏障之后的代码排在读屏障之前

无法保证指令交错，指令交错是 CPU 调用的，保证的是线程内的重排，可见性和有序性，不能保证原子性

DCL 问题
在 new 对象的时候，有可能发生指令重排序，导致对象还没被初始化 init 调用构造方法，就已经被赋值，此时第二个线程进来，判断 INSTANCE 是否为空，此时 INSTANCE 已经被赋值了，所以返回的是一个 Broken 对象

happen-before
happen-before 规定了对共享变量的写操作对其它线程的读操作可见，它是可见性与有序性的一套规则总结，抛开这套规则，JMM 不能保证一个线程对共享变量的写，对其它线程对该共享变量的读可见

- syn 下线程对共享变量的写，对其它线程的读可见
```java
public static void main(String[] args) {
        new Thread(() -> {
            synchronized (lock) {
                x = 10;
            }
        }, "t1").start();

        new Thread(() -> {
            synchronized (lock) {
                System.out.println(x);
            }
        }, "t2").start();
    }
```
- volatile 修饰的共享变量，线程的写，对其它线程读可见
```java
static voliate int x = 0;
public static void main(String[] args) {
        new Thread(() -> {
            x = 10;
        }, "t1").start();

        new Thread(() -> {
            System.out.println(x);
        }, "t2").start();
    }
```
- 线程 start 前对共享变量的写，对该线程开始后对共享变量的读可见，因为开始之后，才会把主存的变量复制一份到工作内存中
- 线程结束后对变量的写，对其它线程得知它结束后的读可见，因为线程结束后，会将变量写入到主存中，join/alive
- 线程t1 打断 t2 线程的写，对得知 t2 被打断的读可见
- 对变量默认值（0，false, null） 的写，对其它线程的读可见
- 具有传递性，其实是因为 volatile，因为 volatile 的写屏障，会让写屏障前面的代码全部写入到主存中
