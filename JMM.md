JMM
定义了主存，工作内存的抽象概念，底层对应着 CPU 寄存器、缓存、硬件内存、CPU 指令优化等

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
