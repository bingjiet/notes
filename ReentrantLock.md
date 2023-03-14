- 可重入
- 可打断
  - lock.lockInterruptibly(), 可以让被其他线程打断
  ```java
  private static ReentrantLock lock = new ReentrantLock();
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            try {
                System.out.println("获取可打断锁");
                lock.lockInterruptibly();
            } catch (InterruptedException e) {
                System.out.println("锁被打断");
                throw new RuntimeException(e);
            }

            try {
                System.out.println("获取到锁");
            } finally {
                lock.unlock();
            }
        }, "t1");

        lock.lock();
        t1.start();

        System.out.println("等待 5s 后打断");
        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        t1.interrupt();

    }
  ```
  
 - 锁超时
  - lock.tryLock()，尝试获取锁，返回值是布尔值，返回 true 则获取到锁，反之获取锁失败
  - lock.tryLock(long n) 带超时的时间，尝试获取锁，超过时间就不再获取，可以被其他线程打断

- 公平锁
  通过构造方法开启是否公平
  
- 条件变量
  - 支持多个条件变量
    
  
