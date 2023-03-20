通过创建副本对象来避免共享的手段称为保护性拷贝



享元模式属于结构模式

JDK 中的包装对象都是运用了享元模式，尽可能的让重复对象复用，Long 对象在 -128 - 127 范围内，不会创建新的 Long 对象，而是重用这些对象，当超过这个范围之后，才会去创建对象

在对象进行静态初始化的时候，就已经创建好这些对象存放在 cache 的 Long 数组中

```java
private static class LongCache {
	private LongCache(){}

	static final Long cache[] = new Long[-(-128) + 127 + 1];

	static {
		for(int i = 0; i < cache.length; i++)
		cache[i] = new Long(i - 128);
	}
}
```

Byte, Short, Long 的缓存范围都是 -127 - 128

Character 的缓存范围是 0 - 127

Integer 的默认范围值是 -128 - 127，最小值无法改变，但是最大值可以虚拟机参数进行调整

-Djava.lang.Integer.IntegerCache.hign

Boolean 缓存了 TRUE 和 FALSE



BigDecimal 和 BigInteger 都是不可变类，都用了保护性拷贝，是线程安全的，虽然方法是原子性的，但是在多个方法执行的时候，并不是线程安全的



final 原理

```java
fina int i = 0
```

final 变量在赋值的时候通过 putfield 指令完成，这条指令会加入写屏障，在初始化 i 的时候有可能出现 i 先初始化但是未赋值的可能性，通过写屏障确保它们的指令不会被重排序

final 在不同类中的读，直接复制一份值到栈中



### final

![image-20230320221648366](C:\Users\30624\AppData\Roaming\Typora\typora-user-images\image-20230320221648366.png)

![image-20230320221911834](C:\Users\30624\AppData\Roaming\Typora\typora-user-images\image-20230320221911834.png)

![image-20230320222946311](C:\Users\30624\AppData\Roaming\Typora\typora-user-images\image-20230320222946311.png)

![image-20230320222542723](C:\Users\30624\AppData\Roaming\Typora\typora-user-images\image-20230320222542723.png)

![image-20230320223218727](C:\Users\30624\AppData\Roaming\Typora\typora-user-images\image-20230320223218727.png)

![image-20230320224236271](C:\Users\30624\AppData\Roaming\Typora\typora-user-images\image-20230320224236271.png)

![image-20230320225104731](C:\Users\30624\AppData\Roaming\Typora\typora-user-images\image-20230320225104731.png)

![image-20230320225442847](C:\Users\30624\AppData\Roaming\Typora\typora-user-images\image-20230320225442847.png)

![image-20230320231139504](C:\Users\30624\AppData\Roaming\Typora\typora-user-images\image-20230320231139504.png)

![image-20230320231148615](C:\Users\30624\AppData\Roaming\Typora\typora-user-images\image-20230320231148615.png)