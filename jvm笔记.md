Hotspot JVM

#### 定义

Java Virtual Machine（java程序的运行环境）

#### 好处

- 一次编写，到运行
- 自动内存管理，垃圾回收功能
- 数组下标越界检查
- 多态

#### 比较jvm & jre & jdk

![image-20231209104404455](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209104404455.png)

JRE = JVM + 基础类库

JDK =JVM + 基础类库 + 编译工具

## 学习路线

![image-20231209105031318](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209105031318.png)

- jvm内存结构
- gc垃圾回收
- java class
- classloader
- jit compiler

## 内存结构

### 程序计数器（寄存器）

![image-20231209105639708](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209105639708.png)

作用：指出下一条jvm指令的执行地址。因为要频繁地存取下一条jvm指令的执行地址，所以需要使用寄存器来作为程序计数器，提高性能。

特点：

- 线程私有的。每个线程都有一个程序计数器
- 不会存在内存溢出

### 栈

虚拟机栈：每个线程运行时需要的内存空间（存储栈帧）

栈帧：每个方法运行时需要的内存（参数，局部变量，返回地址）

活动栈帧：每个线程只能有一个活动栈帧（栈顶元素），对应着当前正在执行的方法

占内存溢出：`Java.lang.stackOverflowError` 

#### 问题辨析

1.垃圾回收是否涉及栈内存？

垃圾回收机制不会涉及栈内存，只会回收堆内存中的对象。

2.栈内存分配越大越好吗?

并不是这样的，当栈内存分配的越大，那么能创建的线程数目则将减少，因为每个线程私有一个虚拟机栈。我们一般使用默认的虚拟机栈大小，在Linux、MacOS、Solaris中，默认是1024KB；Windows中默认虚拟机栈大小取决于虚拟机内存，也可以使用`-Xss`指定虚拟机栈大小（`-Xss256k`）。

3.方法内的局部变量是否线程安全?

对于局部变量：每次方法调用会产生一个栈帧，栈帧内有私有的局部变量，所以是线程安全的。如果是static变量，则多个线程共享此变量，不是线程安全的。

#### 判断线程安全

```java
public class main1 {
    public static void main(String[] args) {

    }
    //下面各个方法会不会造成线程安全问题？

    //不会
    public static void m1() {
        StringBuilder sb = new StringBuilder();
        sb.append(1);
        sb.append(2);
        sb.append(3);
        System.out.println(sb.toString());
    }

    //会，可能会有其他线程使用这个对象
    public static void m2(StringBuilder sb) {
        sb.append(1);
        sb.append(2);
        sb.append(3);
        System.out.println(sb.toString());
    }

    //会，其他线程可能会拿到这个线程的引用
    public static StringBuilder m3() {
        StringBuilder sb = new StringBuilder();
        sb.append(1);
        sb.append(2);
        sb.append(3);
        return sb;
    }
    
}
```

第一个是线程安全的。第二个因为接收外界参数，可能造成数据共享，则可能存在线程安全问题。第三个因为有方法返回值（对象），存在局部变量逃离了作用域的访问，则不是线程安全的。

#### 占内存溢出

- 栈帧过多（递归中，无限地调用本方法；循环引用）

- 栈帧过大

#### 线程运行诊断

定位：

- top：定位是哪个进程对cpu占用过高
- ps H -eo pid,tid,%cpu | grep pid：刚才通过top查到的进程号，用ps命令进一步定位是哪个线程引起的cpu占用过高
- jstack pid：通过查看进程中的线程的nid，刚才通过ps命令看到的tid来对比定位，注意jstack查找出的nid是16进制的，需要转换

### 本地方法栈

一些带有**native关键字**的方法就是需要JAVA去调用本地的C或者C++方法，因为JAVA有时候没法直接和操作系统底层交互，所以需要用到本地方法去调用底层功能。

### 堆

- 通过new关键字创建的对象都使用堆内存

特点

- 堆中的对象是线程共享的，堆中的对象都需要考虑线程安全问题
- 有垃圾回收机制

#### 堆内存溢出

堆内存溢出：`java.lang.OutofMemoryError ：java heap space`

可以通过参数指定堆空间大小`-Xmx`，默认是4g（设置堆内存`-Xmx2g`）

堆空间较大时，不容易堆内存溢出；如果要排查堆内存溢出错误，则要将堆内存尽量设小一些，尽早暴露堆内存溢出这一问题。

一个会引起堆内存溢出的例子：

```java
/**
 * 演示堆内存溢出 java.lang.OutOfMemoryError: Java heap space
 * -Xmx8m ，最大堆空间的jvm虚拟机参数，默认是4g
 */
public class main1 {
    public static void main(String[] args) {
        int i = 0;
        try {
            ArrayList<String> list = new ArrayList<>();// new 一个list 存入堆中
            String a = "hello";
            while (true) {
                list.add(a);// 不断地向list 中添加 a
                a = a + a;
                i++;
            }
        } catch (Throwable e) {// list 使用结束，被jc 垃圾回收
            e.printStackTrace();
            System.out.println(i);
        }
    }
}
```

jps工具

- 查看当前系统中有哪些java进程，可以获取进程id

jmap工具

- 查看某进程堆内存占用情况，只能查询某一时刻某进程的堆内存占用情况，不能连续监控

jconsole工具

- 图形界面，多功能监测工具，可以连续监测

实际中，可以先使用`jps`查看系统中有哪些java进程，然后根据pid查询某进程堆内存占用情况（jdk8以后的用法）

```java
jhsdb jmap --heap --pid 3616
```

也可以使用`jconsole`图形化界面，来实时监控

![image-20231209140708743](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209140708743.png)

也可以使用`visualvm`来更加细致化地监控是哪个变量导致堆内存占用过高

### 方法区

所有java虚拟机共享的区域，它用于存储已被虚拟机加载的类信息（比如class文件）、常量、静态变量、即时编译器编译后的代码等数据。在虚拟机启动时创建方法区。

![image-20231209144058404](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209144058404.png)

1.6中方法区包括：使用永久代来实现。常量池（包括StringTable字符串表），类的信息（field，method），类加载器。

1.8中方法区包括：使用元空间来实现，不占用jvm内存，启动到了本地内存中。

#### 方法区溢出问题

可以手动指定方法区大小`-XX:MaxMetaspaceSize=8m`

```java
package ustc.iat;

import jdk.internal.org.objectweb.asm.ClassWriter;
import jdk.internal.org.objectweb.asm.Opcodes;

/**
 * 演示元空间内存溢出:java.lang.OutOfMemoryError: Metaspace
 * -XX:MaxMetaspaceSize=8m
 */
public class Demo1_8 extends ClassLoader { // 需要继承类加载器，获得defineClass方法

	public static void main(String[] args) {
		int j = 0;
		try {
			Demo1_8 test = new Demo1_8();
			for (int i = 0; i < 100000; i++,j++) {
				//ClassWriter 作用是生产类的二进制字节码
				ClassWriter cw = new ClassWriter(0);
				//版本号，public，类名，包名，父类，接口
				cw.visit(Opcodes.V1_8, Opcodes.ACC_PUBLIC, "Class" + i, null, "java/lang/Object", null);
				//返回 byte[]
				byte[] code = cw.toByteArray();
				//执行类的加载
				test.defineClass("Class" + i, code, 0, code.length);
			}
		} finally {
			System.out.println(j);
		}
	}
	
}
```

- 1.8以前会导致**永久代**内存溢出（jvm内存）`java.lang.OutOfMemoryError: PermGen space`
- 1.8以后会导致**元空间**内存溢出（本地内存）`java.lang.OutOfMemoryError: Metaspace`

#### 运行时常量池

反编译字节码

```bash
javap -v .\Demo1_9.class
```

`-v`：显示详细的信息，包括类、方法、字段的修饰符、数据类型、字节码指令等。

- **常量池**：就是一张表，虚拟机指令根据这张常量表找到要执行的类名、方法名、参数类型、字面量等信息
- **运行时常量池**：常量池是 *.class 文件中的，当该类被加载，它的常量池信息就会放入运行时常量池，并把里面的符号地址变为真实地址

#### StringTable

```java
String s1 = "a"; // 常量池
String s2 = "b"; // 常量池
String s3 = "a" + "b"; // 常量池
String s4 = s1 + s2; // 堆
String s5 = "ab"; // 常量池
String s6 = s4.intern();
// 此处"ab"已经存在于常量池中，所以添加失败，s4还在堆中，s6位于常量池中
// 若将s5那行和s6那行交换，则"ab"原先不存在于字符串常量池中。在jdk1.8中，s4会放入常量池中，将常量池中的字符串返回；在jdk1.6中，s4会拷贝"ab"放入常量池中，s4还位于堆中，将常量池中的字符串返回

//问
system.out.println(s3 == s4);  // false
System.out.println(s3 == s5);  // true
System.out.println(s3 == s6);  // true
```

分析：

```java
package ustc.iat;

/**
 * @author mkzou
 * @create 2023-12-09 15:36
 */
public class Demo1_22 {

	// StringTable[ "a", "b", "ab"]

	// 常量池中的信息都会被加载到运行时常量池中
	// 这时a b ab都是常量池中的符号，还没变成java字符串对象，懒汉式加载
	// ldc #2 会把a变为"a"字符串对象
	// ldc #3 会把b变为"b"字符串对象
	// ldc #4 会把ab变为"ab"字符串对象

	public static void main(String[] args) {
		String s1 = "a"; // "懒惰加载","字符串常量池中是唯一的"
		String s2 = "b";
		String s3 = "ab";
		String s4 = s1 + s2; // s1和s2是变量，在运行的时候，他的结果可能修改，所以要在运行期间使用StringBuilder动态拼接
		// new StringBuilder().append("a").append("b").toString() ---> new String("ab")
		// 存入4号变量
		System.out.println(s3 == s4); // false 地址不同，是两个对象，必须在运行器件使用StringBuilder动态拼接
		String s5 = "a" + "b";
		System.out.println(s3 == s5); // true javac编译期间的优化，结果已经在编译期间确定为"ab"
	}

}
```

两则对比

第一组对比（jdk 1.8下）：

```java
package ustc.iat;

/**
 * @author mkzou
 * @create 2023-12-09 16:15
 */
public class Demo1_24 {

    public static void main(String[] args) {
       String x1 = "cd";
       String x2 = new String("c") + new String("d");
       // new String("a")和new String("b")结果放在堆中
       // x2 ：new String("ab") 结果也在堆中
       String x3 = x2.intern(); // 将字符串对象放入常量池中，如果有则不会放入，使用已有的常量池中的字符串返回；如果没有则放入常量池中，并返回已有的常量池中的字符串

       System.out.println("cd" == x1);  
       // true，显而易见
       System.out.println(x1 == x2);  
       // false，此处因为字符串常量池中已经有ab，所以，x2加入失败，x2还在堆空间中，所以不相等
       System.out.println(x1 == x3);  
       // true，此处x3是字符串常量池中的字符串，二者相等
    }

}
```

和

```java
package ustc.iat;

/**
 * @author mkzou
 * @create 2023-12-09 16:15
 */
public class Demo1_24 {

    public static void main(String[] args) {
       String x2 = new String("c") + new String("d");
       // new String("a")和new String("b")结果放在堆中
       // x2 ：new String("ab") 结果也在堆中
       String x3 = x2.intern(); // 将字符串对象放入常量池中，如果有则不会放入，使用已有的常量池中的字符串返回；如果没有则放入常量池中，并返回已有的常量池中的字符串
       String x1 = "cd";

       System.out.println("cd" == x1);  
       // true，显然
       System.out.println(x1 == x2);  
       // true，此处因为字符串常量池中没有ab，所以，x2加入成功，x2还在字符串常量池中，所以相等
       System.out.println(x1 == x3);  
       // true，此处x3是字符串常量池中的字符串，二者相等
    }

}
```

第二则对比：

- jdk 1.8中：

```java
String x2 = new String("c") + new String("d"); // 堆
String x1 = "cd"; // 常量池
String x3 = x2.intern(); // 因为“cd”已经存在于常量池中，所以不能加入失败，返回值x3是在字符串常量池中的

System.out.print1n(x1 == x2);  // jdk1.8下：false
System.out.print1n(x1 == x3);  // jdk1.8下：true
```

```java
String x2 = new String("c") + new String("d"); // 堆
String x3 = x2.intern(); // 因为“cd”还不在常量池中，所以加入成功，此时x2在常量池中，返回值x3是在字符串常量池中的
String x1 = "cd"; // 常量池中已经存在，则直接使用已经存在的

System.out.print1n(x1 == x2);  // jdk1.8下：true
System.out.print1n(x1 == x3);  // jdk1.8下：true
```

- jdk 1.6中：

```java
String x2 = new String("c") + new String("d"); // 堆
String x1 = "cd"; // 常量池
String x3 = x2.intern(); // 因为“cd”已经存在于常量池中，所以不能加入失败，返回值x3是在字符串常量池中的

System.out.print1n(x1 == x2);  // jdk1.6下：false
System.out.print1n(x1 == x3);  // jdk1.6下：true
```

```java
String x2 = new String("c") + new String("d"); // 堆
String x3 = x2.intern(); // 因为“cd”还不在常量池中，把x2拷贝一份放入字符串常量池，此时x2还在堆中，返回值x3是在字符串常量池中的
String x1 = "cd"; // 常量池中已经存在，则直接使用已经存在的

System.out.print1n(x1 == x2);  // jdk1.6下：false
System.out.print1n(x1 == x3);  // jdk1.6下：true
```

StringTable特性：

- 常量池中的字符串仅是符号，只有在被用到时才会转化为对象
- 利用串池的机制，来避免重复创建字符串对象
- 字符串变量拼接的原理是StringBuilder（1.8）
- 字符串常量拼接的原理是编译器优化
- 可以使用intern()方法，主动将串池中还没有的字符串对象放入串池中
  - 1.8将这个字符串对象尝试放入串池，如果有则并不会放入；如果没有，则放入串池，会把串池中的对象返回
  - 1.6将这个字符串对象尝试放入串池，如果有则并不会放入；如果没有，会把此对象复制一份，放入串池，把串池中的对象返回

#### StringTable位置

![image-20240121163459219](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240121163459219.png)

- jdk1.6：位于方法区中的常量池中，永久代的内存回收效率很低，需要在Full GC的时候才会触发垃圾回收，得等到老年代空间不足时才触发，时机较晚，导致StringTable回收效率较低，所以可能导致永久代内存不足。
- jdk1.8：位于堆中，只需要Minor GC（回收新生代）就会触发垃圾回收，减轻了StringTable堆内存的占用。

```java
public class Demo1_25 {

	/**
	 * 演示StringTable 位署
	 * 在jdk8下设置堆内存-Xmx10m  -XX:-UseGCOverheadLimit
	 * 在jdk6下设置永久代大小-XX:MaxPermSize=10m
	 */
	public static void main(String[] args) throws InterruptedException {
		List<String> list = new ArrayList<String>();
		int i = 0;
		try {
			for (int j = 8; i < 260000; j++) {
				list.add(String.valueOf(j).intern());
				i++;
			}
		} catch (Throwable e) {
			e.printStackTrace();
		} finally {
			System.out.println(i);
		}
	}

}
```

在jdk8下设置堆内存`-XX:-UseGCOverheadLimit`：是Java虚拟机的一项参数，用于禁用过度开销限制。如果98%的时间回收了2%的堆内存被回收，则会根据这个开关的开闭情况进行报错。这个开关关闭（此处关闭了这个开关），则报`OutOfMemoryError`错误；这个开关打开，则报`java.lang.OutOfMemoryError: GC overhead limit exceeded`错误。

jdk1.8下设置堆内存最大值为10m：`-Xmx10m`

jdk1.8下设置堆内存初始值为10m：`-Xms10m`

jdk1.6下设置永久代大小为10m：`-XX:MaxPermSize=10m`

#### StringTable垃圾回收机制

```java
/**
 * 演示 StringTable 垃圾回收
 * -Xmx10m -XX:+PrintStringTableStatistics -XX:+PrintGCDetails -verbose:gc
 * -Xmx10m : 设置堆内存大小为10m
 * -XX:+PrintStringTableStatistics : 打印字符串常量池的信息
 * -XX:+PrintGCDetails -verbose:gc : 打印垃圾回收详细信息
 */
public class Demo1_26 {

	public static void main(String[] args) {
		int i = 0;
		try {
			for(int j = 0; j < 10000; j++) { // j = 100, j = 10000
				String.valueOf(j).intern();
				i++;
			}
		}catch (Exception e) {
			e.printStackTrace();
		}finally {
			System.out.println(i);
		}
	}

}
```

输出结果：

```bash
[GC (Allocation Failure) [PSYoungGen: 2048K->504K(2560K)] 2048K->724K(9728K), 0.0015844 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
10000
Heap
 PSYoungGen      total 2560K, used 638K [0x00000000ffd00000, 0x0000000100000000, 0x0000000100000000)
  eden space 2048K, 6% used [0x00000000ffd00000,0x00000000ffd21950,0x00000000fff00000)
  from space 512K, 98% used [0x00000000fff00000,0x00000000fff7e010,0x00000000fff80000)
  to   space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
 ParOldGen       total 7168K, used 220K [0x00000000ff600000, 0x00000000ffd00000, 0x00000000ffd00000)
  object space 7168K, 3% used [0x00000000ff600000,0x00000000ff637030,0x00000000ffd00000)
 Metaspace       used 3230K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 350K, capacity 388K, committed 512K, reserved 1048576K
SymbolTable statistics:
Number of buckets       :     20011 =    160088 bytes, avg   8.000
Number of entries       :     13241 =    317784 bytes, avg  24.000
Number of literals      :     13241 =    566464 bytes, avg  42.781
Total footprint         :           =   1044336 bytes
Average bucket size     :     0.662
Variance of bucket size :     0.662
Std. dev. of bucket size:     0.814
Maximum bucket size     :         6
StringTable statistics:
Number of buckets       :     60013 =    480104 bytes, avg   8.000
Number of entries       :      3507 =     84168 bytes, avg  24.000
Number of literals      :      3507 =    241496 bytes, avg  68.861
Total footprint         :           =    805768 bytes
Average bucket size     :     0.058
Variance of bucket size :     0.057
Std. dev. of bucket size:     0.240
Maximum bucket size     :         2

Process finished with exit code 0
```

其中：

```bash
[GC (Allocation Failure) [PSYoungGen: 2048K->504K(2560K)] 2048K->724K(9728K), 0.0015844 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

发生了一次minor GC，因为分配内存不足。

#### StringTable性能调优

- 设置StringTable大小

  `-XX:StringTableSize=20000` : 设置桶的个数，最少为1009

  因为StringTable是由HashTable实现的。所以如果系统中字符串常量特别多的化，可以**适当增加HashTable桶的个数**，有一个更好的哈希分布，减少哈希碰撞，来减少字符串放入串池所需要的时间。

- 考虑是否需要将字符串对象入池，可以通过intern方法减少重复入池

  单词表重复读10遍
  
  ```java
  /**
   * -XX:StringTableSize=200000 -XX:+PrintStringTableStatistics
   * -Xms500m -Xmx500m -XX:+PrintStringTableStatistics -XX:StringTableSize=200000
  */
  public class Demo1_27 {
  
  	public static void main(String[] args) throws IOException {
  		List<String> address = new ArrayList<>();
  		System.in.read();
          // 每个单次重复读取十次
  		for (int i = 0; i < 10; i++) {
  			try (BufferedReader reader = new BufferedReader(new InputStreamReader(new FileInputStream("linux.words"), "utf-8"))) {
  				String line = null;
  				long start = System.nanoTime();
  				while (true) {
  					line = reader.readLine();
  					if (line == null) {
  						break;
  					}
  					address.add(line.intern()); // 直接address.add(line)会重复读取十次
  				}
  				System.out.println("cost:" + (System.nanoTime() - start) / 1000000);
  			}
  			System.in.read();
  		}
  	}
  
  }
  ```
  
  此处使用`line.intern()`会大大减少内存占用，因为存在大量重复字符串，将字符串放入常量池可以避免因为重复而导致的空间浪费。
  
  `-XX:StringTableSize=200000` 可以用来设置桶的大小，当桶数减少时，一个桶的链表长度也将减少，，则耗费时间将增加

### 直接内存

定义：

- 常见于 NIO 操作时，用于数据缓冲区

  NIO（New I/O）是Java中的一种面向通道和缓冲区的 I/O 操作方式。它在 Java 1.4 版本引入，为处理I/O操作提供了一种更为灵活和高效的机制。相对于传统的I/O操作，NIO 提供了更好的可扩展性和高并发性。

  NIO 的主要组件包括：

  1. **通道（Channel）：** 通道是与文件、套接字等 I/O 源/目标进行连接的对象。通道可以读取和写入数据，并支持异步操作。
  2. **缓冲区（Buffer）：** 缓冲区是 NIO 中用于数据存储和传输的对象。数据从通道读取到缓冲区，或者从缓冲区写入到通道。
  3. **选择器（Selector）：** 选择器是用于监控多个通道的对象，可以通过一个线程来管理多个通道的 I/O 事件。这使得一个线程可以有效地管理多个通道的 I/O 操作，提高了系统的性能。

  一则对比：

  - 传统IO（阻塞IO）：

    ![image-20231209184142800](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209184142800.png)

    赋值效率很低。系统缓存区中的数据还需要拷贝到java堆内存中。

    ![image-20231209184622647](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209184622647.png)

  - NIO：

    ![image-20231209184233409](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209184233409.png)

    赋值效率较高。java代码和系统都可以直接访问direct memory，二者共享direct memory，比传统IO少了一次赋值操作，速度得到了提升。

    ![image-20231209184704607](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209184704607.png)

- 分配回收成本较高，但读写性能高

- 不受 JVM 内存回收管理，可能会引发内存泄露，内存溢出的问题。


#### 直接内存溢出

```java
public class Demo1_28 {

	static int _100Mb = 1024 * 1024 * 100;

	public static void main(String[] args) {
		List<ByteBuffer> list = new ArrayList<>();
		int i = 0;
		try {
			while (true) {
				ByteBuffer byteBuffer = ByteBuffer.allocateDirect(_100Mb);
				list.add(byteBuffer);
				i++;
			}
		} finally {
			System.out.println(i);
		}
	}

}
```

`ByteBuffer.allocateDirect`直接内存分配

报错信息

```bash
Exception in thread "main" java.lang.OutOfMemoryError: Direct buffer memory
	at java.nio.Bits.reserveMemory(Bits.java:694)
	at java.nio.DirectByteBuffer.<init>(DirectByteBuffer.java:123)
	at java.nio.ByteBuffer.allocateDirect(ByteBuffer.java:311)
	at ustc.iat.Demo1_28.main(Demo1_28.java:20)
```

#### 直接内存分配和回收原理

```java
package ustc.iat;

import sun.misc.Unsafe;

import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.ByteBuffer;

public class Demo1_29 {

	public static int _1GB = 1024 * 1024 * 1024;

	public static void main(String[] args) throws IOException, NoSuchFieldException, IllegalAccessException {
		// method();
		method1();
	}

	// 演示 直接内存 是被 unsafe 创建与回收
	private static void method1() throws IOException, NoSuchFieldException, IllegalAccessException {

		Field field = Unsafe.class.getDeclaredField("theUnsafe");
		field.setAccessible(true);
		Unsafe unsafe = (Unsafe)field.get(Unsafe.class);

		long base = unsafe.allocateMemory(_1GB);
		unsafe.setMemory(base,_1GB, (byte)0);
		System.out.println("分配完毕");
		System.in.read();
		System.out.println("开始释放");

		unsafe.freeMemory(base);
		System.in.read();
	}

	// 演示 直接内存被 释放，在这里调用了unsafe来进行直接内存的分配和回收
	private static void method() throws IOException {
		ByteBuffer byteBuffer = ByteBuffer.allocateDirect(_1GB);
		System.out.println("分配完毕");
		System.in.read();
		System.out.println("开始释放");
		byteBuffer = null;
		System.gc(); // 手动 gc
		System.in.read();
	}

}
```

直接内存的回收不是通过 jvm的垃圾回收`System.gc();`来释放的，而是通过`unsafe.freeMemory` 来手动释放。

剖析底层源码：

##### 直接内存的分配

当我们执行如下代码时：

```java
ByteBuffer byteBuffer = ByteBuffer.allocateDirect(_1GB);
```

调用了：

```java
public static ByteBuffer allocateDirect(int capacity) {
    return new DirectByteBuffer(capacity);
}
```

间接调用了：

```java
static final Unsafe UNSAFE = Unsafe.getUnsafe();

DirectByteBuffer(int cap) {

    super(-1, 0, cap, cap, null);
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    Bits.reserveMemory(size, cap);

    long base = 0;
    try {
        base = UNSAFE.allocateMemory(size);
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    UNSAFE.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
    
}
```

在此处有：

```java
base = UNSAFE.allocateMemory(size);
UNSAFE.setMemory(base, size, (byte) 0);
```

使用`Unsafe`类分配内存。

##### 直接内存的回收

使用虚引用对象`cleaner`进行垃圾回收

```java
cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
```

构造方法中的`new Deallocator(base, size, cap)`即为`thunk`。使用虚引用对象当`this`被垃圾回收时，则触发`cleaner.clean()`

```java
public void clean() {
    if (!remove(this))
        return;
    try {
        thunk.run();
    } catch (final Throwable x) {
        AccessController.doPrivileged(new PrivilegedAction<>() {
            public Void run() {
                if (System.err != null)
                    new Error("Cleaner terminated abnormally", x)
                    .printStackTrace();
                System.exit(1);
                return null;
            }});
    }
}
```

`cleaner.clean()`中`thunk.run()`执行（此处`new Deallocator(base, size, cap)`即为`thunk`）

```java
private static class Deallocator implements Runnable
{

    private long address;
    private long size;
    private int capacity;

    private Deallocator(long address, long size, int capacity) {
        assert (address != 0);
        this.address = address;
        this.size = size;
        this.capacity = capacity;
    }

    public void run() {
        if (address == 0) {
            // Paranoia
            return;
        }
        UNSAFE.freeMemory(address);
        address = 0;
        Bits.unreserveMemory(size, capacity);
    }
}
```

`thunk.run()`中执行`UNSAFE.freeMemory(address);`完成直接内存的内存释放。

#### 禁用显示内存回收

```java
package ustc.iat;

import sun.misc.Unsafe;

import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.ByteBuffer;

public class Demo1_30 {

	public static int _1GB = 1024 * 1024 * 1024;

	/**
	 * -XX:+DisableExplicitGC 禁止显示的垃圾回收，即禁用System.gc();
	 */
	private static void method() throws IOException, NoSuchFieldException, IllegalAccessException {
		Field field = Unsafe.class.getDeclaredField("theUnsafe");
		field.setAccessible(true);
		Unsafe unsafe = (Unsafe)field.get(Unsafe.class);

		long base = unsafe.allocateMemory(_1GB);
		unsafe.setMemory(base,_1GB, (byte)0);
		System.out.println("分配完毕");
		System.in.read();
		System.out.println("开始释放");

        // 显示的垃圾回收，Full gc，比较影响性能，不光回收新生代，也会回收老年代
		unsafe.freeMemory(base);
        // System.gc();
		System.in.read();
	}

	public static void main(String[] args) {
		try {
			method();
		} catch (IOException e) {
			throw new RuntimeException(e);
		} catch (NoSuchFieldException e) {
			throw new RuntimeException(e);
		} catch (IllegalAccessException e) {
			throw new RuntimeException(e);
		}
	}

}
```

使用`-XX:+DisableExplicitGC`禁止显示的垃圾回收，即禁用`System.gc();`，这是因为显示的垃圾回收`System.gc();`是`Full gc`，比较影响性能，不光回收新生代，也会回收老年代

## 垃圾回收

#### 引用计数法

当一个对象被引用时，就当引用对象的值+1，当值为 0 时，就表示该对象不被引用，可以被垃圾收集器回收。引用计数法有一个弊端：

![image-20231209205439168](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209205439168.png)

当出现循环引用时，两个对象的计数都为1，导致两个对象都无法被释放。（Python早期版本使用引用计数法）

#### 可达性分析方法

JVM 中的垃圾回收器通过可达性分析来探索所有存活的对象。

- 先确定GC Root对象（一定不能被垃圾回收的对象）

- 先扫描所有对象，判断当前对象是否被GC Root对象直接或间接引用。如果是，则不能垃圾回收；如果不是，则可以进行垃圾回收。

GC Root 的对象可以是：

- 虚拟机栈（栈帧中的本地变量表）中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中 JNI（即一般说的Native方法）引用的对象

使用MAT（由eclipse提供）来进行查看GC Root

#### 五种引用

![image-20231209213221349](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209213221349.png)

##### 强引用

- 只有所有 GC Roots 对象都不通过强引用引用该对象，该对象才能被垃圾回收

  当B对象和C对象都不直接或间接引用A1对象时，A1对象才能被回收

##### 软引用

- 仅有软引用引用该对象、没有强引用直接或间接引用该对象时，**在垃圾回收后，内存仍不足时会再次触发垃圾回收，会回收软引用对象**。当被回收时，软引用自身会进入引用队列（可以使用引用队列，也可以不使用引用队列）

  **单独只使用软引用：**

  ```java
  /**
   * 演示 软引用
   * 虚拟机设置参数 -Xmx20m -XX:+PrintGCDetails -verbose:gc
   */
  public class Demo2_2 {
  
  	public static int _4MB = 4 * 1024 * 1024;
  
  	public static void main(String[] args) throws IOException {
  		method2();
  	}
  
  	// 设置堆内存为20m，-Xmx20m，则会堆内存溢出
      // 设置-XX:+PrintGCDetails -verbose:gc 打印垃圾回收详情信息
  	public static void method1() throws IOException {
  		ArrayList<byte[]> list = new ArrayList<>();
  
  		for(int i = 0; i < 5; i++) {
  			list.add(new byte[_4MB]);
  		}
  		System.in.read();
  	}
  
  	// 演示 软引用
  	public static void method2() throws IOException {
  		// list -> SoftReference -> byte[]
  		// 当垃圾回收后，内存仍不足时会再次触发垃圾回收，会回收软引用对象
  		ArrayList<SoftReference<byte[]>> list = new ArrayList<>();
  		for(int i = 0; i < 5; i++) {
  			SoftReference<byte[]> ref = new SoftReference<>(new byte[_4MB]);
  			System.out.println(ref.get());
  			list.add(ref);
  			System.out.println(list.size());
  		}
  		System.out.println("循环结束：" + list.size());
  		for(SoftReference<byte[]> ref : list) {
  			System.out.println(ref.get());
  		}
  	}
  }
  ```

  `method1()`会导致堆内存溢出，因为此处数组中五个元素都是强引用，占用的空间超出了堆内存大小。考虑使用软引用解决这一问题，`list -强引用->  SoftReference -软引用-> byte[]` ，当执行`Full GC`仍然内存不足时，会释放软引用所引用的对象。

  **配合引用队列一起使用：**

  ```java
  // 引用队列
  public static ReferenceQueue<byte[]> queue = new ReferenceQueue<>();
  
  // 演示 软引用
  public static void method2() throws IOException {
      // list -> SoftReference -> byte[]
      // 当垃圾回收后，内存仍不足时会再次触发垃圾回收，会回收软引用对象
      ArrayList<SoftReference<byte[]>> list = new ArrayList<>();
      for(int i = 0; i < 5; i++) {
          // 关联了软引用对象和引用队列，当软引用所引用的byte[]数组被回收时，软引用自身会加入到queue中去
          SoftReference<byte[]> ref = new SoftReference<>(new byte[_4MB], queue);
          System.out.println(ref.get());
          list.add(ref);
          System.out.println(list.size());
      }
      System.out.println("循环结束：" + list.size());
      // 将null对象移除
      Reference<? extends byte[]> poll = queue.poll();
      while (poll != null) {
          list.remove(poll);
          poll = queue.poll();
      }
      // 打印结果
      for(SoftReference<byte[]> ref : list) {
          System.out.println(ref.get());
      }
  }
  ```

##### 弱引用

- 仅有弱引用引用该对象、没有强引用直接或间接引用该对象时，**在垃圾回收时，无论内存是否充足，都会回收弱引用对象**。当被回收时，弱引用自身会进入引用队列（可以使用引用队列，也可以不适用引用队列）

  例子：

  ```java
  /**
   * 弱引用
   * -Xmx20m -XX:+PrintGCDetails -verbose:gc
   */
  public class Demo2_3 {
  
  	public static void main(String[] args) {
  		// method1();
  		method2();
  	}
  
  	public static int _4MB = 4 * 1024 *1024;
  
  	// 演示 弱引用
  	public static void method1() {
  		List<WeakReference<byte[]>> list = new ArrayList<>();
  		for(int i = 0; i < 10; i++) {
  			WeakReference<byte[]> weakReference = new WeakReference<>(new byte[_4MB]);
  			list.add(weakReference);
  
  			for(WeakReference<byte[]> wake : list) {
  				System.out.print(wake.get() + ",");
  			}
  			System.out.println();
  		}
  	}
  
  	// 引用队列
  	public static ReferenceQueue<byte[]> queue = new ReferenceQueue<>();
  
  	// 演示 弱引用搭配 引用队列
  	public static void method2() {
  		List<WeakReference<byte[]>> list = new ArrayList<>();
  
  		for(int i = 0; i < 9; i++) {
  			WeakReference<byte[]> weakReference = new WeakReference<>(new byte[_4MB], queue);
  			list.add(weakReference);
  		}
  		System.out.println("===========================================");
  
  		Reference<? extends byte[]> poll = queue.poll();
  		while (poll != null) {
  			list.remove(poll);
  			poll = queue.poll();
  		}
  		// 打印结果
  		for(WeakReference<byte[]> wake : list) {
  			System.out.print(wake.get() + ",");
  		}
  	}
  
  }
  ```

  与软引用类似，不过无论内存空间是否充足，弱引用搜索引用的对象都会被回收。

##### 虚引用

- （必须配合引用队列一起使用）当虚引用引用的对象被垃圾回收时，则会放入引用队列，从而由一个线程来调用虚引用对象的`clean()`方法，进而调用`UNSAFE.freeMemory(address);`来完成直接内存的垃圾回收

##### 终结器引用

- （必须配合引用队列一起使用）在垃圾回收时，终结器引用入队（被引用对象暂时没有被回收），再由`Finalizer`线程通过终结器引用找到被引用对象并调用它的`finalize`方法，第二次垃圾回收时才能回收被引用对象。

  缺点：工作效率很低，`Finalizer`线程优先级很低，这会导致待销毁对象的`finalize`方法迟迟得不到调用，对象占用的内存迟迟得不到释放。

#### 可达性分析方法---标记清除

步骤：

- 扫描对象是否被GC Root引用。如果某对象没有GC Root引用，则可以回收。
- 对于要回收的垃圾对象所占用的空间释放，即将起始地址和终点地址记录下来，放在空闲地址列表中。（无需将其全部置0）

**优点**：垃圾回收速度快

**缺点**：容易产生内存碎片，空间利用率降低，内存碎片无法存储大对象

![image-20231209224905436](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209224905436.png)

#### 可达性分析方法---标记整理

**优点**：没有内存碎片

**缺点**：垃圾回收速度慢

![image-20231209230017375](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209230017375.png)

#### 可达性分析方法---复制

**优点**：没有内存碎片

**缺点**：垃圾回收速度慢，占用双倍的内存空间

**步骤**：

- 将被GC Root所引用的对象复制到另一个区域

  ![image-20231209230157893](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209230157893.png)

- 清空From

  ![image-20231209230222300](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209230222300.png)

- 交换From和To

  ![image-20231209230240921](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209230240921.png)

#### 分代垃圾回收

**新生代**：

包括：

- 伊甸园
- 幸存区from
- 幸存区to

规则：

- 对象诞生在伊甸园中。
- 当新生代空间不足时，触发`minor gc` ，使用可达性分析方法中的复制，将伊甸园和from区存活的对象复制到幸存区to中，同时让存活的对象年龄+1，将伊甸园和幸存区from中标记为垃圾的对象释放，然后交换幸存区from和幸存区to。
- `minor gc` 会引发 `stop the world`，暂停其他用户线程，等垃圾回收结束后，恢复用户线程运行
- 当幸存区对象的寿命超过阈值15（4bit）时，说明该对象使用频率较高，因此会晋升到老年代。
- 对于存储大对象（老年代够用，新生代不够），则直接晋升老年代。

**老年代**：

当老年代空间不足时，会先触发`minor gc`，如果空间仍然不足，那么就触发`full fc` ，`stop the world`停止的时间更长！如果执行`full gc`后，老年代空间还是不足，则会触发`out of memory`错误

![image-20231209230851820](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231209230851820.png)

#### JVM相关参数

| 含义               | 参数                                                         |
| ------------------ | ------------------------------------------------------------ |
| 堆初始大小         | -Xms                                                         |
| 堆最大大小         | -Xmx 或 -XX:MaxHeapSize=size                                 |
| 新生代大小         | -Xmn 或 (-XX:NewSize(初始大小)=size + -XX:MaxNewSize(最大大小)=size ) |
| 幸存区比例（动态） | -XX:InitialSurvivorRatio=ratio(初始幸存区比例) 和 -XX:+UseAdaptiveSizePolicy（开启动态调整幸存区比例） |
| 幸存区比例（静态） | -XX:SurvivorRatio=ratio(默认8，例子：10m新生代，8m伊甸园，1m幸存区from，1m幸存区to) |
| 晋升阈值           | -XX:MaxTenuringThreshold=threshold（默认15）                 |
| 晋升详情           | -XX:+PrintTenuringDistribution                               |
| GC详情             | -XX:+PrintGCDetails -verbose:gc                              |
| FullGC 前 MinorGC  | -XX:+ScavengeBeforeFullGC（在full gc前执行一次minor gc，默认打开） |

#### gc分析

```java
public class Demo2_4 {

    private static final int _512KB = 512 * 1024;

    private static final int _1MB = 1024 * 1024;

    private static final int _6MB = 6 * 1024 * 1024;

    private static final int _7MB = 7 * 1024 * 1024;

    private static final int _8MB = 8 * 1024 * 1024;

    // -Xms20m -Xmx20m -Xmn10m -XX:+UseSerialGC -XX:+PrintGCDetails -verbose:gc
    // 设置堆内存初始大小20m，最大大小20m，新生代大小10m，使用串行垃圾回收器，打印垃圾回收信息
    public static void main(String[] args) {
       List<byte[]> list = new ArrayList<>();
       list.add(new byte[_7MB]);
       list.add(new byte[_512KB]);
       list.add(new byte[_512KB]);
       list.add(new byte[_6MB]);
       list.add(new byte[_512KB]);
       list.add(new byte[_512KB]);
       list.add(new byte[_512KB]);
       list.add(new byte[_512KB]);
       list.add(new byte[_512KB]);
       list.add(new byte[_512KB]);
    }

}
```

通过上面的代码，给 list 分配内存，来观察 新生代和老年代的情况，什么时候触发 minor gc，什么时候触发 full gc 等情况，使用前需要设置 jvm 参数`-Xms20m -Xmx20m -Xmn10m -XX:+UseSerialGC -XX:+PrintGCDetails -verbose:gc`。结果如下：

```bash
Connected to the target VM, address: '127.0.0.1:3988', transport: 'socket'
[GC (Allocation Failure) [DefNew: 2273K->632K(9216K), 0.0013931 secs] 2273K->632K(19456K), 0.0014871 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [DefNew: 8476K->512K(9216K), 0.0031956 secs] 8476K->8308K(19456K), 0.0032196 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [DefNew: 8354K->8354K(9216K), 0.0000095 secs][Tenured: 7796K->8308K(10240K), 0.0031168 secs] 16151K->15988K(19456K), [Metaspace: 3013K->3013K(1056768K)], 0.0031595 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 def new generation   total 9216K, used 8520K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  97% used [0x00000000fec00000, 0x00000000ff3d20b8, 0x00000000ff400000)
  from space 1024K,  50% used [0x00000000ff400000, 0x00000000ff480010, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 tenured generation   total 10240K, used 9844K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  96% used [0x00000000ff600000, 0x00000000fff9d380, 0x00000000fff9d400, 0x0000000100000000)
 Metaspace       used 3019K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 321K, capacity 386K, committed 512K, reserved 1048576K
Disconnected from the target VM, address: '127.0.0.1:3988', transport: 'socket'

Process finished with exit code 0
```

上述代码，进行了三次`minor gc`，最终新生代伊甸园区`97%`占用，幸存区from`50%`占用，幸存区to`0%`占用，老年代`96%`占用。如果在上述代码再增加一行

```java
list.add(new byte[_512KB]);
```

则会导致`out of memory`错误。

对于存储大对象（老年代够用，新生代不够），则直接晋升老年代。

```java
public class Demo2_4 {

    private static final int _512KB = 512 * 1024;

	private static final int _1MB = 1024 * 1024;

	private static final int _6MB = 6 * 1024 * 1024;

	private static final int _7MB = 7 * 1024 * 1024;

	private static final int _8MB = 8 * 1024 * 1024;

	private static final int _9MB = 9 * 1024 * 1024;

	// -Xms20m -Xmx20m -Xmn10m -XX:+UseSerialGC -XX:+PrintGCDetails -verbose:gc
	// 设置堆内存初始大小20m，最大大小20m，新生代大小10m，使用串行垃圾回收器，打印垃圾回收信息
	public static void main(String[] args) {
		List<byte[]> list = new ArrayList<>();
		list.add(new byte[_9MB]);
	}

}
```

运行结果：

```jaav
Heap
 def new generation   total 9216K, used 2021K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  24% used [0x00000000fec00000, 0x00000000fedf9788, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 tenured generation   total 10240K, used 9216K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  90% used [0x00000000ff600000, 0x00000000fff00010, 0x00000000fff00200, 0x0000000100000000)
 Metaspace       used 3194K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 344K, capacity 388K, committed 512K, reserved 1048576K
```

上述代码伊甸园默认为新生代的`80%`，即`8m`，但是存入了一个大对象，这个大对象超过了伊甸园大小、幸存区from大小，但是没超过老年代大小，所有直接晋升老年代，未发生垃圾回收，老年代空间占用达到`90%`。

当申请两个8MB的内存空间时，则报堆内存溢出错误信息。

```java
package ustc.iat;

import java.util.ArrayList;
import java.util.List;

/**
 * @author mkzou
 * @create 2023-12-10 14:29
 */
public class Demo2_4 {

	private static final int _512KB = 512 * 1024;

	private static final int _1MB = 1024 * 1024;

	private static final int _4MB = 4 * 1024 * 1024;

	private static final int _7MB = 7 * 1024 * 1024;

	private static final int _8MB = 8 * 1024 * 1024;

	private static final int _9MB = 9 * 1024 * 1024;

	// -Xms20m -Xmx20m -Xmn10m -XX:+UseSerialGC -XX:+PrintGCDetails -verbose:gc
	// 设置堆内存初始大小20m，最大大小20m，新生代大小10m，使用串行垃圾回收器，打印垃圾回收信息
	public static void main(String[] args) throws InterruptedException {

		List<byte[]> list = new ArrayList<>();
		list.add(new byte[_8MB]);
		list.add(new byte[_8MB]);

		System.out.println("sleep...");
		Thread.sleep(0000L);
	}

}
```

运行结果：

```ba
[GC (Allocation Failure) [DefNew: 1857K->624K(9216K), 0.0014147 secs][Tenured: 8192K->8815K(10240K), 0.0023781 secs] 10049K->8815K(19456K), [Metaspace: 3220K->3220K(1056768K)], 0.0039179 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure) [Tenured: 8815K->8797K(10240K), 0.0018954 secs] 8815K->8797K(19456K), [Metaspace: 3220K->3220K(1056768K)], 0.0019198 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 def new generation   total 9216K, used 246K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,   3% used [0x00000000fec00000, 0x00000000fec3d890, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 tenured generation   total 10240K, used 8797K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  85% used [0x00000000ff600000, 0x00000000ffe977b8, 0x00000000ffe97800, 0x0000000100000000)
 Metaspace       used 3252K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 353K, capacity 388K, committed 512K, reserved 1048576K
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at ustc.iat.Demo2_4.main(Demo2_4.java:30)
```

如果`Full gc`也没有作用，则报错`java.lang.OutOfMemoryError: Java heap space`。

**注**：线程内因为out of memory而结束不会导致进程的结束：

```java
private static final int _9MB = 9 * 1024 * 1024;

// -Xms20m -Xmx20m -Xmn10m -XX:+UseSerialGC -XX:+PrintGCDetails -verbose:gc
// 设置堆内存初始大小20m，最大大小20m，新生代大小10m，使用串行垃圾回收器，打印垃圾回收信息
public static void main(String[] args) throws InterruptedException {
    new Thread(() -> {
        List<byte[]> list = new ArrayList<>();
        list.add(new byte[_9MB]);
        list.add(new byte[_9MB]);
    }).start();

    System.out.println("sleep...");
    Thread.sleep(10000L);
}
```

结果：

```java
sleep...
[GC (Allocation Failure) [DefNew: 4366K->829K(9216K), 0.0017751 secs][Tenured: 9216K->10043K(10240K), 0.0021514 secs] 13582K->10043K(19456K), [Metaspace: 4150K->4150K(1056768K)], 0.0039888 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure) [Tenured: 10043K->9987K(10240K), 0.0015496 secs] 10043K->9987K(19456K), [Metaspace: 4150K->4150K(1056768K)], 0.0015719 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Exception in thread "Thread-0" java.lang.OutOfMemoryError: Java heap space
	at ustc.iat.Demo2_4.lambda$main$0(Demo2_4.java:30)
	at ustc.iat.Demo2_4$$Lambda$1/1096979270.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:748)
Heap
 def new generation   total 9216K, used 1376K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  16% used [0x00000000fec00000, 0x00000000fed58248, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 tenured generation   total 10240K, used 9987K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  97% used [0x00000000ff600000, 0x00000000fffc0f60, 0x00000000fffc1000, 0x0000000100000000)
 Metaspace       used 4670K, capacity 4740K, committed 4992K, reserved 1056768K
  class space    used 517K, capacity 560K, committed 640K, reserved 1048576K
```

#### 垃圾回收器

串行：

- 单线程垃圾回收器。垃圾回收时，其他线程都暂停。
- 适用于堆内存较小，适合个人电脑

吞吐量优先

- 多线程的
- 适用于堆内存较大，需要多核cpu来支持
- 单位时间内，`stop the world`时间最短（0.2 + 0.2 = 0.4）

响应时间优先

- 多线程的
- 适用于堆内存较大，需要多核cpu来支持
- 尽可能让单次的`stop the world`时间最短（0.1 + 0.1 + 0.1 + 0.1 + 0.1 = 0.5）

#### 串行垃圾回收器

开启语句：`-XX:+UseSerialGC`，串行垃圾回收器会对新生代和老年代都进行垃圾回收。对于新生代（serial），使用可达性分析方法（复制）进行垃圾回收；对于老年代（serialOld），使用可达性分析方法（标记整理）

![image-20231210152332217](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210152332217.png)

**安全点**：让其他线程都在这个点停下来，以免垃圾回收时移动对象地址，使得其他线程找不到被移动的对象。
因为是串行的，所以只有一个垃圾回收线程。且在该线程执行回收工作时，其他线程进入阻塞状态，当垃圾回收执行完毕后，再唤醒其他线程。

#### 吞吐量优先垃圾回收器（jdk 1.8默认使用）

目标：单位时间内，`stop thw world`时间最短。

开启语句：`-XX:+UseParallelGC ~ -XX:+UsePrallerOldGC`，只要开启其中一个，另一个就会联代开启。

除此以外，还有几个参数：

```java
-XX:+UseAdaptiveSizePolicy // 开启动态调整幸存区比例
-XX:GCTimeRatio=ratio // 1 / (1+radio)，默认是99，那么这个值是0.01，即100min中，只能有1min用于内存回收，虚拟机会以此为目标，调整垃圾回收策略，努力达到这个目标。99有点困难，所以一般取 19，当ratio增大时，则要求堆内存增大
-XX:MaxGCPauseMillis=ms // 最大暂停毫秒数默认值200ms 如果堆内存增大，则每次垃圾回收时间会增长，二者此消彼长
-XX:ParallelGCThreads=n // 并行线程的数目
```

需要设置合理的堆内存

```markdown
- 当堆内存增加时，GCTimeRatio（垃圾回收占总时间的比重）将会减小，但是MaxGCPauseMillis（单次垃圾回收的最大时间）将会增加
- 当堆内存减少时，GCTimeRatio（垃圾回收占总时间的比重）将会增加，但是MaxGCPauseMillis（单次垃圾回收的最大时间）将会减小
```

对于新生代（serial），使用可达性分析方法（复制）进行垃圾回收；对于老年代（serialOld），使用可达性分析方法（标记整理）

**安全点**：让其他线程都在这个点停下来，以免垃圾回收时移动对象地址，使得其他线程找不到被移动的对象。
然后所有cpu都开始执行垃圾回收线程。当垃圾回收执行完毕后，再一起运行，在垃圾回收过程中，cpu使用率会接近100%。

![image-20231210153856336](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210153856336.png)

#### 响应时间优先垃圾回收器（CMS垃圾回收器）（标记清除+并发）

在垃圾回收的`stop the word` 时间减少（只有在初始标记和重新标记的时候才会 `STW`），垃圾回收和用户线程并发执行；但是在另一些阶段，还是需要`stop the word`

目标：尽可能让单次的`stop thw world`时间最短

开启语句：`-XX:+UseConcMarkSweepGC ~ -XX:+UseParNewGC ~ SerialOld`。在老年代上，使用`-XX:+UseConcMarkSweepGC`可达性分析方法（标记清除）；在新生代上，使用`-XX:+UseParNewGC`可达性分析方法（复制），如果`-XX:+UseConcMarkSweepGC`并发失败，则会退化为串行`SerialOld`（基于标记整理）

其他参数：

```java
-XX:ParallelGCThreads=n ~ -XX:ConcGCThreads=threads
-XX:CMSInitiatingOccupancyFraction=percent
-XX:+CMSScavengeBeforeRemark
```

- `-XX:ParallelGCThreads=n`是并行线程数目（一般是cpu核数），`-XX:ConcGCThreads=threads`是并发线程数目，一般并发线程数目取并行线程数目的四分之一。这可能会导致计算同样的任务，速度降低

- `-XX:CMSInitiatingOccupancyFraction=percent`是当老年代内存占比达到`percent`时，执行一次垃圾回收，预留一些空间给**浮动垃圾**，默认值`65%`。当这个值设置地越小，则垃圾回收更早进行
- `-XX:+CMSScavengeBeforeRemark`是一个开关。若开启，则会在重新标记之前，对新生代进行一次垃圾回收`UseParNewGC`，减轻重新标记的工作量。

![image-20231210160445191](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210160445191.png)

步骤：

- 初始标记：用户线程停止运行，标记 GC Roots 对象。这个阶段会`stop the word`，但是速度很快。
- 并发标记：用户线程恢复运行，进行GC Roots Tracing（被GC Root直接、间接引用的对象）的过程，找出存活对象且用户线程可并发执行。
- 重新标记：用户线程停止运行，为了修正并发标记期间因用户程序继续运行而导致标记产生变动的那一部分对象的标记记录。仍然存在`stop the word`问题
- 并发清除：用户线程恢复运行，对标记的对象进行清除回收，清除的过程中，可能任然会有新的垃圾产生，这些垃圾就叫**浮动垃圾**，需要预留一些空间给浮动垃圾。如果当用户需要存入一个很大的对象时，新生代放不下去，老年代由于浮动垃圾过多，就会退化为`SerialOld`，将老年代垃圾进行可达性分析方法（标记整理），当然这也是很耗费时间的！

存在的问题：

- 因为老年代使用`-XX:+UseConcMarkSweepGC`可达性分析方法（标记清除），所以这会使得老年代存在大量的内存碎片，产生并发失败，从而退化为`SerialOld`，在老年代上使用可达性分析方法（标记整理），这时垃圾回收时间会突然增大，影响用户体验。

#### G1垃圾回收器

jdk9默认使用G1垃圾回收器，取代了cms垃圾回收器

**适用场景：**

- 同时注重吞吐量和低延迟（响应时间），默认暂停目标200ms
- 超大堆内存（内存大的），会将堆内存划分为多个大小相等的区域region
- 整体上是标记-整理算法，两个区域之间是复制算法

**相关JVM参数：**

- `-XX:+UseG1GC`：手动开启G1垃圾回收器

- `-XX:G1HeapRegionsize=size`：超大堆内存（内存大的），会将堆内存划分为多个大小相等的区域，这个区域大小即为`size`

- `-XX:MaxGCPauseMillis=time`：指定最长的停顿时间

**垃圾回收阶段：**

![image-20231210164826410](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210164826410.png)

- Young Collection：对新生代垃圾收集。

  ![image-20231210165033424](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210165033424.png)

  E：伊甸园。当E逐渐占满（E这一块逐渐占满），会触发新生代垃圾回收，会有`stop the word`，时间较短

  ![image-20231210165159155](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210165159155.png)

  垃圾回收之后，幸存的对象会使用复制算法放入幸存区E

  ![image-20231210165313287](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210165313287.png)

  随着时间的推移，幸存区中的一些对象可能会晋升到老年代O

- Young Collection + Concurrent Mark（新生代的垃圾回收和并发标记阶段）：如果老年代内存到达一定的阈值，新生代垃圾收集同时会执行一些并发的标记。

  初始标记：找到GC Root根对象。（新生代GC就会发生）

  并发标记：从GC Root根对象出发，找到那些被根对象直接、间接引用的对象。（当老年代占用的比例达到一定的阈值时，默认：45%，可以通过`XX:InitiatingHeapOccupancyPercent=percent`进行设置）

  ![image-20231210170148323](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210170148323.png)

- Mixed Collection：会对新生代 + 老年代 + 幸存区等进行混合收集，然后收集结束，会重新进入新生代收集。

  

  ![image-20231210170233953](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210170233953.png)

  会对 E S O 进行全面的回收

  - 最终标记会 STW

  - 拷贝存活会 STW

  - **Q：**为什么有的老年代被拷贝了，有的没拷贝？

    **A：**因为指定了最大停顿时间`-XX:MaxGCPauseMillis=time`，如果对所有老年代都进行回收，耗时可能过高。为了保证时间不超过设定的停顿时间，会回收最有价值的老年代（回收后，能够得到更多内存）

#### Minor GC

Serial GC：新生代内存不足发生的垃圾收集- minor gc

Parallel GC：新生代内存不足发生的垃圾收集- minor gc

CMS：新生代内存不足发生的垃圾收集- minor gc

G1：新生代内存不足发生的垃圾收集- minor gc

#### Full GC

Serial GC：老年代内存不足发生的垃圾收集- full gc

Parallel GC：老年代内存不足发生的垃圾收集- full gc

CMS：老年代内存不足

- 如果垃圾回收速度大于垃圾产生速度，并发收集阶段，不会触发Full GC

- 如果垃圾回收速度小于垃圾产生速度，并发收集失败，退化为串行收集，会触发Full GC。

G1：老年代内存不足

- 老年代所占内存超过阈值（默认45%）

  - 如果垃圾回收速度大于垃圾产生速度，并发收集阶段，不会触发Full GC

  - 如果垃圾回收速度小于垃圾产生速度，并发收集失败，会触发Full GC（多线程）

#### Young Collection 跨代引用

新生代回收的跨代引用（老年代引用新生代）问题

![image-20231210183913057](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210183913057.png)

- Remembered Set存在于E中，用于保存新生代对象对应的脏卡

  当对E进行垃圾回收时，可以先通过Remembered Set知晓其对应哪些脏卡，然后在脏卡区域遍历GC Root，减少了GC Root的遍历时间

- 脏卡：O 被划分为多个区域（一个区域512K），如果该区域引用了新生代对象，则该区域被称为脏卡
- 在引用变更时通过post-write barried（写屏障，在每次对象引用发生变更时，都要去更新脏卡），上述操作是异步的，不会立刻更新，将更新的指令其加入dirty card queue，由一个线程异步地完成脏卡的更新操作

#### Remark重新标注阶段

使用`pre-write barrier`（写屏障）和`satb_mark_queue`（队列）

在垃圾回收时，收集器处理对象的过程中

- 黑色：已被处理，需要保留的
- 灰色：正在处理中的
- 白色：还未处理的

例子：

- 初始

![image-20231210190444699](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210190444699.png)

- 开始处理B，此时，用户线程将B对象的C属性删除，则C还是未处理的

![image-20231210190541611](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210190541611.png)

- 但是此时，用户又将C对象作为A对象的一个属性

![image-20231210190711275](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210190711275.png)

- 此时C还是未处理的，应该会被回收，但是C对象是A对象的一个属性，不应当删除，此处产生错误。
- 改正方法：当对象的引用发生改变时，jvm会给其加入一个**写屏障（pre-write barrier）**，将对象C加入一个队列，并且**标记为还未处理完**，等到重新标记阶段，会先`stop the word`，然后重新标记线程会把队列的每个对象取出，再做检查，此时发现一个未处理完的C对象且有一个强引用引用C对象，此时就会将C对象标记为已处理（需要保留），此时C对象就不会被垃圾回收

![image-20231210190134473](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210190134473.png)

重新标记阶段：

​							![image-20231210191321700](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20231210191321700.png)

#### 字符串去重（jdk8u20 优化）

开启方式：`-XX:+UseStringDeduplication`

过程:

```java
String s1 = new String("hello");
String s2 = new String("hello");
```

- 将所有新分配的字符串（底层是 char[] ）放入一个队列
- 当新生代回收时，G1 并发检查是否有重复的字符串
- 如果字符串的值一样，就让他们引用同一个字符串对象
- 字符串去重与 String.intern() 的区别
  - String.intern() 关注的是字符串对象String
  - 字符串去重关注的是 char[]

优点

- 节省了大量内存

缺点

- 新生代回收时间略微增加，导致略微多占用 CPU

#### 并发标记类卸载（jdk8u20 优化）

在并发标记阶段结束以后，就能知道哪些类不再被使用。如果一个类加载器的所有类都不在使用，则卸载它所加载的所有类

开启方法：`-XX:+ClassUnloadingwithConcurrentMark`（默认启用）,jdk类加载器不会卸载，始终存在。

#### 回收巨型对象（jdk8u60）

![image-20240122113904372](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122113904372.png)

- 一个对象大于region的一半时，就称为巨型对象
- G1不会对巨型对象进行拷贝
- 回收时被优先考虑巨型对象
- 通过检查`Remembered Set`，巨型对象同老年代的引用为0时，就可以在新生代垃圾回收时处理掉（优先回收）

#### 并发标记起始时间的调整（jdk9）

当老年代占比达到阈值（默认45%）时，如果垃圾回收速度小于垃圾产生速度，并发收集失败，会触发Full GC（多线程）。**为了避免Full GC，可以提前开始并发标记、垃圾回收**。

- jdk9之前，使用`-XX:InitatingHeapOccupancyPercent`类设置老年代占比的阈值，当达到这个阈值时，进行CMS垃圾回收（但是这个阈值如果过大了，则会导致Full GC；如果设置得过小了，则会频繁地并发标记、混合收集）

- jdk9之后，可以动态调整（以避免Full GC）
  - `-XX:InitatingHeapOccupancyPercent` 设置初始值
  - 在运行过程中会动态调整，阈值会动态调大、调小
  - 添加一个安全的空挡空间

#### 查看虚拟机参数

```bash
java -XX:+PrintFlagsFinal -version | findstr "GC"
```

### 垃圾回收调优

#### 确定目标

吞吐量优先（科学计算）还是响应时间优先（互联网项目）

- 响应时间优先。CMS，G1，ZGC
- 吞吐量优先。Parallel GC

#### 考虑代码方面的原因

- 数据是否太多？

  例如：`resultSet = statement.executeQuery("select * from 大表")`。此处如果将整个大表全部读入内存，则可能会导致内存溢出，此处应该修改为`resultSet = statement.executeQuery("select * from 大表 limit N")` ，一次只读一部分表进入内存（前N条数据）。

- 数据是否太臃肿？

  例1：使用Integer来存储整型数据，歧视也可以考虑使用int来存储，这样更加节省堆空间。

  例2：查询用户，包括：详细住址，年龄，性别等很多属性。如果我们只要用户的年龄，那么我们只要吧用户的年龄查出来即可，不必把用户的其他信息也一并查询出来。

- 是否存在内存泄漏？

  例1：static Map map = new HashMap();。对于这些长时间存在的对象，推荐使用软引用、弱引用，也可以考虑第三方缓存实现，例如redis，cache。

#### 新生代调优

`-Xnn` 设置新生代初始和最大内存。

新生代特点：

- 所有new操作的内存分配的内存很廉价
- TLAB（thread-local allocation buffer），每个线程局部私有的伊甸园，可以用于创建对象。
- 死亡对象的回收代价是零（复制算法，把还存活的对象从幸存区From复制到幸存区To）
- 大部分对象用过即死亡
- Minor GC的时间远远低于Full GC（1-2个数量级）

新生代越大越好？

- 当新生代比较小时，则会频繁地发生Minor GC；当新生代过大时，则导致老年代空间较小，则Full GC发生的频率将增大。Oracle推荐新生代的空间占比大于25%，小于50%。

新生代理想的大小

- 并发量 * 一次请求响应占用的内存

幸存区需要保存：当前活跃对象（一会会被回收） + 需要晋升的对象

如果幸存区比较小（空间紧张），则JJVM会动态去调整晋升阈值，一些age较小的对象会提前晋升到老年代，但是如果一个存活时间较短的对象晋升到了老年代，则需要等到老年代内存不足时，触发Full GC，才能将其内存回收。

晋升阈值：`  -XX:MaxTenuringThreshold=threshold`

打印晋升详情：`  -XX:+PrintTenuringDistribution`

晋升阈值应该得当，让长时间存活的对象尽早晋升到老年代，如果长时间存活的对象一直留在幸存区，只能耗费幸存区的内存，会导致对象一直被复制（从幸存区From复制到幸存区To，这会加重复制开销）对性能是一种负担。

#### 老年代调优

以CMS为例

- CMS的老年代越大越好（CMS在垃圾回收的同时，其他用户进程在并发的执行，这会产生**浮动垃圾**，若浮动垃圾导致内存不足，则并发失败，退化为串行 `SerialOld`，所以老年代越大越好，避免浮动垃圾引起的并发失败）
- 在老年代调优之前先尝试不做调优。如果没有发生Full GC，则不做老年代调优；如果发生了Full GC，也先尝试对新生代进行调优，然后再对老年代调优
- 观察Full GC时，老年代内存占用情况，然后将老年代内存调大1/4 ~ 1/3。
- `-XX:InitatingHeapOccupancyPercent` 设置老年代内存占比的阈值，当达到阈值时，进行CMS垃圾回收。一般是75%-80%

#### GC调优案例

案例1：Full GC 和 Minor GC频繁

思考：新生代空间紧张，不断发生Minor Gc，然后导致幸存区空间紧张，会导致晋升阈值下降，大量存活时间较短的对象进入老年代，导致老年代内存不足，发生Full GC。

解决：调大新生代内存，然后Minor GC发生的频率下降；增大幸存区空间，提高晋升阈值，则生命周期较短的对象尽可能留在新生代，而不进入老年代，减少老年代内存占用，老年代Full GC发生频率也将下降。

案例2：请求高峰期发生Full GC，单次暂停时间特别长（CMS）

思考：查看GC日志，可以发现：CMS的初始标记和并发标记速度都是比较快的，CMS单次暂停时间特别长可能是重新标记（会扫描整个堆内存，不光扫描老年代，也会扫描新生代，这是一个遍历算法）阶段导致的，因为重新标记阶段会导致STW。

解决：开启 `-XX:+CMSScavengeBeforeRemark` 开关。会在重新标记之前，对新生代进行一次垃圾回收`UseParNewGC`，减轻重新标记的工作量。

案例3：老年代充裕的情况下，发生了Full GC（CMS jdk 1.7）

解决：因为jdk 1.7，永久代内存不足也会导致Full GC；jdk 1.8采用元空间，元空间内存大小等于操作系统内存大小，所以是比较充裕的，所以不会导致元空间不足。

## 类加载和字节码技术

![image-20240122170822055](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122170822055.png)

java源代码先编译成字节码文件.class，然后通过类加载器加载到虚拟机，然后又执行引擎中的解释器解释执行，解释过程中，会对热点代码进行运行期的编译处理。

### 类文件结构

JVM规范下，类文件结构：

![image-20240122171309704](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122171309704.png)

magic：**魔数**，0-3字节，`ca fe ba be` ，表示是否是.class类型的文件。

minor_version和major_version：**版本信息**，`00 00 00 34`，表示Java 8，小版本是`00 00`，大版本是`00 34`。

**常量池**（constant_pool）。首先是常量池长度，两个字节 `00 23` 表示常量池大小为35，所以常量池有#1~#34项，#0项不计入。

对于第#1项 `0a`（十进制下是10）通过查表发现，表示一个Method引用信息，`00 06`（十进制下是6） 和`00 15`（十进制下是21），#6项表示这个方法的所属类，#21表示这个方法的方法名；

再看第#6项，`07`（十进制下是7）通过查表发现，表示一个Class信息，`00 1c`（十进制下是28）；

再看第#28项，`01`（十进制下是1）通过查表发现，表示一个utf8串，紧接着的`00 10`（十进制下是16）表示长度，然后读取接下来16个字符，即为类名`java/lang/Object`；

再看第#21项，`0c`（十进制下是12）通过查表发现是名称+类型，`00 07` 表示名称，`00 08` 表示类型；

再看第#7项，`01`（十进制下是1）通过查表发现，表示一个utf8，紧接着的`00 06`（十进制下是6）表示长度，然后读取接下来6个字符，即为名称`<init>`，即构造方法；

再看第#8项，`01`（十进制下是1）通过查表发现，表示一个utf8，紧接着的`00 03`（十进制下是3）表示长度，然后读取接下来3个字符，即为类型`()v`，表示无参、无返回值.

常量池类型表：

![image-20240122181758889](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122181758889.png)

再看第#2项，`09`（十进制下是9）通过查表发现，表示一个Field信息，`00 16` 和 `00 17` 表示它引用了常量池中#22和#23来获得这个成员变量的所属类和成员变量名；

再看第#22项，`07`（十进制下是7），表示Class信息，`00 1d` 引用了常量池中的#29项；

再看第#29项，`01`（十进制下是1），表示一个utf8串，紧接着的`00 10`（十进制下是16）表示长度，然后读取接下来16个字符，即为类型 `java/lang/System`；

再看第#23项，`0c`（十进制下是12）通过查表发现是名称+类型，`00 1e` （十进制下是30）表示名称，`00 1f` （十进制下是31）表示类型；

再看第#30项，`01`（十进制下是1）通过查表发现，表示一个utf8，紧接着的`00 03`（十进制下是3）表示长度，然后读取接下来3个字符，即为名称 `out`；

再看第#31项，`01`（十进制下是1）通过查表发现，表示一个utf8，紧接着的`00 15`（十进制下是21）表示长度，然后读取接下来21个字符，即为类型 `Ljava/io/PrintStream;` 。

**访问标识**

![image-20240122185301613](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122185301613.png)

`0x0020` 表示类；`0x1000` 表示人工合成的，不是源代码。

例子：对于`0x0021` 是 `0x0020`（类）+ `0x0001`（公共的），即为公共的类；

**本类的全限定名**、**父类的全限定名**、**接口**（数量，数组）、**成员变量**（成员变量数目、成员变量数组）

成员变量类型![image-20240122185934822](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122185934822.png)

**方法信息**（方法数量、方法数组），一个方法有访问修饰符，名称，参数描述，方法属性数量，方法属性组成

**附加属性**

![image-20240122190927568](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122190927568.png)

### 字节码指令

例1：

![image-20240122192143013](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122192143013.png)

例2：

![image-20240122192526998](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122192526998.png)

### javap

指令：

```bash
javap -v .\HelloWorld.class
```

运行结果：

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/HelloWorld.class
  Last modified 2024年1月22日; size 551 bytes
  SHA-256 checksum c16e28c2e077f0e224b171d173fad5c68dbb9b01b23acd3e8df94d94b4c7fa85
  Compiled from "HelloWorld.java"
public class ustc.iat.HelloWorld
  minor version: 0
  major version: 52						  // jdk 8
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER   // 访问修饰符，public class
  this_class: #5                          // ustc/iat/HelloWorld
  super_class: #6                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #21.#22        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #23            // hello world
   #4 = Methodref          #24.#25        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #26            // ustc/iat/HelloWorld
   #6 = Class              #27            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lustc/iat/HelloWorld;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               SourceFile
  #19 = Utf8               HelloWorld.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = Class              #28            // java/lang/System
  #22 = NameAndType        #29:#30        // out:Ljava/io/PrintStream;
  #23 = Utf8               hello world
  #24 = Class              #31            // java/io/PrintStream
  #25 = NameAndType        #32:#33        // println:(Ljava/lang/String;)V
  #26 = Utf8               ustc/iat/HelloWorld
  #27 = Utf8               java/lang/Object
  #28 = Utf8               java/lang/System
  #29 = Utf8               out
  #30 = Utf8               Ljava/io/PrintStream;
  #31 = Utf8               java/io/PrintStream
  #32 = Utf8               println
  #33 = Utf8               (Ljava/lang/String;)V
{
  public ustc.iat.HelloWorld();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
//  栈的深度 局部变量表的长度 参数长度
      stack=1, locals=1, args_size=1
         0: aload_0  // this
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
        // line 7 表示 java源代码的行号
        // 0 表示 字节码中的行号
      LocalVariableTable: // 本地变量表
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/HelloWorld;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String hello world
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable: // java源代码 & 字节码行号对应
        line 10: 0
        line 11: 8
      LocalVariableTable: // 本地变量表
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
}
SourceFile: "HelloWorld.java"
```

### 图解方法执行流程

java源代码：

```java
public class Demo3_1 {    
	public static void main(String[] args) {        
		int a = 10;        
		int b = Short.MAX_VALUE + 1;        
		int c = a + b;        
		System.out.println(c);   
    } 
}
```

字节码文件：

```bash
PS D:\ideaproj\jvm\target\classes\ustc\iat> javap -v .\Demo3_1.class
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_1.class
  Last modified 2024年1月22日; size 601 bytes
  SHA-256 checksum 394469d7466e7339fc65faa5c710e49d7aa9dc2676afa45c6d38ac7d7161b2ca
  Compiled from "Demo3_1.java"
public class ustc.iat.Demo3_1
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #6                          // ustc/iat/Demo3_1
  super_class: #7                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #7.#25         // java/lang/Object."<init>":()V
   #2 = Class              #26            // java/lang/Short
   #3 = Integer            32768
   #4 = Fieldref           #27.#28        // java/lang/System.out:Ljava/io/PrintStream;
   #5 = Methodref          #29.#30        // java/io/PrintStream.println:(I)V
   #6 = Class              #31            // ustc/iat/Demo3_1
   #7 = Class              #32            // java/lang/Object
   #8 = Utf8               <init>
   #9 = Utf8               ()V
  #10 = Utf8               Code
  #11 = Utf8               LineNumberTable
  #12 = Utf8               LocalVariableTable
  #13 = Utf8               this
  #14 = Utf8               Lustc/iat/Demo3_1;
  #15 = Utf8               main
  #16 = Utf8               ([Ljava/lang/String;)V
  #17 = Utf8               args
  #18 = Utf8               [Ljava/lang/String;
  #19 = Utf8               a
  #20 = Utf8               I
  #21 = Utf8               b
  #22 = Utf8               c
  #23 = Utf8               SourceFile
  #24 = Utf8               Demo3_1.java
  #25 = NameAndType        #8:#9          // "<init>":()V
  #26 = Utf8               java/lang/Short
  #27 = Class              #33            // java/lang/System
  #28 = NameAndType        #34:#35        // out:Ljava/io/PrintStream;
  #29 = Class              #36            // java/io/PrintStream
  #30 = NameAndType        #37:#38        // println:(I)V
  #31 = Utf8               ustc/iat/Demo3_1
  #32 = Utf8               java/lang/Object
  #33 = Utf8               java/lang/System
  #34 = Utf8               out
  #35 = Utf8               Ljava/io/PrintStream;
  #36 = Utf8               java/io/PrintStream
  #37 = Utf8               println
  #38 = Utf8               (I)V
{
  public ustc.iat.Demo3_1();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_1;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=1
         0: bipush        10
         2: istore_1
         3: ldc           #3                  // int 32768
         5: istore_2
         6: iload_1
         7: iload_2
         8: iadd
         9: istore_3
        10: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;      
        13: iload_3
        14: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
        17: return
      LineNumberTable:
        line 9: 0
        line 10: 3
        line 11: 6
        line 12: 10
        line 13: 17
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      18     0  args   [Ljava/lang/String;
            3      15     1     a   I
            6      12     2     b   I
           10       8     3     c   I
}
SourceFile: "Demo3_1.java"
```

#### 常量池载入运行时常量池

![image-20240122194615900](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122194615900.png)

如果int的数值没有超出short，则直接和字节码存在一起，（例如int a = 10;）；一旦超出short存储范围，则存储在常量池中。

#### 方法字节码载入方法区

![image-20240122195049637](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122195049637.png)

#### main线程开始执行，分配栈帧内存

![image-20240122195146133](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122195146133.png)

绿色的是局部变量表，蓝色的是操作数栈。根据 `stack=2, locals=4, args_size=1` 和 `stack=1, locals=1, args_size=1`，可知局部变量表大小为4，操作数栈大小为2。

#### bipush 10

将一个 byte 压入操作数栈（其长度会补齐 4 个字节），类似的指令还有

- sipush 将一个 short 压入操作数栈（其长度会补齐 4 个字节）
- ldc 将一个 int 压入操作数栈
- ldc2_w 将一个 long 压入操作数栈（分两次压入，因为 long 是 8 个字节）
- 这里小的数字都是和字节码指令存在一起，超过 short 范围的数字存入了常量池

![image-20240122195652745](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122195652745.png)

#### istore_1

将操作数栈栈顶元素弹出，放入局部变量表的 slot 1 中，对应代码中的 a = 10

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210210230717611.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70)

#### ldc #3

读取运行时常量池中 #3 ，即 32768 (超过 short 最大值范围的数会被放到运行时常量池中)，将其加载到操作数栈中。注意 Short.MAX_VALUE 是 32767，所以 32768 = Short.MAX_VALUE + 1 实际是在编译期间计算好的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210210230918171.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70)

#### istore 2

将操作数栈中的元素弹出，放到局部变量表的 2 号位置

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210210231005919.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70)

#### iload1 iload2

将局部变量表中 1 号位置和 2 号位置的元素放入操作数栈中。因为只能在操作数栈中执行运算操作

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210210231211695.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70)

#### iadd

将操作数栈中的两个元素弹出栈并相加，结果在压入操作数栈中。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210210231236404.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70)

#### istore 3

将操作数栈中的元素弹出，放入局部变量表的3号位置。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210210231319967.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70)

#### getstatic #4

在运行时常量池中找到 #4 ，发现是一个对象，在堆内存中找到该对象，并将其引用放入操作数栈中

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210210231759663.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210210231827339.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70)

#### iload 3

将局部变量表中 3 号位置的元素压入操作数栈中。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210210232008706.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70)

#### invokevirtual #5

找到常量池 #5 项，定位到方法区 java/io/PrintStream.println:(I)V 方法，生成新的栈帧（分配 locals、stack等），传递参数，执行新栈帧中的字节码

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210210232148931.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70)

执行完毕，弹出栈帧，清除 main 操作数栈内容

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210210232228908.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70)

#### return

完成 main 方法调用，弹出 main 栈帧，程序结束

### 案例

**案例1**

java源代码

```java
package ustc.iat;

/**
 * @author mkzou
 * @create 2024-01-22 21:10
 */
public class Demo3_2 {

	public static void main(String[] args) {
		int a = 10;
		int b = a++ + ++a + a--;
		System.out.println(a);
		System.out.println(b);
	}

}
```

字节码

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_2.class
  Last modified 2024年1月22日; size 576 bytes
  SHA-256 checksum db5aafb4983fb519329c35cfdec115b99c06a29fe3e30addba73fd04c43d2b39
  Compiled from "Demo3_2.java"
public class ustc.iat.Demo3_2
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #4                          // ustc/iat/Demo3_2
  super_class: #5                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #5.#22         // java/lang/Object."<init>":()V
   #2 = Fieldref           #23.#24        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = Methodref          #25.#26        // java/io/PrintStream.println:(I)V
   #4 = Class              #27            // ustc/iat/Demo3_2
   #5 = Class              #28            // java/lang/Object
   #6 = Utf8               <init>
   #7 = Utf8               ()V
   #8 = Utf8               Code
   #9 = Utf8               LineNumberTable
  #10 = Utf8               LocalVariableTable
  #11 = Utf8               this
  #12 = Utf8               Lustc/iat/Demo3_2;
  #13 = Utf8               main
  #14 = Utf8               ([Ljava/lang/String;)V
  #15 = Utf8               args
  #16 = Utf8               [Ljava/lang/String;
  #17 = Utf8               a
  #18 = Utf8               I
  #19 = Utf8               b
  #20 = Utf8               SourceFile
  #21 = Utf8               Demo3_2.java
  #22 = NameAndType        #6:#7          // "<init>":()V
  #23 = Class              #29            // java/lang/System
  #24 = NameAndType        #30:#31        // out:Ljava/io/PrintStream;
  #25 = Class              #32            // java/io/PrintStream
  #26 = NameAndType        #33:#34        // println:(I)V
  #27 = Utf8               ustc/iat/Demo3_2
  #28 = Utf8               java/lang/Object
  #29 = Utf8               java/lang/System
  #30 = Utf8               out
  #31 = Utf8               Ljava/io/PrintStream;
  #32 = Utf8               java/io/PrintStream
  #33 = Utf8               println
  #34 = Utf8               (I)V
{
  public ustc.iat.Demo3_2();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_2;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: bipush        10
         2: istore_1
         3: iload_1
         4: iinc          1, 1
         7: iinc          1, 1
        10: iload_1
        11: iadd
        12: iload_1
        13: iinc          1, -1
        16: iadd
        17: istore_2
        18: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;      
        21: iload_1
        22: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
        25: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;      
        28: iload_2
        29: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
        32: return
      LineNumberTable:
        line 10: 0
        line 11: 3
        line 12: 18
        line 13: 25
        line 14: 32
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      33     0  args   [Ljava/lang/String;
            3      30     1     a   I
           18      15     2     b   I
}
SourceFile: "Demo3_2.java"
```

分析：

- iinc 直接在局部变量slot上进行运算
- a++ 和 ++a 的区别是先执行 iload 还是先执行 iinc
  - a++ 先执行 iload，再执行innc
  - ++a 先执行 innc，再执行iload

### 条件判断指令

对于byte，short，char都会按照int比较，因为操作数栈是4字节的

![image-20240122212537575](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240122212537575.png)

java源码：

```java
public class Demo3_3 {

	public static void main(String[] args) {
		int a = 0;
		if(a == 0) {
			a = 10;
		} else {
			a = 20;
		}
	}

}
```

字节码：

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_3.class
  Last modified 2024年1月22日; size 460 bytes
  SHA-256 checksum ccdff4eeb49f61cc4ae7e430dd3a7e1c231ec0f928d96a234fcbb1959ac8e07a
  Compiled from "Demo3_3.java"
public class ustc.iat.Demo3_3
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // ustc/iat/Demo3_3
  super_class: #3                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #3.#20         // java/lang/Object."<init>":()V
   #2 = Class              #21            // ustc/iat/Demo3_3
   #3 = Class              #22            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               LocalVariableTable
   #9 = Utf8               this
  #10 = Utf8               Lustc/iat/Demo3_3;
  #11 = Utf8               main
  #12 = Utf8               ([Ljava/lang/String;)V
  #13 = Utf8               args
  #14 = Utf8               [Ljava/lang/String;
  #15 = Utf8               a
  #16 = Utf8               I
  #17 = Utf8               StackMapTable
  #18 = Utf8               SourceFile
  #19 = Utf8               Demo3_3.java
  #20 = NameAndType        #4:#5          // "<init>":()V
  #21 = Utf8               ustc/iat/Demo3_3
  #22 = Utf8               java/lang/Object
{
  public ustc.iat.Demo3_3();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_3;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=2, args_size=1
         0: iconst_0
         1: istore_1
         2: iload_1
         3: ifne          12
         6: bipush        10
         8: istore_1
         9: goto          15
        12: bipush        20
        14: istore_1
        15: return
      LineNumberTable:
        line 11: 0
        line 12: 2
        line 13: 6
        line 15: 12
        line 17: 15
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      16     0  args   [Ljava/lang/String;
            2      14     1     a   I
      StackMapTable: number_of_entries = 2
        frame_type = 252 /* append */
          offset_delta = 12
          locals = [ int ]
        frame_type = 2 /* same */
}
SourceFile: "Demo3_3.java"
```

### 循环判断指令

java源码：

```java
public class Demo3_3 {

	public static void main(String[] args) {
		int a = 0;
		while(a < 10) {
			a++;
		}
	}

}
```

字节码：

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_3.class
  Last modified 2024年1月22日; size 455 bytes
  SHA-256 checksum 2a15dab0c2fbaf1038ff0c0326a6f1438b2765c555cc9dd855ae7edb34ea9a05
  Compiled from "Demo3_3.java"
public class ustc.iat.Demo3_3
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // ustc/iat/Demo3_3
  super_class: #3                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #3.#20         // java/lang/Object."<init>":()V
   #2 = Class              #21            // ustc/iat/Demo3_3
   #3 = Class              #22            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               LocalVariableTable
   #9 = Utf8               this
  #10 = Utf8               Lustc/iat/Demo3_3;
  #11 = Utf8               main
  #12 = Utf8               ([Ljava/lang/String;)V
  #13 = Utf8               args
  #14 = Utf8               [Ljava/lang/String;
  #15 = Utf8               a
  #16 = Utf8               I
  #17 = Utf8               StackMapTable
  #18 = Utf8               SourceFile
  #19 = Utf8               Demo3_3.java
  #20 = NameAndType        #4:#5          // "<init>":()V
  #21 = Utf8               ustc/iat/Demo3_3
  #22 = Utf8               java/lang/Object
{
  public ustc.iat.Demo3_3();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_3;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: iconst_0
         1: istore_1
         2: iload_1
         3: bipush        10
         5: if_icmpge     14
         8: iinc          1, 1
        11: goto          2
        14: return
      LineNumberTable:
        line 10: 0
        line 11: 2
        line 12: 8
        line 14: 14
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      15     0  args   [Ljava/lang/String;
            2      13     1     a   I
      StackMapTable: number_of_entries = 2
        frame_type = 252 /* append */
          offset_delta = 2
          locals = [ int ]
        frame_type = 11 /* same */
}
SourceFile: "Demo3_3.java"
```

**练习**

java源码：

```java
public class Demo3_3 {
    
	public static void main(String[] args) {
		int i = 0;
		int x = 0;
		while (i < 10) {
			x = x++;
			i++;
		}
		System.out.println(x); // 0
	}

}
```

字节码：

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_3.class
  Last modified 2024年1月22日; size 610 bytes
  SHA-256 checksum cd8f74fa04f932381863645f153339242ce7f6db138b7508e8d37992111f4f03
  Compiled from "Demo3_3.java"
public class ustc.iat.Demo3_3
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #4                          // ustc/iat/Demo3_3
  super_class: #5                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #5.#23         // java/lang/Object."<init>":()V
   #2 = Fieldref           #24.#25        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = Methodref          #26.#27        // java/io/PrintStream.println:(I)V
   #4 = Class              #28            // ustc/iat/Demo3_3
   #5 = Class              #29            // java/lang/Object
   #6 = Utf8               <init>
   #7 = Utf8               ()V
   #8 = Utf8               Code
   #9 = Utf8               LineNumberTable
  #10 = Utf8               LocalVariableTable
  #11 = Utf8               this
  #12 = Utf8               Lustc/iat/Demo3_3;
  #13 = Utf8               main
  #14 = Utf8               ([Ljava/lang/String;)V
  #15 = Utf8               args
  #16 = Utf8               [Ljava/lang/String;
  #17 = Utf8               i
  #18 = Utf8               I
  #19 = Utf8               x
  #20 = Utf8               StackMapTable
  #21 = Utf8               SourceFile
  #22 = Utf8               Demo3_3.java
  #23 = NameAndType        #6:#7          // "<init>":()V
  #24 = Class              #30            // java/lang/System
  #25 = NameAndType        #31:#32        // out:Ljava/io/PrintStream;
  #26 = Class              #33            // java/io/PrintStream
  #27 = NameAndType        #34:#35        // println:(I)V
  #28 = Utf8               ustc/iat/Demo3_3
  #29 = Utf8               java/lang/Object
  #30 = Utf8               java/lang/System
  #31 = Utf8               out
  #32 = Utf8               Ljava/io/PrintStream;
  #33 = Utf8               java/io/PrintStream
  #34 = Utf8               println
  #35 = Utf8               (I)V
{
  public ustc.iat.Demo3_3();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_3;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: iconst_0
         1: istore_1
         2: iconst_0
         3: istore_2
         4: iload_1
         5: bipush        10
         7: if_icmpge     21
        10: iload_2
        11: iinc          2, 1
        14: istore_2
        15: iinc          1, 1
        18: goto          4
        21: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;      
        24: iload_2
        25: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
        28: return
      LineNumberTable:
        line 11: 0
        line 12: 2
        line 13: 4
        line 14: 10
        line 15: 15
        line 17: 21
        line 18: 28
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      29     0  args   [Ljava/lang/String;
            2      27     1     i   I
            4      25     2     x   I
      StackMapTable: number_of_entries = 2
        frame_type = 253 /* append */
          offset_delta = 4
          locals = [ int, int ]
        frame_type = 16 /* same */
}
SourceFile: "Demo3_3.java"
```

iload_2：本地变量表 -> 操作数栈

istore_2：操作数栈 -> 本地变量表

### 构造方法

1.  \<cinit>()V

java源代码

```java
public class Demo3_4 {

	static int i = 10;

	static {
		i = 20;
	}

	static {
		i = 30;
	}

	public static void main(String[] args) {

	}

}
```

字节码

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_4.class
  Last modified 2024年1月22日; size 485 bytes
  SHA-256 checksum d53da8da6814ed625d45cfcdc6c3467082927d77b88cc9b918893660efd67638
  Compiled from "Demo3_4.java"
public class ustc.iat.Demo3_4
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #3                          // ustc/iat/Demo3_4
  super_class: #4                         // java/lang/Object
  interfaces: 0, fields: 1, methods: 3, attributes: 1
Constant pool:
   #1 = Methodref          #4.#21         // java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#22         // ustc/iat/Demo3_4.i:I
   #3 = Class              #23            // ustc/iat/Demo3_4
   #4 = Class              #24            // java/lang/Object
   #5 = Utf8               i
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lustc/iat/Demo3_4;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               <clinit>
  #19 = Utf8               SourceFile
  #20 = Utf8               Demo3_4.java
  #21 = NameAndType        #7:#8          // "<init>":()V
  #22 = NameAndType        #5:#6          // i:I
  #23 = Utf8               ustc/iat/Demo3_4
  #24 = Utf8               java/lang/Object
{
  static int i;
    descriptor: I
    flags: (0x0008) ACC_STATIC

  public ustc.iat.Demo3_4();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_4;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 21: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       1     0  args   [Ljava/lang/String;

  static {};
    descriptor: ()V
    flags: (0x0008) ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: bipush        10
         2: putstatic     #2                  // Field i:I
         5: bipush        20
         7: putstatic     #2                  // Field i:I
        10: bipush        30
        12: putstatic     #2                  // Field i:I
        15: return
      LineNumberTable:
        line 9: 0
        line 12: 5
        line 16: 10
        line 17: 15
}
SourceFile: "Demo3_4.java"
```

编译器会按照从上到下的顺序，收集所有static静态代码块和静态成员赋值的代码，合并成一个特殊的方法\<cint>()V；

2.init

java源代码

```java
public class Demo3_5 {

	private String a = "s1";

	{
		b = 20;
	}

	private int b = 10;

	{
		a = "s2";
	}

	public Demo3_5(String a, int b) {
		this.a = a;
		this.b = b;
	}

	public static void main(String[] args) {
		Demo3_5 d = new Demo3_5("s3", 30);
		System.out.println(d.a);
		System.out.println(d.b);
	}

}
```

字节码

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_5.class
  Last modified 2024年1月22日; size 798 bytes
  SHA-256 checksum 67d4548a012e4ce1c59a95aa57e188d321a7ef862cd765e8eb9528a823390cbc
  Compiled from "Demo3_5.java"
public class ustc.iat.Demo3_5
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #6                          // ustc/iat/Demo3_5
  super_class: #12                        // java/lang/Object
  interfaces: 0, fields: 2, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #12.#31        // java/lang/Object."<init>":()V
   #2 = String             #32            // s1
   #3 = Fieldref           #6.#33         // ustc/iat/Demo3_5.a:Ljava/lang/String;
   #4 = Fieldref           #6.#34         // ustc/iat/Demo3_5.b:I
   #5 = String             #35            // s2
   #6 = Class              #36            // ustc/iat/Demo3_5
   #7 = String             #37            // s3
   #8 = Methodref          #6.#38         // ustc/iat/Demo3_5."<init>":(Ljava/lang/String;I)V
   #9 = Fieldref           #39.#40        // java/lang/System.out:Ljava/io/PrintStream;
  #10 = Methodref          #41.#42        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #11 = Methodref          #41.#43        // java/io/PrintStream.println:(I)V
  #12 = Class              #44            // java/lang/Object
  #13 = Utf8               a
  #14 = Utf8               Ljava/lang/String;
  #15 = Utf8               b
  #16 = Utf8               I
  #17 = Utf8               <init>
  #18 = Utf8               (Ljava/lang/String;I)V
  #19 = Utf8               Code
  #20 = Utf8               LineNumberTable
  #21 = Utf8               LocalVariableTable
  #22 = Utf8               this
  #23 = Utf8               Lustc/iat/Demo3_5;
  #24 = Utf8               main
  #25 = Utf8               ([Ljava/lang/String;)V
  #26 = Utf8               args
  #27 = Utf8               [Ljava/lang/String;
  #28 = Utf8               d
  #29 = Utf8               SourceFile
  #30 = Utf8               Demo3_5.java
  #31 = NameAndType        #17:#45        // "<init>":()V
  #32 = Utf8               s1
  #33 = NameAndType        #13:#14        // a:Ljava/lang/String;
  #34 = NameAndType        #15:#16        // b:I
  #35 = Utf8               s2
  #36 = Utf8               ustc/iat/Demo3_5
  #37 = Utf8               s3
  #38 = NameAndType        #17:#18        // "<init>":(Ljava/lang/String;I)V
  #39 = Class              #46            // java/lang/System
  #40 = NameAndType        #47:#48        // out:Ljava/io/PrintStream;
  #41 = Class              #49            // java/io/PrintStream
  #42 = NameAndType        #50:#51        // println:(Ljava/lang/String;)V
  #43 = NameAndType        #50:#52        // println:(I)V
  #44 = Utf8               java/lang/Object
  #45 = Utf8               ()V
  #46 = Utf8               java/lang/System
  #47 = Utf8               out
  #48 = Utf8               Ljava/io/PrintStream;
  #49 = Utf8               java/io/PrintStream
  #50 = Utf8               println
  #51 = Utf8               (Ljava/lang/String;)V
  #52 = Utf8               (I)V
{
  public ustc.iat.Demo3_5(java.lang.String, int);
    descriptor: (Ljava/lang/String;I)V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=3
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: ldc           #2                  // String s1
         7: putfield      #3                  // Field a:Ljava/lang/String;
        10: aload_0
        11: bipush        20
        13: putfield      #4                  // Field b:I
        16: aload_0
        17: bipush        10
        19: putfield      #4                  // Field b:I
        22: aload_0
        23: ldc           #5                  // String s2
        25: putfield      #3                  // Field a:Ljava/lang/String;
        28: aload_0
        29: aload_1
        30: putfield      #3                  // Field a:Ljava/lang/String;
        33: aload_0
        34: iload_2
        35: putfield      #4                  // Field b:I
        38: return
      LineNumberTable:
        line 21: 0
        line 9: 4
        line 12: 10
        line 15: 16
        line 18: 22
        line 22: 28
        line 23: 33
        line 24: 38
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      39     0  this   Lustc/iat/Demo3_5;
            0      39     1     a   Ljava/lang/String;
            0      39     2     b   I

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=4, locals=2, args_size=1
         0: new           #6                  // class ustc/iat/Demo3_5
         3: dup
         4: ldc           #7                  // String s3
         6: bipush        30
         8: invokespecial #8                  // Method "<init>":(Ljava/lang/String;I)V
        11: astore_1
        12: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;      
        15: aload_1
        16: getfield      #3                  // Field a:Ljava/lang/String;
        19: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        22: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;      
        25: aload_1
        26: getfield      #4                  // Field b:I
        29: invokevirtual #11                 // Method java/io/PrintStream.println:(I)V
        32: return
      LineNumberTable:
        line 27: 0
        line 28: 12
        line 29: 22
        line 30: 32
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      33     0  args   [Ljava/lang/String;
           12      21     1     d   Lustc/iat/Demo3_5;
}
SourceFile: "Demo3_5.java"
```

### 方法调用

java源代码

```java
public class Demo3_6 {

	public Demo3_6() {

	}

	private void test1() {

	}

	private final void test2() {

	}

	public void test3() {

	}

	public static void test4() {

	}

	public static void main(String[] args) {
		Demo3_6 obj = new Demo3_6();
		obj.test1();
		obj.test2();
		obj.test3();
		Demo3_6.test4();
	}
}
```

字节码

总结：

- invokespecial（静态绑定）：private类型的方法、private final类型的方法、构造方法

- invokevirtual（动态绑定）：public普通方法（可能存在方法重写的情况，所以在编译期间无法确定调用哪个对象的方法）

- invokestatic（静态绑定）：static静态方法

  静态绑定的可以在编译期间唯一确定，静态绑定性能更优。

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_6.class
  Last modified 2024年1月22日; size 732 bytes
  SHA-256 checksum 40d2aafc7a0cb2e7f6c40330d34782de7d8945a88099051a0891c588420da1ee
  Compiled from "Demo3_6.java"
public class ustc.iat.Demo3_6
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // ustc/iat/Demo3_6
  super_class: #8                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 6, attributes: 1
Constant pool:
   #1 = Methodref          #8.#27         // java/lang/Object."<init>":()V
   #2 = Class              #28            // ustc/iat/Demo3_6
   #3 = Methodref          #2.#27         // ustc/iat/Demo3_6."<init>":()V
   #4 = Methodref          #2.#29         // ustc/iat/Demo3_6.test1:()V
   #5 = Methodref          #2.#30         // ustc/iat/Demo3_6.test2:()V
   #6 = Methodref          #2.#31         // ustc/iat/Demo3_6.test3:()V
   #7 = Methodref          #2.#32         // ustc/iat/Demo3_6.test4:()V
   #8 = Class              #33            // java/lang/Object
   #9 = Utf8               <init>
  #10 = Utf8               ()V
  #11 = Utf8               Code
  #12 = Utf8               LineNumberTable
  #13 = Utf8               LocalVariableTable
  #14 = Utf8               this
  #15 = Utf8               Lustc/iat/Demo3_6;
  #16 = Utf8               test1
  #17 = Utf8               test2
  #18 = Utf8               test3
  #19 = Utf8               test4
  #20 = Utf8               main
  #21 = Utf8               ([Ljava/lang/String;)V
  #22 = Utf8               args
  #23 = Utf8               [Ljava/lang/String;
  #24 = Utf8               obj
  #25 = Utf8               SourceFile
  #26 = Utf8               Demo3_6.java
  #27 = NameAndType        #9:#10         // "<init>":()V
  #28 = Utf8               ustc/iat/Demo3_6
  #29 = NameAndType        #16:#10        // test1:()V
  #30 = NameAndType        #17:#10        // test2:()V
  #31 = NameAndType        #18:#10        // test3:()V
  #32 = NameAndType        #19:#10        // test4:()V
  #33 = Utf8               java/lang/Object
{
  public ustc.iat.Demo3_6();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 9: 0
        line 11: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_6;

  public void test3();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 23: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       1     0  this   Lustc/iat/Demo3_6;

  public static void test4();
    descriptor: ()V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=0, locals=0, args_size=0
         0: return
      LineNumberTable:
        line 27: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #2                  // class ustc/iat/Demo3_6
         3: dup
         4: invokespecial #3                  // Method "<init>":()V
         7: astore_1
         8: aload_1
         9: invokespecial #4                  // Method test1:()V
        12: aload_1
        13: invokespecial #5                  // Method test2:()V
        16: aload_1
        17: invokevirtual #6                  // Method test3:()V
        20: invokestatic  #7                  // Method test4:()V
        23: return
      LineNumberTable:
        line 30: 0
        line 31: 8
        line 32: 12
        line 33: 16
        line 34: 20
        line 35: 23
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      24     0  args   [Ljava/lang/String;
            8      16     1   obj   Lustc/iat/Demo3_6;
}
SourceFile: "Demo3_6.java"
```

### 多态的原理

`-XX:+UseCompressedOops -XX:+UseCompressedClassPointers ` ：禁用指针压缩

运行HSDB工具

```bash
java -cp .\lib\sa-jdi.jar sun.jvm.hotspot.HSDB
```

因为普通成员方法需要在运行时才能确定具体的内容，所以虚拟机需要调用 invokevirtual 指令
在执行 invokevirtual 指令时，经历了以下几个步骤

- 先通过栈帧中对象的引用找到对象
- 分析对象头，找到对象实际的 Class
- Class 结构中有 vtable
- 查询 vtable 找到方法的具体地址
- 执行方法的字节码

### 异常处理

**try - catch**

java 源代码

```java
public class Demo3_8 {

	public static void main(String[] args) {
		int i = 0;
		try {
			i = 10;
		}catch (Exception e) {
			i = 20;
		}
	}

}
```

字节码

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_8.class
  Last modified 2024-1-23; size 548 bytes
  MD5 checksum 8944742c9fb7c0a38aff02973f3771da
  Compiled from "Demo3_8.java"
public class ustc.iat.Demo3_8
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#25         // java/lang/Object."<init>":()V
   #2 = Class              #26            // java/lang/Exception
   #3 = Class              #27            // ustc/iat/Demo3_8
   #4 = Class              #28            // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Utf8               LineNumberTable
   #9 = Utf8               LocalVariableTable
  #10 = Utf8               this
  #11 = Utf8               Lustc/iat/Demo3_8;
  #12 = Utf8               main
  #13 = Utf8               ([Ljava/lang/String;)V
  #14 = Utf8               e
  #15 = Utf8               Ljava/lang/Exception;
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               i
  #19 = Utf8               I
  #20 = Utf8               StackMapTable
  #21 = Class              #17            // "[Ljava/lang/String;"
  #22 = Class              #26            // java/lang/Exception
  #23 = Utf8               SourceFile
  #24 = Utf8               Demo3_8.java
  #25 = NameAndType        #5:#6          // "<init>":()V
  #26 = Utf8               java/lang/Exception
  #27 = Utf8               ustc/iat/Demo3_8
  #28 = Utf8               java/lang/Object
{
  public ustc.iat.Demo3_8();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_8;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=1
         0: iconst_0
         1: istore_1
         2: bipush        10
         4: istore_1
         5: goto          12
         8: astore_2		// 将异常对象的引用存到局部变量表的 slot 1
         9: bipush        20
        11: istore_1
        12: return
      // 异常表（检测从第二行到第五行（不含）的代码，如果发生异常，然后对异常进行匹配，如果匹配成功，发生的异常是否是type中声明的异常，如果是，则跳转到target）
      Exception table:
         from    to  target type
             2     5     8   Class java/lang/Exception
      LineNumberTable:
        line 10: 0
        line 12: 2
        line 15: 5
        line 13: 8
        line 14: 9
        line 16: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            9       3     2     e   Ljava/lang/Exception;
            0      13     0  args   [Ljava/lang/String;
            2      11     1     i   I
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 8
          locals = [ class "[Ljava/lang/String;", int ]
          stack = [ class java/lang/Exception ]
        frame_type = 3 /* same */
}
SourceFile: "Demo3_8.java"
```

多了一个Exception table

**多个catch块**

java源码

```java
public class Demo3_9 {

	public static void main(String[] args) {
		int i = 0;
		try {
			i = 10;
		} catch (ArithmeticException e) {
			i = 20;
		} catch (NullPointerException e) {
			i = 40;
		} catch (Exception e) {
			i = 30;
		}
	}

}
```

字节码

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_9.class
  Last modified 2024-1-23; size 776 bytes
  MD5 checksum 7c0c6d42c7e94e007a6a31815b31dbcd
  Compiled from "Demo3_9.java"
public class ustc.iat.Demo3_9
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#31         // java/lang/Object."<init>":()V
   #2 = Class              #32            // java/lang/ArithmeticException
   #3 = Class              #33            // java/lang/NullPointerException
   #4 = Class              #34            // java/lang/Exception
   #5 = Class              #35            // ustc/iat/Demo3_9
   #6 = Class              #36            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lustc/iat/Demo3_9;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               e
  #17 = Utf8               Ljava/lang/ArithmeticException;
  #18 = Utf8               Ljava/lang/NullPointerException;
  #19 = Utf8               Ljava/lang/Exception;
  #20 = Utf8               args
  #21 = Utf8               [Ljava/lang/String;
  #22 = Utf8               i
  #23 = Utf8               I
  #24 = Utf8               StackMapTable
  #25 = Class              #21            // "[Ljava/lang/String;"
  #26 = Class              #32            // java/lang/ArithmeticException
  #27 = Class              #33            // java/lang/NullPointerException
  #28 = Class              #34            // java/lang/Exception
  #29 = Utf8               SourceFile
  #30 = Utf8               Demo3_9.java
  #31 = NameAndType        #7:#8          // "<init>":()V
  #32 = Utf8               java/lang/ArithmeticException
  #33 = Utf8               java/lang/NullPointerException
  #34 = Utf8               java/lang/Exception
  #35 = Utf8               ustc/iat/Demo3_9
  #36 = Utf8               java/lang/Object
{
  public ustc.iat.Demo3_9();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 9: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_9;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=3, args_size=1
         0: iconst_0
         1: istore_1
         2: bipush        10
         4: istore_1
         5: goto          26
         8: astore_2
         9: bipush        20
        11: istore_1
        12: goto          26
        15: astore_2
        16: bipush        40
        18: istore_1
        19: goto          26
        22: astore_2
        23: bipush        30
        25: istore_1
        26: return
      Exception table:
         from    to  target type
             2     5     8   Class java/lang/ArithmeticException
             2     5    15   Class java/lang/NullPointerException
             2     5    22   Class java/lang/Exception
      LineNumberTable:
        line 12: 0
        line 14: 2
        line 21: 5
        line 15: 8
        line 16: 9
        line 21: 12
        line 17: 15
        line 18: 16
        line 21: 19
        line 19: 22
        line 20: 23
        line 22: 26
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            9       3     2     e   Ljava/lang/ArithmeticException;
           16       3     2     e   Ljava/lang/NullPointerException;
           23       3     2     e   Ljava/lang/Exception;
            0      27     0  args   [Ljava/lang/String;
            2      25     1     i   I
      StackMapTable: number_of_entries = 4
        frame_type = 255 /* full_frame */
          offset_delta = 8
          locals = [ class "[Ljava/lang/String;", int ]
          stack = [ class java/lang/ArithmeticException ]
        frame_type = 70 /* same_locals_1_stack_item */
          stack = [ class java/lang/NullPointerException ]
        frame_type = 70 /* same_locals_1_stack_item */
          stack = [ class java/lang/Exception ]
        frame_type = 3 /* same */
}
SourceFile: "Demo3_9.java"
```

**multi-catch**

jdk 1.7后支持的新语法

java 源代码

```java
public class Demo3_10 {

	public static void main(String[] args) {
		try {
			Demo3_10 demo310 = new Demo3_10();
			Method method = Demo3_10.class.getMethod("test");
			method.invoke(demo310);
		} catch (NoSuchMethodException | InvocationTargetException | IllegalAccessException e) {
			throw new RuntimeException(e);
		}
	}

	public void test() {
		System.out.println("ok");
	}

}
```

字节码

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_10.class
  Last modified 2024-1-23; size 1309 bytes
  MD5 checksum e3a7d8a638c18568b5ac7445f80b57e3
  Compiled from "Demo3_10.java"
public class ustc.iat.Demo3_10
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #7.#38         // java/lang/Object."<init>":()V
   #2 = Class              #39            // ustc/iat/Demo3_10
   #3 = Methodref          #2.#38         // ustc/iat/Demo3_10."<init>":()V
   #4 = String             #35            // test
   #5 = Class              #40            // java/lang/Class
   #6 = Methodref          #5.#41         // java/lang/Class.getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
   #7 = Class              #42            // java/lang/Object
   #8 = Methodref          #43.#44        // java/lang/reflect/Method.invoke:(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;
   #9 = Class              #45            // java/lang/NoSuchMethodException
  #10 = Class              #46            // java/lang/reflect/InvocationTargetException
  #11 = Class              #47            // java/lang/IllegalAccessException
  #12 = Class              #48            // java/lang/RuntimeException
  #13 = Methodref          #12.#49        // java/lang/RuntimeException."<init>":(Ljava/lang/Throwable;)V
  #14 = Fieldref           #50.#51        // java/lang/System.out:Ljava/io/PrintStream;
  #15 = String             #52            // ok
  #16 = Methodref          #53.#54        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #17 = Utf8               <init>
  #18 = Utf8               ()V
  #19 = Utf8               Code
  #20 = Utf8               LineNumberTable
  #21 = Utf8               LocalVariableTable
  #22 = Utf8               this
  #23 = Utf8               Lustc/iat/Demo3_10;
  #24 = Utf8               main
  #25 = Utf8               ([Ljava/lang/String;)V
  #26 = Utf8               demo310
  #27 = Utf8               method
  #28 = Utf8               Ljava/lang/reflect/Method;
  #29 = Utf8               e
  #30 = Utf8               Ljava/lang/ReflectiveOperationException;
  #31 = Utf8               args
  #32 = Utf8               [Ljava/lang/String;
  #33 = Utf8               StackMapTable
  #34 = Class              #55            // java/lang/ReflectiveOperationException
  #35 = Utf8               test
  #36 = Utf8               SourceFile
  #37 = Utf8               Demo3_10.java
  #38 = NameAndType        #17:#18        // "<init>":()V
  #39 = Utf8               ustc/iat/Demo3_10
  #40 = Utf8               java/lang/Class
  #41 = NameAndType        #56:#57        // getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
  #42 = Utf8               java/lang/Object
  #43 = Class              #58            // java/lang/reflect/Method
  #44 = NameAndType        #59:#60        // invoke:(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;
  #45 = Utf8               java/lang/NoSuchMethodException
  #46 = Utf8               java/lang/reflect/InvocationTargetException
  #47 = Utf8               java/lang/IllegalAccessException
  #48 = Utf8               java/lang/RuntimeException
  #49 = NameAndType        #17:#61        // "<init>":(Ljava/lang/Throwable;)V
  #50 = Class              #62            // java/lang/System
  #51 = NameAndType        #63:#64        // out:Ljava/io/PrintStream;
  #52 = Utf8               ok
  #53 = Class              #65            // java/io/PrintStream
  #54 = NameAndType        #66:#67        // println:(Ljava/lang/String;)V
  #55 = Utf8               java/lang/ReflectiveOperationException
  #56 = Utf8               getMethod
  #57 = Utf8               (Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
  #58 = Utf8               java/lang/reflect/Method
  #59 = Utf8               invoke
  #60 = Utf8               (Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;
  #61 = Utf8               (Ljava/lang/Throwable;)V
  #62 = Utf8               java/lang/System
  #63 = Utf8               out
  #64 = Utf8               Ljava/io/PrintStream;
  #65 = Utf8               java/io/PrintStream
  #66 = Utf8               println
  #67 = Utf8               (Ljava/lang/String;)V
{
  public ustc.iat.Demo3_10();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 10: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_10;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=3, args_size=1
         0: new           #2                  // class ustc/iat/Demo3_10
         3: dup
         4: invokespecial #3                  // Method "<init>":()V
         7: astore_1
         8: ldc           #2                  // class ustc/iat/Demo3_10
        10: ldc           #4                  // String test
        12: iconst_0
        13: anewarray     #5                  // class java/lang/Class
        16: invokevirtual #6                  // Method java/lang/Class.getMethod:(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;
        19: astore_2
        20: aload_2
        21: aload_1
        22: iconst_0
        23: anewarray     #7                  // class java/lang/Object
        26: invokevirtual #8                  // Method java/lang/reflect/Method.invoke:(Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;
        29: pop
        30: goto          43
        33: astore_1
        34: new           #12                 // class java/lang/RuntimeException
        37: dup
        38: aload_1
        39: invokespecial #13                 // Method java/lang/RuntimeException."<init>":(Ljava/lang/Throwable;)V
        42: athrow
        43: return
      Exception table:
         from    to  target type
             0    30    33   Class java/lang/NoSuchMethodException
             0    30    33   Class java/lang/reflect/InvocationTargetException
             0    30    33   Class java/lang/IllegalAccessException
      LineNumberTable:
        line 14: 0
        line 15: 8
        line 16: 20
        line 19: 30
        line 17: 33
        line 18: 34
        line 20: 43
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            8      22     1 demo310   Lustc/iat/Demo3_10;
           20      10     2 method   Ljava/lang/reflect/Method;
           34       9     1     e   Ljava/lang/ReflectiveOperationException;
            0      44     0  args   [Ljava/lang/String;
      StackMapTable: number_of_entries = 2
        frame_type = 97 /* same_locals_1_stack_item */
          stack = [ class java/lang/ReflectiveOperationException ]
        frame_type = 9 /* same */

  public void test();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #14                 // Field java/lang/System.out:Ljava/io/PrintStream;      
         3: ldc           #15                 // String ok
         5: invokevirtual #16                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 23: 0
        line 24: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lustc/iat/Demo3_10;
}
SourceFile: "Demo3_10.java"
```

**finally**

finally中的代码一定会被执行

java源代码

```java
public class Demo3_11 {

	public static void main(String[] args) {
		int i = 0;
		try {
			i = 10;
		} catch (Exception e) {
			i = 20;
		} finally {
			i = 30;
		}
	}

}
```

字节码

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_11.class
  Last modified 2024-1-23; size 631 bytes
  MD5 checksum dcf64e677cb49321c3ae4aacd4a7de39
  Compiled from "Demo3_11.java"
public class ustc.iat.Demo3_11
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#26         // java/lang/Object."<init>":()V
   #2 = Class              #27            // java/lang/Exception
   #3 = Class              #28            // ustc/iat/Demo3_11
   #4 = Class              #29            // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Utf8               LineNumberTable
   #9 = Utf8               LocalVariableTable
  #10 = Utf8               this
  #11 = Utf8               Lustc/iat/Demo3_11;
  #12 = Utf8               main
  #13 = Utf8               ([Ljava/lang/String;)V
  #14 = Utf8               e
  #15 = Utf8               Ljava/lang/Exception;
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               i
  #19 = Utf8               I
  #20 = Utf8               StackMapTable
  #21 = Class              #17            // "[Ljava/lang/String;"
  #22 = Class              #27            // java/lang/Exception
  #23 = Class              #30            // java/lang/Throwable
  #24 = Utf8               SourceFile
  #25 = Utf8               Demo3_11.java
  #26 = NameAndType        #5:#6          // "<init>":()V
  #27 = Utf8               java/lang/Exception
  #28 = Utf8               ustc/iat/Demo3_11
  #29 = Utf8               java/lang/Object
  #30 = Utf8               java/lang/Throwable
{
  public ustc.iat.Demo3_11();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_11;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
Code:
     stack=1, locals=4, args_size=1
        0: iconst_0
        1: istore_1
        // try块
        2: bipush        10
        4: istore_1
        // try块执行完后，会执行finally    
        5: bipush        30
        7: istore_1
        8: goto          27
       // catch块     
       11: astore_2 // 异常信息放入局部变量表的2号槽位
       12: bipush        20
       14: istore_1
       // catch块执行完后，会执行finally        
       15: bipush        30
       17: istore_1
       18: goto          27
       // 出现异常，但未被 Exception 捕获，会抛出其他异常，这时也需要执行 finally 块中的代码   
       21: astore_3		// 将异常存到局部变量表的3号槽位
       22: bipush        30
       24: istore_1
       25: aload_3	// 取出之前存的异常
       26: athrow  // 抛出异常（操作数栈顶）
       27: return
     Exception table:
        from    to  target type
            2     5    11   Class java/lang/Exception
            2     5    21   any
           11    15    21   any
      LineNumberTable:
        line 10: 0
        line 12: 2
        line 16: 5
        line 17: 8
        line 13: 11
        line 14: 12
        line 16: 15
        line 17: 18
        line 16: 21
        line 17: 25
        line 18: 27
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
           12       3     2     e   Ljava/lang/Exception;
            0      28     0  args   [Ljava/lang/String;
            2      26     1     i   I
      StackMapTable: number_of_entries = 3
        frame_type = 255 /* full_frame */
          offset_delta = 11
          locals = [ class "[Ljava/lang/String;", int ]
          stack = [ class java/lang/Exception ]
        frame_type = 73 /* same_locals_1_stack_item */
          stack = [ class java/lang/Throwable ]
        frame_type = 5 /* same */
}
SourceFile: "Demo3_11.java"
```

其中，Exception table如下：

```bash
      Exception table:
         from    to  target type
             2     5    11   Class java/lang/Exception
             2     5    21   any
            11    15    21   any
```

finally块中的内容被拷贝到try块和catch块中。

- 第一行：try块中有被catch捕获的错误，直接return
- 第二行：try块中有其他没有被catch捕获的错误，直接goto到finally块
- 第三行：catch块中其他没有被catch捕获的错误，直接goto到finally块

**finally面试题**

java源码

```java
public class Demo3_12 {

	public static void main(String[] args) {
		int i = Demo3_12.test();
		// 结果为 20
		System.out.println(i);
	}

	public static int test() {
		int i;
		try {
			i = 10;
			return i;
		} finally {
			i = 20;
			return i;
		}
	}

}
```

字节码

```bash
  Last modified 2024-1-23; size 722 bytes
  MD5 checksum b81a73339b5a63cd9ab7f6440ac1ba88
  Compiled from "Demo3_12.java"
public class ustc.iat.Demo3_12
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#26         // java/lang/Object."<init>":()V
   #2 = Methodref          #5.#27         // ustc/iat/Demo3_12.test:()I
   #3 = Fieldref           #28.#29        // java/lang/System.out:Ljava/io/PrintStream;
   #4 = Methodref          #30.#31        // java/io/PrintStream.println:(I)V
   #5 = Class              #32            // ustc/iat/Demo3_12
   #6 = Class              #33            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lustc/iat/Demo3_12;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               i
  #19 = Utf8               I
  #20 = Utf8               test
  #21 = Utf8               ()I
  #22 = Utf8               StackMapTable
  #23 = Class              #34            // java/lang/Throwable
  #24 = Utf8               SourceFile
  #25 = Utf8               Demo3_12.java
  #26 = NameAndType        #7:#8          // "<init>":()V
  #27 = NameAndType        #20:#21        // test:()I
  #28 = Class              #35            // java/lang/System
  #29 = NameAndType        #36:#37        // out:Ljava/io/PrintStream;
  #30 = Class              #38            // java/io/PrintStream
  #31 = NameAndType        #39:#40        // println:(I)V
  #32 = Utf8               ustc/iat/Demo3_12
  #33 = Utf8               java/lang/Object
  #34 = Utf8               java/lang/Throwable
  #35 = Utf8               java/lang/System
  #36 = Utf8               out
  #37 = Utf8               Ljava/io/PrintStream;
  #38 = Utf8               java/io/PrintStream
  #39 = Utf8               println
  #40 = Utf8               (I)V
{
  public ustc.iat.Demo3_12();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_12;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: invokestatic  #2                  // Method test:()I
         3: istore_1
         4: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;      
         7: iload_1
         8: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
        11: return
      LineNumberTable:
        line 10: 0
        line 12: 4
        line 13: 11
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      12     0  args   [Ljava/lang/String;
            4       8     1     i   I

  public static int test();
    descriptor: ()I
    flags: ACC_PUBLIC, ACC_STATIC
Code:
     stack=1, locals=3, args_size=0
        0: bipush        10
        2: istore_0
        3: iload_0
        4: istore_1  // 暂存返回值
        5: bipush        20
        7: istore_0
        8: iload_0
        9: ireturn	// ireturn 会返回操作数栈顶的整型值 20
       // 如果出现异常，还是会执行finally 块中的内容，没有抛出异常
       10: astore_2
       11: bipush        20
       13: istore_0
       14: iload_0
       15: ireturn	// 这里没有 athrow 了，也就是如果在 finally 块中如果有返回操作的话，且 try 块中出现异常，会吞掉异常！
     Exception table:
        from    to  target type
            0     5    10   any
      LineNumberTable:
        line 18: 0
        line 19: 3
        line 21: 5
        line 22: 8
        line 21: 10
        line 22: 14
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            3       7     0     i   I
           14       2     0     i   I
      StackMapTable: number_of_entries = 1
        frame_type = 74 /* same_locals_1_stack_item */
          stack = [ class java/lang/Throwable ]
}
SourceFile: "Demo3_12.java"
```

- 由于 ﬁnally 中的 ireturn 被插入了所有可能的流程，因此返回结果肯定以ﬁnally的为准
- 至于字节码中第 2 行，似乎没啥用，且留个伏笔，看下个例子
- 跟上例中的 ﬁnally 相比，发现没有 athrow 了，这告诉我们：如果在 ﬁnally 中出现了 return，会吞掉异常，所以不要在finally中进行返回操作

**finally中return会吞掉异常**

java源代码

```java
public static int test() {
    int i;
    try {
        i = 10;
        //  这里应该会抛出异常
        i = i/0;
        return i;
    } finally {
        i = 20;
        return i;
    }
}
```

字节码

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_12.class
  Last modified 2024-1-23; size 730 bytes
  MD5 checksum 5c4029accaeb6bf4ddb28d37d23a6c90
  Compiled from "Demo3_12.java"
public class ustc.iat.Demo3_12
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#26         // java/lang/Object."<init>":()V
   #2 = Methodref          #5.#27         // ustc/iat/Demo3_12.test:()I
   #3 = Fieldref           #28.#29        // java/lang/System.out:Ljava/io/PrintStream;
   #4 = Methodref          #30.#31        // java/io/PrintStream.println:(I)V
   #5 = Class              #32            // ustc/iat/Demo3_12
   #6 = Class              #33            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lustc/iat/Demo3_12;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               i
  #19 = Utf8               I
  #20 = Utf8               test
  #21 = Utf8               ()I
  #22 = Utf8               StackMapTable
  #23 = Class              #34            // java/lang/Throwable
  #24 = Utf8               SourceFile
  #25 = Utf8               Demo3_12.java
  #26 = NameAndType        #7:#8          // "<init>":()V
  #27 = NameAndType        #20:#21        // test:()I
  #28 = Class              #35            // java/lang/System
  #29 = NameAndType        #36:#37        // out:Ljava/io/PrintStream;
  #30 = Class              #38            // java/io/PrintStream
  #31 = NameAndType        #39:#40        // println:(I)V
  #32 = Utf8               ustc/iat/Demo3_12
  #33 = Utf8               java/lang/Object
  #34 = Utf8               java/lang/Throwable
  #35 = Utf8               java/lang/System
  #36 = Utf8               out
  #37 = Utf8               Ljava/io/PrintStream;
  #38 = Utf8               java/io/PrintStream
  #39 = Utf8               println
  #40 = Utf8               (I)V
{
  public ustc.iat.Demo3_12();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_12;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: invokestatic  #2                  // Method test:()I
         3: istore_1
         4: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;      
         7: iload_1
         8: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
        11: return
      LineNumberTable:
        line 10: 0
        line 12: 4
        line 13: 11
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      12     0  args   [Ljava/lang/String;
            4       8     1     i   I

  public static int test();
    descriptor: ()I
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=0
         0: bipush        10
         2: istore_0
         3: iload_0
         4: iconst_0
         5: idiv
         6: istore_0
         7: iload_0
         8: istore_1
         9: bipush        20
        11: istore_0
        12: iload_0
        13: ireturn
        14: astore_2
        15: bipush        20
        17: istore_0
        18: iload_0
        19: ireturn
      Exception table:
         from    to  target type
             0     9    14   any
      LineNumberTable:
        line 18: 0
        line 20: 3
        line 21: 7
        line 23: 9
        line 24: 12
        line 23: 14
        line 24: 18
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            3      11     0     i   I
           18       2     0     i   I
      StackMapTable: number_of_entries = 1
        frame_type = 78 /* same_locals_1_stack_item */
          stack = [ class java/lang/Throwable ]
}
SourceFile: "Demo3_12.java"
```

**try中有return**

java源代码

```java
public static int test() {
    int i = 10;
    try {
        return i;
    } finally {
        i = 20;
    }
}
```

字节码

```bash
  Last modified 2024-1-23; size 719 bytes
  MD5 checksum 7a7c7d6a092a3a9cbd6ec8700f1cc7c9
  Compiled from "Demo3_12.java"
public class ustc.iat.Demo3_12
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#26         // java/lang/Object."<init>":()V
   #2 = Methodref          #5.#27         // ustc/iat/Demo3_12.test:()I
   #3 = Fieldref           #28.#29        // java/lang/System.out:Ljava/io/PrintStream;
   #4 = Methodref          #30.#31        // java/io/PrintStream.println:(I)V
   #5 = Class              #32            // ustc/iat/Demo3_12
   #6 = Class              #33            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lustc/iat/Demo3_12;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               i
  #19 = Utf8               I
  #20 = Utf8               test
  #21 = Utf8               ()I
  #22 = Utf8               StackMapTable
  #23 = Class              #34            // java/lang/Throwable
  #24 = Utf8               SourceFile
  #25 = Utf8               Demo3_12.java
  #26 = NameAndType        #7:#8          // "<init>":()V
  #27 = NameAndType        #20:#21        // test:()I
  #28 = Class              #35            // java/lang/System
  #29 = NameAndType        #36:#37        // out:Ljava/io/PrintStream;
  #30 = Class              #38            // java/io/PrintStream
  #31 = NameAndType        #39:#40        // println:(I)V
  #32 = Utf8               ustc/iat/Demo3_12
  #33 = Utf8               java/lang/Object
  #34 = Utf8               java/lang/Throwable
  #35 = Utf8               java/lang/System
  #36 = Utf8               out
  #37 = Utf8               Ljava/io/PrintStream;
  #38 = Utf8               java/io/PrintStream
  #39 = Utf8               println
  #40 = Utf8               (I)V
{
  public ustc.iat.Demo3_12();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_12;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: invokestatic  #2                  // Method test:()I
         3: istore_1
         4: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;      
         7: iload_1
         8: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
        11: return
      LineNumberTable:
        line 10: 0
        line 12: 4
        line 13: 11
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      12     0  args   [Ljava/lang/String;
            4       8     1     i   I

  public static int test();
    descriptor: ()I
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
         stack=1, locals=3, args_size=0
            0: bipush        10
            2: istore_0 // 赋值给i 10
            3: iload_0	// 加载到操作数栈顶
            4: istore_1 // 加载到局部变量表的1号位置
            5: bipush        20
            7: istore_0 // 赋值给i 20
            8: iload_1 // 加载局部变量表1号位置的数10到操作数栈
            9: ireturn // 返回操作数栈顶元素 10
           10: astore_2 // 将异常放到局部变量表的2号位置
           11: bipush        20
           13: istore_0
           14: aload_2 // 加载异常
           15: athrow // 抛出异常
         Exception table:
            from    to  target type
                3     5    10   any
      LineNumberTable:
        line 16: 0
        line 18: 3
        line 20: 5
        line 18: 8
        line 20: 10
        line 21: 14
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            3      13     0     i   I
      StackMapTable: number_of_entries = 1
        frame_type = 255 /* full_frame */
          offset_delta = 10
          locals = [ int ]
          stack = [ class java/lang/Throwable ]
}
SourceFile: "Demo3_12.java"
```

**synchronized**

java源代码

```java
public class Demo3_13 {

	public static void main(String[] args) {
		Object lock = new Object();
		synchronized (lock) {
			System.out.println("ok");
		}
	}

}
```

字节码

```bash
Classfile /D:/ideaproj/jvm/target/classes/ustc/iat/Demo3_13.class
  Last modified 2024-1-23; size 701 bytes
  MD5 checksum b9afcd133d3b85ca2574ff707558020c
  Compiled from "Demo3_13.java"
public class ustc.iat.Demo3_13
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #2.#26         // java/lang/Object."<init>":()V
   #2 = Class              #27            // java/lang/Object
   #3 = Fieldref           #28.#29        // java/lang/System.out:Ljava/io/PrintStream;
   #4 = String             #30            // ok
   #5 = Methodref          #31.#32        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #6 = Class              #33            // ustc/iat/Demo3_13
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lustc/iat/Demo3_13;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               lock
  #19 = Utf8               Ljava/lang/Object;
  #20 = Utf8               StackMapTable
  #21 = Class              #17            // "[Ljava/lang/String;"
  #22 = Class              #27            // java/lang/Object
  #23 = Class              #34            // java/lang/Throwable
  #24 = Utf8               SourceFile
  #25 = Utf8               Demo3_13.java
  #26 = NameAndType        #7:#8          // "<init>":()V
  #27 = Utf8               java/lang/Object
  #28 = Class              #35            // java/lang/System
  #29 = NameAndType        #36:#37        // out:Ljava/io/PrintStream;
  #30 = Utf8               ok
  #31 = Class              #38            // java/io/PrintStream
  #32 = NameAndType        #39:#40        // println:(Ljava/lang/String;)V
  #33 = Utf8               ustc/iat/Demo3_13
  #34 = Utf8               java/lang/Throwable
  #35 = Utf8               java/lang/System
  #36 = Utf8               out
  #37 = Utf8               Ljava/io/PrintStream;
  #38 = Utf8               java/io/PrintStream
  #39 = Utf8               println
  #40 = Utf8               (Ljava/lang/String;)V
{
  public ustc.iat.Demo3_13();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lustc/iat/Demo3_13;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=1
         0: new           #2                  // class java/lang/Object
         3: dup 				// 复制一份栈顶，然后压入栈中。用于函数消耗
         4: invokespecial #1                  // Method java/lang/Object."<init>":()V
         7: astore_1 // 将栈顶的对象地址方法 局部变量表中 1 中
         8: aload_1 // 加载到操作数栈
         9: dup // 复制一份，放到操作数栈，用于加锁时消耗
        10: astore_2 // 将操作数栈顶元素弹出，暂存到局部变量表的 2 号槽位。这时操作数栈中有一份对象的引用
        11: monitorenter // 加锁
        12: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
        15: ldc           #4                  // String ok
        17: invokevirtual #5                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        20: aload_2 // 加载对象到栈顶
        21: monitorexit // 释放锁
        22: goto          30
        // 异常情况的解决方案 :释放锁！否则将永远被加锁
        25: astore_3   // 存异常
        26: aload_2
        27: monitorexit
        28: aload_3  // 加载异常
        29: athrow  // 返回异常
        30: return
        // 异常表！
      Exception table:
         from    to  target type
            12    22    25   any
            25    28    25   any
        line 10: 0
        line 11: 8
        line 12: 12
        line 13: 20
        line 14: 30
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      31     0  args   [Ljava/lang/String;
            8      23     1  lock   Ljava/lang/Object;
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 25
          locals = [ class "[Ljava/lang/String;", class java/lang/Object, class java/lang/Object ]     
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
}
SourceFile: "Demo3_13.java"
```

## 编译期处理

### 默认构造器

如果自己没有定义构造器，则系统会提供一个无参构造器

```java
public class Candy1 {

}
```

经过编译期优化后

```java
public class Candy1 {
   // 这个无参构造器是java编译器帮我们加上的
   public Candy1() {
      // 即调用父类 Object 的无参构造方法，即调用 java/lang/Object." <init>":()V
      super();
   }
}
```

### 自动拆装箱（jdk5之后）

```java
public class Candy2 {
   public static void main(String[] args) {
      Integer x = 1;
      int y = x;
   }
}
```

jdk5以后无需手动拆装箱 `Integer x = Integer.valueOf(1);` 和 `int y = x.intValue();`

### 泛型擦除

泛型也是在 JDK 5 开始加入的特性，但 java 在编译泛型代码后会执行泛型擦除的动作，即泛型信息在编译为字节码之后就丢失了，实际的类型都当做了 Object 类型来处理：

```java
public class Candy3 {
   public static void main(String[] args) {
      List<Integer> list = new ArrayList<>();
      list.add(10);
      Integer x = list.get(0);
   }
}
```

对应字节码

```java
Code:
    stack=2, locals=3, args_size=1
       0: new           #2                  // class java/util/ArrayList
       3: dup
       4: invokespecial #3                  // Method java/util/ArrayList."<init>":()V
       7: astore_1
       8: aload_1
       9: bipush        10
      11: invokestatic  #4                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      // 这里进行了泛型擦除，实际调用的是add(Objcet o)
      14: invokeinterface #5,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z

      19: pop
      20: aload_1
      21: iconst_0
      // 这里也进行了泛型擦除，实际调用的是get(Object o)   
      22: invokeinterface #6,  2            // InterfaceMethod java/util/List.get:(I)Ljava/lang/Object;
// 这里进行了类型转换，将 Object 转换成了 Integer
      27: checkcast     #7                  // class java/lang/Integer
      30: astore_2
      31: return
```

所以调用 get 函数取值时，有一个类型转换的操作。

```java
Integer x = (Integer) list.get(0);
```

如果要将返回结果赋值给一个 int 类型的变量，则还有自动拆箱的操作

```java
int x = (Integer) list.get(0).intValue();
```

通过反射可以获得参数类型和泛型类型，java代码如下：

```java
public class Demo3_14 {

	public static void main(String[] args) throws NoSuchMethodException {
		Method method = Demo3_14.class.getMethod("test", List.class, Map.class);
		Type[] types = method.getGenericParameterTypes();
		for(Type type : types) {
			if(type instanceof ParameterizedType) {
				ParameterizedType parameterizedType = (ParameterizedType) type;
				System.out.println("原始类型 : " + parameterizedType.getRawType());
				Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
				for (int i = 0; i < actualTypeArguments.length; i++) {
					System.out.println("泛型类型[" + i + "] : " + actualTypeArguments[i]);
				}
			}
		}
	}

	public Set<Integer> test(List<String> list, Map<Integer, Object> map) {
		return null;
	}

}
```

### 可变参数（jdk5新特性）

可变参数也是 JDK 5 开始加入的新特性

java源代码

```java
public class Candy4 {
   public static void foo(String... args) {
      // 将 args 赋值给 arr ，可以看出 String... 实际就是 String[]  
      String[] arr = args;
      System.out.println(arr.length);
   }

   public static void main(String[] args) {
      foo("hello", "world");
   }
}
```

在编译期，上述代码被优化为

```java
public class Candy4 {
   public Candy4 {}
   public static void foo(String[] args) {
      String[] arr = args;
      System.out.println(arr.length);
   }

   public static void main(String[] args) {
      foo(new String[]{"hello", "world"});
   }
}
```

注意，如果调用的是 foo() ，即未传递参数时，等价代码为 foo(new String[]{}) ，创建了一个空数组，而不是直接传递的 null。

### 增强for循环（jdk5引入的新特性）

在数组中：

java源代码

```java
public class Candy5 {
	public static void main(String[] args) {
        // 数组赋初值的简化写法也是一种语法糖。
		int[] arr = {1, 2, 3, 4, 5};
		for(int x : arr) {
			System.out.println(x);
		}
	}
}
```

在编译期，上述代码被优化为

```java
public class Candy5 {
    public Candy5() {}

	public static void main(String[] args) {
		int[] arr = new int[]{1, 2, 3, 4, 5};
		for(int i = 0; i < arr.length; ++i) {
			int x = arr[i];
			System.out.println(x);
		}
	}
}
```

在集合中：

java源代码

```java
public class Candy5 {
   public static void main(String[] args) {
      List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
      for (Integer x : list) {
         System.out.println(x);
      }
   }
}
```

在编译期，上述代码被优化为

```java
public class Candy5 {
    public Candy5(){}
    
   public static void main(String[] args) {
      List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
      // 获得该集合的迭代器
      Iterator<Integer> iterator = list.iterator();
      while(iterator.hasNext()) {
         Integer x = iterator.next();
         System.out.println(x);
      }
   }
}
```

### switch字符串（jdk7新特性）

从 JDK 7 开始，switch 可以作用于字符串和枚举类，这个功能其实也是语法糖。

java源代码

```java
public class Cnady6 {
   public static void main(String[] args) {
      String str = "hello";
      switch (str) {
         case "hello" :
            System.out.println("h");
            break;
         case "world" :
            System.out.println("w");
            break;
         default:
            break;
      }
   }
}
```

在编译期，上述代码被优化为

```java
public class Candy6 {
   public Candy6() {
      
   }
   public static void main(String[] args) {
      String str = "hello";
      int x = -1;
      // 通过字符串的 hashCode + value 来判断是否匹配
      switch (str.hashCode()) {
         // hello 的 hashCode
         case 99162322 :
            // 再次比较，因为字符串的 hashCode 有可能相等
            if(str.equals("hello")) {
               x = 0;
            }
            break;
         // world 的 hashCode
         case 11331880 :
            if(str.equals("world")) {
               x = 1;
            }
            break;
         default:
            break;
      }

      // 用第二个 switch 在进行输出判断
      switch (x) {
         case 0:
            System.out.println("h");
            break;
         case 1:
            System.out.println("w");
            break;
         default:
            break;
      }
   }
}
```

过程说明：

- 在编译期间，单个的 switch 被分为了两个

  - 第一个用来匹配字符串，并给byte类型的变量 x 赋值
    - 字符串的匹配用到了字符串的 hashCode ，还用到了 equals 方法
    - 使用 hashCode 是为了提高比较效率，使用 equals 是防止有 hashCode 冲突（如 BM 和 C .）

  - 第二个用来根据byte类型的变量 x 的值来决定输出语句

### switch枚举（jdk7新特性）

java源码

```java
enum SEX {
   MALE, FEMALE;
}
public class Candy7 {
   public static void main(String[] args) {
      SEX sex = SEX.MALE;
      switch (sex) {
         case MALE:
            System.out.println("man");
            break;
         case FEMALE:
            System.out.println("woman");
            break;
         default:
            break;
      }
   }
}
```

在编译期，上述代码被优化为

```java
enum SEX {
   MALE, FEMALE;
}

public class Candy7 {
   /**     
    * 定义一个合成类（仅 jvm 使用，对我们不可见）     
    * 用来映射枚举的 ordinal 与数组元素的关系     
    * 枚举的 ordinal 表示枚举对象的序号，从 0 开始     
    * 即 MALE 的 ordinal()=0，FEMALE 的 ordinal()=1     
    */ 
   static class $MAP {
      // 数组大小即为枚举元素个数，里面存放了 case 用于比较的数字
      static int[] map = new int[2];
      static {
         // ordinal 即枚举元素对应所在的位置，MALE 为 0 ，FEMALE 为 1
         map[SEX.MALE.ordinal()] = 1;
         map[SEX.FEMALE.ordinal()] = 2;
      }
   }

   public static void main(String[] args) {
      SEX sex = SEX.MALE;
      // 将对应位置枚举元素的值赋给 x ，用于 case 操作
      int x = $MAP.map[sex.ordinal()];
      switch (x) {
         case 1:
            System.out.println("man");
            break;
         case 2:
            System.out.println("woman");
            break;
         default:
            break;
      }
   }
}
```

编译器在编译期间声明了一个内部类，内部类中有一个数组，数组的大小即为枚举类中元素的个数，枚举类中每一个元素有orginal()序号，序号从0开始，正好作为内部类中数组的下标，然后数组的值从1开始，在switch中，直接switch从1开始的int，输入的枚举类的元素sex，经过变换得到 `int x = $MAP.map[sex.ordinal()]`，然后switch(x)即可。

### 枚举类（jdk7新增特性）

JDK 7 新增了枚举类

```java
enum SEX {
   MALE, FEMALE;
}
```

在编译期，上述代码被优化为

```java
public final class Sex extends Enum<Sex> {   
   // 对应枚举类中的元素
   public static final Sex MALE;    
   public static final Sex FEMALE;    
   private static final Sex[] $VALUES;
   
    static {       
    	// 调用构造函数，传入枚举元素的值及 ordinal
    	MALE = new Sex("MALE", 0);  		// 0是ordinal
        FEMALE = new Sex("FEMALE", 1);   
        $VALUES = new Sex[]{MALE, FEMALE}; 
   }
 	
   // 调用父类中的方法
    private Sex(String name, int ordinal) {     
        super(name, ordinal);    
    }
   
    public static Sex[] values() {  
        return $VALUES.clone();  
    }
    public static Sex valueOf(String name) { 
        return Enum.valueOf(Sex.class, name);  
    } 
   
}
```

### try - with - resources

JDK 7 开始新增了对需要关闭的资源处理的特殊语法，try-with-resources

```java
try(资源变量 = 创建资源对象) {
	
} catch() {

}
```

其中资源对象需要实现 AutoCloseable 接口，例如 InputStream 、 OutputStream 、 Connection 、 Statement 、 ResultSet 等接口都实现了 AutoCloseable ，使用 try-with- resources 可以不用写 finally 语句块，编译器会帮助生成关闭资源代码

例子：

```java
public class Candy9 { 
	public static void main(String[] args) {
		try(InputStream is = new FileInputStream("d:\\1.txt")){	
			System.out.println(is); 
		} catch (IOException e) { 
			e.printStackTrace(); 
		} 
	} 
}
```

在编译期，上述代码被优化为

```java
public class Candy9 { 
    
    public Candy9() { }
   
    public static void main(String[] args) { 
        try {
            InputStream is = new FileInputStream("d:\\1.txt");
            Throwable t = null; 
            try {
                System.out.println(is); 
            } catch (Throwable e1) { 
                // t 是我们代码出现的异常 
                t = e1; 
                throw e1; 
            } finally {
                // 判断了资源不为空 
                if (is != null) { 
                    // 如果我们代码有异常
                    if (t != null) { 
                        try {
                            is.close();    // 关闭资源时还可能出现异常
                        } catch (Throwable e2) { 
                            // 如果 close 出现异常，作为被压制异常添加
                            // 我们希望把申请资源时的异常和关闭资源时
                            t.addSuppressed(e2); 
                        } 
                    } else { 
                        // 如果我们代码没有异常，close 出现的异常就是最后 catch 块中的 e 
                        is.close(); 
                    } 
                } 
            } 
        } catch (IOException e) {
            e.printStackTrace(); 
        } 
    }
}
```

捕获可能的两种异常

1. 在打开输入流的 `try` 块中可能抛出的异常，这里是 `IOException`。该异常被捕获并重新抛出，然后在外层的 `catch` 块中处理。
2. 在关闭输入流的 `finally` 块中可能抛出的异常。无论资源关闭时是否发生了异常，都会进入 `finally` 块。在 `finally` 块中，首先判断输入流 `is` 是否为 `null`，然后根据代码块中是否发生了异常进行相应的处理。如果代码块中有异常，将关闭资源时的异常作为被压制异常添加到之前捕获的异常 `t` 中。如果代码块中没有异常，直接关闭资源。

`addSuppressed(Throwable e)` 的目的：防止异常信息的丢失。见下一个例子

```java
public class Test6 { 
	public static void main(String[] args) { 
		try (MyResource resource = new MyResource()) { 
			int i = 1/0; 
		} catch (Exception e) { 
			e.printStackTrace(); 
		} 
	} 
}
class MyResource implements AutoCloseable { 
	public void close() throws Exception { 
		throw new Exception("close 异常"); 
	} 
}
```

运行结果

```bash
java.lang.ArithmeticException: / by zero
	at ustc.iat.Demo3_15.main(Demo3_15.java:10)
	Suppressed: java.lang.Exception: close 异常
		at ustc.iat.MyResource.close(Demo3_15.java:18)
		at ustc.iat.Demo3_15.main(Demo3_15.java:11)
```

异常信息没有丢失

### 方法重写时的桥接方法

方法重写时对返回值分两种情况：

- 父子类的返回值完全一致
- 子类返回值可以是父类返回值的子类

对于第二种情况，编译器在编译期间进行了处理

java源代码

```java
class A { 
	public Number m() { 
		return 1; 
	} 
}
class B extends A { 
	@Override 
	// 子类 m 方法的返回值是 Integer 是父类 m 方法返回值 Number 的子类 	
	public Integer m() { 
		return 2; 
	} 
}
```

在编译期，上述代码被优化为

```java
class B extends A { 
	public Integer m() { 
		return 2; 
	}
	// 此方法才是真正重写了父类 public Number m() 方法 
	public synthetic bridge Number m() { 
		// 调用 public Integer m() 
		return m(); 
	} 
}
```

可以通过反射来验证上面的代码

```java
public class Demo3_17 {

	public static void main(String[] args) {
		for(Method m : B.class.getDeclaredMethods()) {
			System.out.println(m);
		}
	}

}

class A {
	public Number m() {
		return 1;
	}
}
class B extends A {
	@Override
	// 子类 m 方法的返回值是 Integer 是父类 m 方法返回值 Number 的子类
	public Integer m() {
		return 2;
	}
}
```

运行结果为

```bash
public java.lang.Integer cn.ali.jvm.test.B.m()
public java.lang.Number cn.ali.jvm.test.B.m()
```

### 匿名内部类

java源代码

```java
public class Candy10 {
   public static void main(String[] args) {
      Runnable runnable = new Runnable() {
         @Override
         public void run() {
            System.out.println("running...");
         }
      };
   }
}
```

在编译期，上述代码被优化为

```java
public class Candy10 {
   public static void main(String[] args) {
      // 用额外创建的类来创建匿名内部类对象
      Runnable runnable = new Candy10$1();
   }
}

// 创建了一个额外的类，实现了 Runnable 接口
final class Candy10$1 implements Runnable {
   public Candy10$1() {}

   @Override
   public void run() {
      System.out.println("running...");
   }
}
```

内部类转变为一个新的类，同时实现了对应接口。

如果这个匿名内部类使用了局部变量，那么java源代码为：（此处局部变量应该是final修饰的常量，在编译期间就应该被确定）

```java
public class Candy11 { 
	public static void test(final int x) { 
		Runnable runnable = new Runnable() { 
			@Override 
			public void run() { 	
				System.out.println("ok:" + x); 
			} 
		}; 
	} 
}
```

在编译期，上述代码被优化为

```java
// 额外生成的类 
final class Candy11$1 implements Runnable { 
	int val$x; 
	Candy11$1(int x) { 
		this.val$x = x; 
	}
	public void run() { 
		System.out.println("ok:" + this.val$x); 
	} 
}

public class Candy11 { 
	public static void test(final int x) { 
		Runnable runnable = new Candy11$1(x); 
	} 
}
```

内部类转变为一个新的类，同时实现了对应接口，并且持有一个成员变量。

## 类加载阶段

### 加载

将类的字节码载入方法区（1.8前为永久代，在jvm中；1.8后为元空间，在本地内存中）中，内部采用 C++ 的 instanceKlass 描述 java 类，它的重要 ﬁeld 有：

- java_mirror 即 java 的类镜像，例如对 String 来说，它的镜像类就是 String.class，作用是把 klass 暴露给 java 使用
- super 即父类
- ﬁelds 即成员变量
- methods 即方法
- constants 即常量池
- class_loader 即类加载器
- vtable 虚方法表

- itable 接口方法

注意事项：

- instanceKlass 和 Person.class 持有彼此的引用，instanceKlass 在方法区（1.8前为永久代，在jvm中；1.8后为元空间，在本地内存中），_java_mirror 在堆内存

- 如果这个类还有父类没有加载，先加载父类
- 加载和链接可能是交替运行的
- 类的对象在对象头中保存了*.class的地址。让对象可以通过其找到方法区中的instanceKlass，从而获取类的各种信息

![image-20240123210449978](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240123210449978.png)

### 连接

主要分为以下三个阶段

**验证**

验证类是否符合 JVM规范，安全性检查，用 UE 等支持二进制的编辑器修改 HelloWorld.class 的魔数，在控制台运行

**准备**

为 static 变量分配空间，设置默认值，不对其赋初值

- static 变量在 JDK 7 之前存储于 instanceKlass 末尾，从 JDK 7 开始，存储于 _java_mirror 末尾
- static 变量分配空间和赋值是两个步骤，分配空间在准备阶段完成，赋值在初始化阶段完成
- 如果 static 变量是 final 的基本类型，以及字符串常量，那么编译阶段值就确定了，赋值在准备阶段完成。例如：`static final int c = 20;` 和 `static final String d = "hello";` 是在准备阶段的。
- 如果 static 变量是 final 的，但属于引用类型，则这里需要new关键字，那么赋值也会在初始化阶段完成将常量池中的符号引用解析为直接引用。例如：`static final Object e = new Object();` 是在初始化阶段的。

**解析**

将常量池中的符号引用解析为直接引用

未被解析的案例：

```java
public class Demo3_18 {

	public static void main(String[] args) throws ClassNotFoundException, IOException {
		// loadClass不会触发解析
		ClassLoader classLoader = Demo3_18.class.getClassLoader();
		Class<?> clazz = classLoader.loadClass("ustc.iat.C");
		System.in.read();
	}

}

class C {
	D d = new D();
}

class D {
}
```

使用HSDB工具监测到的结果：

![image-20240123221325916](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240123221325916.png)

被解析的案例：

```java
public class Demo3_18 {

	public static void main(String[] args) throws ClassNotFoundException, IOException {
		// new C();会发生解析
		new C();
		System.in.read();
	}

}

class C {
	D d = new D();
}

class D {
}
```

使用HSDB工具监测到的结果：

![image-20240123221552973](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240123221552973.png)

### 初始化

\<cinit>()v 方法

初始化即调用 \<cinit>()V ，虚拟机会保证这个类的构造方法的线程安全

##### 发生的时机

概括得说，类初始化是【懒惰的】

- main 方法所在的类，总会被首先初始化
- 首次访问这个类的静态变量或静态方法时
- 子类初始化，如果父类还没初始化，会引发
- 子类访问父类的静态变量，只会触发父类的初始化
- Class.forName
- new 会导致初始化

不会导致类初始化的情况

- 访问类的 static final 静态常量（基本类型和字符串）不会触发初始化
- 类对象.class 不会触发初始化
- 创建该类的数组不会触发初始化

java源代码

```java
public class Load1 {
    static {
        System.out.println("main init");
    }
    public static void main(String[] args) throws ClassNotFoundException {
        // 1. 静态常量（基本类型和字符串）不会触发初始化
//         System.out.println(B.b);
        // 2. 类对象.class 不会触发初始化
//         System.out.println(B.class);
        // 3. 创建该类的数组不会触发初始化
//         System.out.println(new B[0]);
        // 4. 不会初始化类 B，但会加载 B、A
//         ClassLoader cl = Thread.currentThread().getContextClassLoader();
//         cl.loadClass("cn.ali.jvm.test.classload.B");
        // 5. 不会初始化类 B，但会加载 B、A
//         ClassLoader c2 = Thread.currentThread().getContextClassLoader();
//         Class.forName("cn.ali.jvm.test.classload.B", false, c2);


        // 1. 首次访问这个类的静态变量或静态方法时
//         System.out.println(A.a);
        // 2. 子类初始化，如果父类还没初始化，会引发
//         System.out.println(B.c);
        // 3. 子类访问父类静态变量，只触发父类初始化
//         System.out.println(B.a);
        // 4. 会初始化类 B，并先初始化类 A
//         Class.forName("cn.ali.jvm.test.classload.B");
    }

}


class A {
    static int a = 0;
    static {
        System.out.println("a init");
    }
}
class B extends A {
    final static double b = 5.0;
    static boolean c = false;
    static {
        System.out.println("b init");
    }
}
```

**练习**

java源代码

```java
public class Load2 {

    public static void main(String[] args) {
        System.out.println(E.a);
        System.out.println(E.b);
        // 会导致 E 类初始化，因为 Integer 是包装类
        System.out.println(E.c);
    }
}

class E {
    public static final int a = 10;
    public static final String b = "hello";
    public static final Integer c = 20;

    static {
        System.out.println("E cinit");
    }
}
```

其中，以下两个只访问了静态常量（基本类型和字符串），所以不会触发初始化

```java
System.out.println(E.a);
System.out.println(E.b);
```

但是，对于静态引用对象，会触发解析和初始化

```java
System.out.println(E.c);
```

## 类加载器

以JDK 8为例

| 名称                                      | 加载的类              | 说明                        |
| ----------------------------------------- | --------------------- | --------------------------- |
| Bootstrap ClassLoader（启动类加载器）     | JAVA_HOME/jre/lib     | 无法直接访问                |
| Extension ClassLoader(拓展类加载器)       | JAVA_HOME/jre/lib/ext | 上级为Bootstrap，显示为null |
| Application ClassLoader(应用程序类加载器) | classpath             | 上级为Extension             |
| 自定义类加载器                            | 自定义                | 上级为Application           |

启动类加载器是由C++编写的，所以无法获取（null），并且扩展类加载器的上级也无法获取（null）

### 启动类加载器

可通过在控制台输入指令，使得类被启动类加器加载

java源代码

```java
public class Demo4_1 {

	public static void main(String[] args) throws ClassNotFoundException {
		Class<?> aClass = Class.forName("ustc.iat.F");
		System.out.println(aClass.getClassLoader());
	}

}

class F {
	static {
		System.out.println("bootstrap F init");
	}
}
```

指令

```bash
java -Xbootclasspath/a:. ustc.iat.Demo4_1
输出：null表示获取不到类加载器，所以是启动类加载器
```

`-Xbootclasspath/a:.` 表示设置booclasspath，其中`/a:.` 表示将当前目录追加到bootclasspath

### 拓展类加载器

负责加载 `JAVA_HOME/jre/lib/ext` 下的类文件

java源代码

```java
public class Demo4_F {

	static {
		System.out.println("classpath F init.");
	}

	public static void main(String[] args) {

	}

}
```

然后将其字节码文件打包成jar包

```bash
PS D:\ideaproj\jvm\target\classes> jar -cvf my.jar ustc\iat\Demo4_F.class
```

再将jar包拷贝到 `JAVA_HOME/jre/lib/ext` 

![image-20240124100924294](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240124100924294.png)

再次运行代码

```java
public class Demo4_1 {

	public static void main(String[] args) throws ClassNotFoundException {
		Class<?> aClass = Class.forName("ustc.iat.Demo4_F");
		System.out.println(aClass.getClassLoader());
	}

}
```

运行结果

```bash
classpath F init.
sun.misc.Launcher$ExtClassLoader@135fbaa4
```

此时不会加载当前类路径下的Demo4_F，而是使用扩展类加载器加载 `JAVA_HOME/jre/lib/ext` 下的类，这符合双亲委派机制，当应用程序类加载器要加载类时，先会询问自己的上级（扩展类加载器）是否加载过加载这个类，如果加载过，则直接用，如果没加载过，则继续询问扩展类加载器的上级（启动类加载器），如果启动类加载器已经加载过了，那么直接用，如果没加载过，则尝试加载，若加载成功，则直接加载；如果加载不了，那么让扩展类加载器继续尝试加载，如果扩展类加载器尝试加载如果加载成功，则直接加载；如果加载不了，那么让应用程序类加载器尝试加载。

### 双亲委派模式

双亲委派模式，即调用类加载器ClassLoader 的 loadClass 方法时，查找类的规则。
loadClass源码

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // 首先查找该类是否已经被该类加载器加载过了
        Class<?> c = findLoadedClass(name);
        // 如果没有被加载过
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                // 看是否被它的上级加载器加载过了 Extension 的上级是Bootstarp，但它显示为null
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    // 看是否被启动类加载器加载过
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
                //捕获异常，但不做任何处理
            }

            if (c == null) {
                // 如果还是没有找到，先让拓展类加载器调用 findClass 方法去找到该类，如果还是没找到，就抛出异常
                // 然后让应用类加载器去找 classpath 下找该类
                long t1 = System.nanoTime();
                c = findClass(name);

                // 记录时间
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

### 线程上下文类加载器

案例1：JDBC加载Driver驱动。

`DriverManager` 是 JDBC（Java Database Connectivity） API 中的一个类，用于管理一组数据库驱动程序。`DriverManager` 的主要作用是协调不同数据库厂商提供的 JDBC 驱动程序，并允许应用程序通过该驱动程序与不同的数据库建立连接。

`Driver` 是 JDBC（Java Database Connectivity） API 中的一个接口。这个接口定义了一种标准的方式，使得不同数据库厂商能够提供符合 JDBC 规范的数据库驱动程序。数据库驱动程序充当了Java应用程序与特定数据库之间的桥梁，负责处理与数据库的通信。

启动类加载器加载DriverManager，所以会到 `JAVA_HOME/jre/lib` 下搜索类，但是这个路径下显然没有 `mysql-connector-java-5.1.47.jar` 包 。DriverManager代码里调用了线程上下文类加载器，这个加载器默认就是使用应用程序类加载器加载类，通过应用程序类加载器加载jdbc驱动，违反了双亲委派机制。

案例2：SPI（Service Provider Interface）是Java中一种服务提供者接口的机制。

在jar包中的 `META-INF/services` 包下，以接口全限定名为文件，文件的内容是实现类的名称

![image-20240124104955882](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240124104955882.png)

因为 SPI的接口由java核心库提供，所以只能由 BootStrap加载，而SPI实现是通过第三方库，所以只能由应用程序类加载器加载。

很多框架使用`面向接口编程 + 解耦`的思想：

- JDBC
- Servlet初始化器
- Spring容器
- Dubbo

![image-20240124110723533](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240124110723533.png)

线程上下文加载器：在线程启动时，JVM会将应用程序类加载器赋值给线程上下文类加载器。

`Thread.currentThread().getContextClassLoader()` 即为线程上下文类加载器（应用程序类加载器）

### 自定义类加载器

**使用场景**

- 想加载非 classpath 随意路径中的类文件
- 通过接口来使用实现，希望解耦时，常用在框架设计
- 这些类希望予以隔离，不同应用的同名类都可以加载，不冲突，常见于 tomcat 容器

**步骤**

- 继承 ClassLoader 父类
- 要遵从双亲委派机制，重写 ﬁndClass 方法，注意，此处不是重写 loadClass 方法，否则不会走双亲委派机制
- 读取类文件的字节码
- 调用父类的 deﬁneClass 方法来加载类
- 使用者调用该类加载器的 loadClass 方法

java代码实现

```java
public class Load {

    public static void main(String[] args) throws ClassNotFoundException, InstantiationException, IllegalAccessException {
       MyClassLoader myClassLoader1 = new MyClassLoader();
       Class<?> c1 = myClassLoader1.loadClass("Demo3_2");
       MyClassLoader myClassLoader2 = new MyClassLoader();
       Class<?> c2 = myClassLoader2.loadClass("Demo3_2");
       // 包名，类名，类加载器完全相同，则这两个类才是同一个
       System.out.println(c1 == c2);
       Object object = c1.newInstance();
       System.out.println(object);

    }

}

class MyClassLoader extends ClassLoader {

    // 重写findClass方法
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
       String path = "D:\\ideaproj\\jvm\\target\\classes\\ustc\\iat\\" + name + ".class";
       System.out.println(path);
       try {
          ByteArrayOutputStream bos = new ByteArrayOutputStream();
          Files.copy(Paths.get(path), bos);
          // 类文件的字节数组
          byte[] bytes = bos.toByteArray();
          // deﬁneClass来加载类
          // 这里需要全限定字
          return defineClass("ustc.iat." + name, bytes, 0, bytes.length);
       } catch (IOException e) {
          e.printStackTrace();
          throw new ClassNotFoundException("类文件不存在", e);
       }
    }

}
```

**破坏双亲委派模式**

- 双亲委派模型的第一次“被破坏”其实发生在双亲委派模型出现之前——即JDK1.2面世以前的“远古”时代
  - 建议用户重写findClass()方法，在类加载器中的loadClass()方法中也会调用该方法
- 双亲委派模型的第二次“被破坏”是由这个模型自身的缺陷导致的
  - 如果有基础类型又要调用回用户的代码，此时也会破坏双亲委派模式
- 双亲委派模型的第三次“被破坏”是由于用户对程序动态性的追求而导致的
  - 这里所说的“动态性”指的是一些非常“热”门的名词：代码热替换（Hot Swap）、模块热部署（Hot Deployment）等

## 运行期优化

JVM 将执行状态分成了 5 个层次：

- 0层：解释执行，用解释器将字节码翻译为机器码
- 1层：使用 C1 即时编译器编译执行（不带 proﬁling）
- 2层：使用 C1 即时编译器编译执行（带基本的profiling）
- 3层：使用 C1 即时编译器编译执行（带完全的profiling）
- 4层：使用 C2 即时编译器编译执行

proﬁling 是指在运行过程中收集一些程序执行状态的数据，例如【方法的调用次数】，【循环的 回边次数】等

即时编译器（JIT）与解释器的区别

- 解释器
  - 将字节码解释为机器码，下次即使遇到相同的字节码，仍会执行重复的解释
  - 是将字节码解释为针对所有平台都通用的机器码

- 即时编译器
  - 将一些字节码编译为机器码，并存入 Code Cache，下次遇到相同的代码，直接执行，无需再编译
  - 根据平台类型，生成平台特定的机器码

对于大部分的不常用的代码，我们无需耗费时间将其编译成机器码，而是采取解释执行的方式运行；另一方面，对于仅占据小部分的热点代码，我们则可以将其编译成机器码，以达到理想的运行速度。 执行效率上简单比较一下 Interpreter < C1 < C2，总的目标是发现热点代码（hotspot名称的由 来），并优化这些热点代码。

### 逃逸分析

逃逸分析（Escape Analysis）简单来讲就是，Java Hotspot 虚拟机可以分析新创建对象的使用范围，并决定是否在 Java 堆上分配内存的一项技术，逃逸分析默认开启，从jdk6开始支持逃逸分析

关闭逃逸分析：`-XX:+DoEscapeAnalysis`

```java
public class Demo5_1 {

	// 关闭逃逸分析
	// -XX:-DoEscapeAnalysis
	public static void main(String[] args) {
		for (int i = 0; i < 300; i++) {
			long start = System.nanoTime();
			for (int j = 0; j < 1000; j++) {
				new Object();
			}
			long end = System.nanoTime();
			System.out.println((i + 1) + " : " + (end - start));

		}
	}

}
```

结果

```bash
1 : 41400
2 : 25700
3 : 23300
4 : 22700
5 : 23500
6 : 22800
7 : 23000
8 : 23500
9 : 22000
10 : 22200

...

291 : 600
292 : 600
293 : 600
294 : 600
295 : 700
296 : 700
297 : 600
298 : 700
299 : 900
300 : 600
```

### 方法内联

如果发现方法是热点方法，并且方法长度不太长，会进行方法内敛，所谓的方法内联就是将方法内代码拷贝到调用者的位置。

方法内联前：

```java
public class Demo5_2 {

	public static void main(String[] args) {
		int x = 0;
		for (int i = 0; i < 1000; i++) {
			long start = System.nanoTime();
			for (int j = 0; j < 1000; j++) {
				x = square(9);
			}
			long end = System.nanoTime();
			System.out.println((i + 1) + " : " + (end  - start));
		}
	}

	private static int square(int i) {
		return i * i;
	}

}
```

方法内敛后：

```java
public static void main(String[] args) {
    int x = 0;
    for (int i = 0; i < 1000; i++) {
        long start = System.nanoTime();
        for (int j = 0; j < 1000; j++) {
            x = 9 * 9;
        }
        long end = System.nanoTime();
        System.out.println((i + 1) + " : " + (end  - start));
    }
}
```

运行结果

```bash
1 : 44400
2 : 36500
3 : 18300
4 : 20200
5 : 19300
6 : 19600
7 : 18200
8 : 18800
9 : 36900
10 : 15900

...

991 : 0
992 : 100
993 : 0
994 : 0
995 : 100
996 : 100
997 : 100
998 : 100
999 : 0
1000 : 0
```

打印方法内联信息

```bash
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining
```

打印结果

![image-20240124195754593](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240124195754593.png)

禁用方法内敛

```bash
-XX:CompileCommand=dontinline,*Demo5_2.square
```

### 字段优化

导入maven坐标

```xml
<dependencies>
    <dependency>
        <groupId>org.openjdk.jmh</groupId>
        <artifactId>jmh-core</artifactId>
        <version>1.20</version>
    </dependency>
    <dependency>
        <groupId>org.openjdk.jmh</groupId>
        <artifactId>jmh-generator-annprocess</artifactId>
        <version>1.30</version>
    </dependency>
</dependencies>
```

java源代码

```java
// 预热，让JIT优化，进行两轮预热
@Warmup(iterations = 2, time = 1)
// 进行五轮测试
@Measurement(iterations = 5, time = 1)
@State(Scope.Benchmark)
public class Demo5_3 {

	int[] elements = randomInts(1_000);

	private static int[] randomInts(int size) {
		Random random = ThreadLocalRandom.current();
		int[] values = new int[size];
		for (int i = 0; i < size; i++) {
			values[i] = random.nextInt();
		}
		return values;
	}

	@Benchmark
	public void test1() {
		for (int i = 0; i < elements.length; i++) {
			doSum(elements[i]);
		}
	}

	@Benchmark
	public void test2() {
		int[] local = elements;
		for (int i = 0; i < local.length; i++) {
			doSum(local[i]);
		}
	}

	@Benchmark
	public void test3() {
		for (int element : elements) {
			doSum(element);
		}
	}

	static int sum = 0;

	// 允许方法内联
	@CompilerControl(CompilerControl.Mode.INLINE)
	private void doSum(int x) {
		sum += x;
	}

	public static void main(String[] args) throws RunnerException {
		Options opt = new OptionsBuilder()
				.include(Demo5_3.class.getCanonicalName())
				.forks(1)
				.build();
		new Runner(opt).run();
	}

}
```

开启方法内联的结果（数量级大概是$10^7$）

```bash
Benchmark       Mode  Cnt        Score         Error  Units
Demo5_3.test1  thrpt    5  2786819.883 ± 1462387.801  ops/s
Demo5_3.test2  thrpt    5  3168521.259 ±   93445.812  ops/s
Demo5_3.test3  thrpt    5  2655471.589 ± 1588702.743  ops/s
```

禁用方法内联的结果（数量级大概是$10^6$）

```bash
Benchmark       Mode  Cnt       Score        Error  Units
Demo5_3.test1  thrpt    5  434569.137 ± 112294.773  ops/s
Demo5_3.test2  thrpt    5  584581.686 ± 120031.425  ops/s
Demo5_3.test3  thrpt    5  599731.058 ±  53864.268  ops/s
```

在禁用方法内联的情况下，方法一性能最差。这是因为内联做了以下优化：

```java
@Benchmark
public void test1() {
    // elements 首次读取会被缓存起来 -> int[] local
    for (int i = 0; i < elements.length; i++) { // 后续999次长度都可以从local缓存获取
        sum += elements[i]; // 1000次的取下标i的元素也可以从local缓存获取
    }
}
```

方法二相当于手动优化了

```java
@Benchmark
public void test2() {
    int[] local = elements;
    for (int i = 0; i < local.length; i++) {
        doSum(local[i]);
    }
}
```

方法三是运行时优化（语法糖，增强for循环），字节码与方法二等价

```java
@Benchmark
public void test3() {
    for (int element : elements) {
        doSum(element);
    }
}
```

给我们的提示

- 尽量使用局部变量，不要使用静态成员变量和成员变量

### 反射优化

java源码

```java
public Object invoke(Object var1, Object[] var2) throws IllegalArgumentException, InvocationTargetException {
    if (++this.numInvocations > ReflectionFactory.inflationThreshold() && !ReflectUtil.isVMAnonymousClass(this.method.getDeclaringClass())) {
        MethodAccessorImpl var3 = (MethodAccessorImpl)(new MethodAccessorGenerator()).generateMethod(this.method.getDeclaringClass(), this.method.getName(), this.method.getParameterTypes(), this.method.getReturnType(), this.method.getExceptionTypes(), this.method.getModifiers());
        this.parent.setDelegate(var3);
    }

    return invoke0(this.method, var1, var2);
}
```

其中，如果 `++this.numInvocations > ReflectionFactory.inflationThreshold()` 即这个方法调用的次数超过了阈值，那么就进行如下方法调用如下方法。

```java
MethodAccessorImpl var3 = (MethodAccessorImpl)(new MethodAccessorGenerator()).generateMethod(this.method.getDeclaringClass(), this.method.getName(), this.method.getParameterTypes(), this.method.getReturnType(), this.method.getExceptionTypes(), this.method.getModifiers());
this.parent.setDelegate(var3);
```

这里的var3为如下的值

![image-20240124210800415](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240124210800415.png)

通过 `arthas-boot.jar` 工具获取这个运行时生成的类

![image-20240124210408822](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240124210408822.png)

发现直接调用方法。其中，阈值是15，代码如下：

```java
private static int inflationThreshold = 15;

static int inflationThreshold() {
    return inflationThreshold;
}
```

如果调用次数没有超出阈值，那么则调用 ` incoke0`

```java
private static native Object invoke0(Method var0, Object var1, Object[] var2);
```

这是一个本地方法，运行效率较低。

所以，前16次调用是通过本地方法实现的，之后的调用是直接调用对象或类的方法。

## java内存模型

JMM 即 Java Memory Model，它定义了主存（共享内存）、工作内存（线程私有）抽象概念，底层对应着 CPU 寄存器、缓存、硬件内存、 CPU 指令优化等。JMM 体现在以下几个方面

- 原子性 - 保证指令不会受到线程上下文切换的影响
- 可见性 - 保证指令不会受 cpu 缓存的影响
- 有序性 - 保证指令不会受 cpu 指令并行优化的影响

### 原子性

**synchronized关键字**

多线程带来的问题（交错执行）

```java
public class Demo6_1 {

	static int i = 0;

	static Object object = new Object();

	public static void main(String[] args) throws InterruptedException {
		Thread t1 = new Thread(() -> {
			for (int j = 0; j < 1000; j++) {
				i++;
			}
		});

		Thread t2 = new Thread(() -> {
			for (int j = 0; j < 1000; j++) {
				i--;
			}
		});

		t1.start();
		t2.start();

		t1.join();
		t2.join();

		System.out.println(i);
	}

}
```

对于 `i++` 而言，此处的i是静态变量，这不是一个原子操作，由以下四个步骤组成：

```bash
getstatic 		i
iconst_1		1
iadd		
putstatic		i
```

java的内存模型：

![image-20240124214951344](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240124214951344.png)

完成 `i++` 和 `i--` 需要在主存和线程内存中进行数据交换。

如果是单线程情况下，不会存在问题；但是如果是多线程情况下，那么可能存在问题，这是因为两个线程的指令交错运行。

**问题解决：synchronized**

```java
public class Demo6_1 {

	static int i = 0;

	static Object object = new Object();

	public static void main(String[] args) throws InterruptedException {
		Thread t1 = new Thread(() -> {
			for (int j = 0; j < 50000; j++) {
				synchronized (object) {
					i++;
				}
			}
		});

		Thread t2 = new Thread(() -> {
			for (int j = 0; j < 50000; j++) {
				synchronized (object) {
					i--;
				}
			}
		});

		t1.start();
		t2.start();

		t1.join();
		t2.join();

		System.out.println(i);
	}

}
```

monitor有三个部分：Owner、EntryList、WaitSe

1. **Owner（拥有者）：** 拥有锁的线程。在 Java 中，每个对象都有一个相关的监视器，而且每个监视器最多只能被一个线程拥有。如果一个线程获得了该对象的锁，那么它就成为了这个监视器的拥有者。
2. **EntryList（等待队列）：** 保存了当前竞争锁失败的线程，这些线程会在等待队列中等待机会再次争夺锁。在 EntryList 中的线程状态为 BLOCKED。
3. **WaitSet（等待集合）：** 保存了调用了 `Object.wait()` 方法被阻塞的线程。这些线程会在 WaitSet 中等待被唤醒。被唤醒的线程会重新进入 EntryList。

当一个线程通过 `synchronized` 关键字获取一个对象的锁时，它就成为了这个监视器的 Owner。其他线程如果想要获取这个锁，就必须进入 EntryList，并在锁释放时进行竞争。同时，线程可以调用 `Object.wait()` 方法来进入 WaitSet，等待其他线程唤醒。t

代码优化：

```java
Thread t1 = new Thread(() -> {
    synchronized (object) {
    	for (int j = 0; j < 1000; j++) {
            i++;
        }
    }
});
```

原来的代码，`monitorEnter` 和 `monitorExit` 执行此时很多，每个线程各执行100次。在改进后的版本中，`monitorEnter` 和 `monitorExit` 执行的次数将大大减少。

### 可见性



java源代码

```java
public class Demo6_2 {

	static volatile boolean run  = true;

	public static void main(String[] args) throws InterruptedException {
		Thread t = new Thread(() -> {
			while (run) {
			}
		});
		t.start();

		Thread.sleep(1000L);
		run = false;
	}

}
```

这个程序在运行一秒之后不会停止。

初始情况下，线程t从主存读取run的值到工作内存

![image-20240124215804348](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240124215804348.png)

由于此处t线程频繁读取主存中的run，所以JIT编译器会将run的值缓存在机子的工作内存中的告诉缓存中，减少t线程对主存中run的访问，提高执行效率

![image-20240124215926455](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240124215926455.png)

但是在运行1秒之后，主线程将主存中的run修改为false，但是t线程还是从自己的告诉缓存中读取run变量，所以t线程一直运行。

![image-20240124220135546](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240124220135546.png)

**问题解决**

volatile（易变关键字）

volatile修饰的变量（静态成员变量或者成员变量）可以避免线程从自己的工作缓存中查找变量的值，必须到主存中获取它的值，线程操作volatile变量都是从主存获取值，并非自己的高速缓存。

**可见性**：一个线程对volatile变量的修改对另一个线程是可见的（不能保证原子性）。

volatile和synchronized：

- volatile能保证可见性，但不能保证原子性，是轻量级操作
- synchronized能保证可见性、原子性，是重量级操作，新能相对较低。

**扩展**

java代码

```java
public class Demo6_2 {

	static boolean run  = true;

	public static void main(String[] args) throws InterruptedException {
		Thread t = new Thread(() -> {
			while (run) {
				System.out.println(1);
			}
		});
		t.start();

		Thread.sleep(1000L);
		run = false;

		System.out.println();
	}

}
```

此处也能在1秒后停止。

println(int)源码中有 `synchronized` 关键字，也能防止t线程从高速缓存中获取run，破坏了JIT优化

```java
public void println(int x) {
    synchronized (this) {
        print(x);
        newLine();
    }
}
```

### 有序性

```java
int num = 0;
boolean ready = false;

// 线程1
public void actor1(I_Result r) {
    if(ready) {
        r.r1 = num + num;
    } else {
        r.r1 = 1;
    }
}

// 线程2
public void actor2(I_Result r) {
    num = 2;
    ready = true;
}

class I_Result {
    public int r1;
}
```

在多线程环境下，以上的代码 r1 的值有三种情况：

- 第一种：线程 2 先执行，然后线程 1 后执行，r1 的结果为 4

- 第二种：线程 1 先执行，然后线程 2 后执行，r1 的结果为 1

- 第三种：线程 2 先执行，但是发送了指令重排，num = 2 与 ready = true 这两行代码语序发生转换，此时num还是0，所以r.r1 = 0 + 0，r1的结果是0

  ```java
  ready = true; // 前
  num = 2; // 后
  ```

**指令重排**：JIT编译器在运行时会进行优化

**测试指令重排**

- 导入maven坐标

```xml
<dependency>
    <groupId>org.openjdk.jcstress</groupId>
    <artifactId>jcstress-core</artifactId>
    <version>0.10</version>
</dependency>
```

java代码

```java
@JCStressTest
@Outcome(id = {"1", "4"}, expect = Expect.ACCEPTABLE, desc = "ok")
@Outcome(id = {"0"}, expect = Expect.ACCEPTABLE_INTERESTING, desc = "error")
@State
public class Demo6_3 {

	int num = 0;
	boolean ready = false;

	@Actor
	public void actor1(I_Result r) {
		if(ready) {
			r.r1 = num + num;
		} else {
			r.r1 = 1;
		}
	}

	@Actor
	public void actor2(I_Result r) {
		num = 2;
		ready = true;
	}

	public static void main(String[] args) {
	}

}
```

**解决方法**

volatile 修饰的变量，可以禁用**指令重排**，禁止的是加 volatile 关键字变量之前的代码重排序

```java
@JCStressTest
@Outcome(id = {"1", "4"}, expect = Expect.ACCEPTABLE, desc = "ok")
@Outcome(id = {"0"}, expect = Expect.ACCEPTABLE_INTERESTING, desc = "error")
@State
public class Demo6_3 {

	int num = 0;
	volatile boolean ready = false;

	@Actor
	public void actor1(I_Result r) {
		if(ready) {
			r.r1 = num + num;
		} else {
			r.r1 = 1;
		}
	}

	@Actor
	public void actor2(I_Result r) {
		num = 2;
		ready = true;
	}

	public static void main(String[] args) {
	}

}
```

**double-checked locking** 问题

java源码

```java
	// 最开始的单例模式是这样的
    public final class Singleton {
        private Singleton() { }
        private static Singleton INSTANCE = null;
        public static Singleton getInstance() {
        // 首次访问会同步，而之后的使用不用进入synchronized
        synchronized(Singleton.class) {
        	if (INSTANCE == null) { // t1
        		INSTANCE = new Singleton();
            }
        }
            return INSTANCE;
        }
    }
// 但是上面的代码块的效率是有问题的，因为即使已经产生了单实例之后，之后调用了getInstance()方法之后还是会加锁，这会严重影响性能！因此就有了模式如下double-checked lockin：
    public final class Singleton {
        private Singleton() { }
        private static Singleton INSTANCE = null;
        public static Singleton getInstance() {
            if(INSTANCE == null) { // t2
                // 首次访问会同步，而之后的使用没有 synchronized
                synchronized(Singleton.class) {
                    if (INSTANCE == null) { // t1
                        INSTANCE = new Singleton();
                    }
                }
            }
            return INSTANCE;
        }
    }
//但是上面的if(INSTANCE == null)判断代码没有在同步代码块synchronized中，不能享有synchronized保证的原子性，可见性。所以可能存在问题
```

其中 `INSTANCE = new Singleton();` 对应的字节码为

```bash
17: new #3 // class cn/itcast/n5/Singleton
// 复制了一个实例的引用
20: dup
// 通过这个复制的引用调用它的构造方法
21: invokespecial #4 // Method "<init>":()V
// 最开始的这个引用用来进行赋值操作
24: putstatic #2 // Field INSTANCE:Lcn/itcast/n5/Singleton;
```

21和24执行的顺序是不固定的，jvm可能优化为先将引用地址赋值给INSTANCE变量后，然后再执行构造方法，在两个线程的情况下，可能出现以下执行顺序

```bash
时间1 t1 线程执行到INSTANCE = new Singleton();
时间2 t1 线程为Singleton对象生成了引用地址
时间3 t1 线程将引用地址赋值给INSTANCE，此时INSTANCE != NULL，但是INSTANCE未经过构造方法初始化
时间4 t2 线程进入getInstance方法，发现INSTANCE != NULL，则直接返回INSTANCE
时间5 t1 线程执行Singleton的构造方法
```

此时t2线程获取的是未被构造函数初始化的单例。

**解决**：使用voltile对INSTANCE修饰即可，禁用指令重排（jdk5以后才能生效）

```java
public final class Singleton {
    private Singleton() { }
    private static volatile Singleton INSTANCE = null;
    public static Singleton getInstance() {
        if(INSTANCE == null) { // t2
            // 首次访问会同步，而之后的使用没有 synchronized
            synchronized(Singleton.class) {
                if (INSTANCE == null) { // t1
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
}
```

**happens-before** 规则

1.线程解锁 m 之前对变量的写，对于接下来对 m 加锁的其它线程对该变量的读可见

```java
public class Demo6_4 {

    static int x;
    static Object m = new Object();

    public static void main(String[] args) {
       new Thread(()->{
          synchronized(m) {
             x = 10;
          }
       },"t1").start();
       new Thread(()->{
          synchronized(m) {
             System.out.println(x);
          }
       },"t2").start();

    }

}
```

2.线程对 volatile 变量的写，对接下来其它线程对该变量的读可见

```java
public class Demo6_4 {

    volatile static int x;

    public static void main(String[] args) {
       new Thread(()->{
          x = 10;
       },"t1").start();
       new Thread(()->{
          System.out.println(x);
       },"t2").start();

    }

}
```

3.线程 start 前对变量的写，对该线程开始后对该变量的读可见

```java
public class Demo6_4 {

    static int x;

    public static void main(String[] args) {
       x = 10;
       new Thread(()->{
          System.out.println(x);
       },"t2").start();
    }

}
```

4.线程结束前对变量的写，对其它线程得知它结束后的读可见

```java
public class Demo6_4 {

    static int x;

    public static void main(String[] args) throws InterruptedException {
       Thread t1 = new Thread(()->{
          x = 10;
       },"t1");
       t1.start();
       t1.join();
       System.out.println(x);
    }

}
```

5.线程 t1 打断 t2（interrupt）前对变量的写，对于其他线程得知 t2 被打断后对变量的读可见

```java
public class Demo6_4 {

	static int x;

	public static void main(String[] args) throws InterruptedException {
		Thread t2 = new Thread(()->{
			while(true) {
				if(Thread.currentThread().isInterrupted()) {
					System.out.println(x);
					break;
				}
			}
		},"t2");
		t2.start();
		new Thread(()->{
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				throw new RuntimeException(e);
			}
			x = 10;
			t2.interrupt();
		},"t1").start();
		while(!t2.isInterrupted()) {
			Thread.yield();
		}
		System.out.println(x);
	}

}
```

## CAS和原子类

### CAS

CAS（Compare and Swap），乐观锁。

例：多线程要对共享变量x执行+1操作。

```java
volatile static int x = 1; // 共享变量
// 线程1
while (true) {
    int oldNum = x;
    int newNum = oldNum + 1;
        
    if(oldNum == x) {
        x = newNum;
        break;
    }
}
```

获取贡献变量时，为了保证改变了的可见性，所以需要把共使用volatile来修饰，结合CAS和volatile可以实现 **无锁并发**，适用于竞争不激烈、多喝CPU的场景。

- 因为没有synchronized，所以线程不会陷入阻塞（涉及上下文切换），所以效率有所提升
- 如果多线程对共享变量竞争激烈，那么会导致频繁的重试，这会影响效率

**乐观锁和悲观锁**

乐观锁：CAS基于乐观锁的思想，最乐观的估计，不怕别的线程来修改共享变量，就算修改了也没有关系，CAS在对共享变量进行修改前会进行Compare and Swap，如果旧值和共享变量相等，那么运行修改，此处为了保证一个线程对共享变量的修改对其他线程可见，所以需要将共享变需要用volatile来修饰

悲观锁：synchronized基于悲观锁的思想，最悲观的估计，得放着其他线程来修改共享变量，其他线程如果访问这个共享变量，则会被阻塞，只有这个线程释放了synchronized锁，其他线程才能访问这个共享变量。

### 原子操作类

juc中提供了原子操作类，可以提供线程安全的操作，例如：AtomicInteger，AtomicBoolean等，他们底层都采用CAS + volatile技术

以 AtomicInteger 为例讨论它的api接口：通过观察源码可以发现，AtomicInteger 内部都是通过cas的原理来实现的。

```java
public static void main(String[] args) {
    AtomicInteger i = new AtomicInteger(0);
    // 获取并自增（i = 0, 结果 i = 1, 返回 0），类似于 i++
    System.out.println(i.getAndIncrement());
    // 自增并获取（i = 1, 结果 i = 2, 返回 2），类似于 ++i
    System.out.println(i.incrementAndGet());
    // 自减并获取（i = 2, 结果 i = 1, 返回 1），类似于 --i
    System.out.println(i.decrementAndGet());
    // 获取并自减（i = 1, 结果 i = 0, 返回 1），类似于 i--
    System.out.println(i.getAndDecrement());
    // 获取并加值（i = 0, 结果 i = 5, 返回 0）
    System.out.println(i.getAndAdd(5));
    // 加值并获取（i = 5, 结果 i = 0, 返回 0）
    System.out.println(i.addAndGet(-5));
    // 获取并更新（i = 0, p 为 i 的当前值, 结果 i = -2, 返回 0）
    // 函数式编程接口，其中函数中的操作能保证原子，但函数需要无副作用
    System.out.println(i.getAndUpdate(p -> p - 2));
    // 更新并获取（i = -2, p 为 i 的当前值, 结果 i = 0, 返回 0）
    // 函数式编程接口，其中函数中的操作能保证原子，但函数需要无副作用
    System.out.println(i.updateAndGet(p -> p + 2));
    // 获取并计算（i = 0, p 为 i 的当前值, x 为参数1, 结果 i = 10, 返回 0）
    // 函数式编程接口，其中函数中的操作能保证原子，但函数需要无副作用
    // getAndUpdate 如果在 lambda 中引用了外部的局部变量，要保证该局部变量是 final 的
    // getAndAccumulate 可以通过 参数1 来引用外部的局部变量，但因为其不在 lambda 中因此不必是 final
    System.out.println(i.getAndAccumulate(10, (p, x) -> p + x));
    // 计算并获取（i = 10, p 为 i 的当前值, x 为参数1值, 结果 i = 0, 返回 0）
    // 函数式编程接口，其中函数中的操作能保证原子，但函数需要无副作用
    System.out.println(i.accumulateAndGet(-10, (p, x) -> p + x));
}
```

## synchronized优化

Java HotSpot 虚拟机中，每个对象都有对象头(包括class 指针和Mark Word)。Mark Word平时存储这个对象的哈希码、分代年龄（在幸存区到老年代晋升被用到），当加锁时，这些信息就根据情况被替换为标记位、线程锁记录指针、重量级锁指针、线程ID等内容。

**轻量级锁**

如果一个对象虽然有多线程访问，但是多线程访问的时间是错开的，没有竞争，那么可以使用轻量级锁

### 锁重入

![image-20240125105147655](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240125105147655.png)

![image-20240125105215819](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240125105215819.png)

在锁重入的时候，对象Mark Word存了两份线程1锁记录地址

### 锁膨胀

![image-20240125105455151](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240125105455151.png)

![image-20240125105622335](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240125105622335.png)

重量锁指针是为了线程1执行完之后能唤醒阻塞中的线程。

### 重量级锁

重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功(即这时候持锁线程已经退出了同步块，释放了锁)，这时当前线程就可以避免阻塞。

在Java6之后自旋锁是**自适应**的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次;反之，就少自旋甚至不自旋，总之，比较智能。

- 自旋会占用CPU时间，单核CPU自旋就是浪费，多核CPU自旋才能发挥优势。
- 好比等红灯时汽车是不是熄火，不熄火相当于自旋（等待时间短了划算)，熄火了相当于阻塞（等待时间长了划算)
- Java 7之后不能控制是否开启自旋功能

自旋重试成功，这样的话，可以不用阻塞，阻塞开销太大

![image-20240125105829648](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240125105829648.png)

自旋失败，自旋重试如果超过阈值，则会陷入阻塞

![image-20240125105919704](C:\Users\29465\AppData\Roaming\Typora\typora-user-images\image-20240125105919704.png)

### 偏向锁

轻量级锁在没有竞争时(就自己这个线程)，**每次重入仍然需要执行CAS操作**，很浪费时间。

Java 6中引入了**偏向锁**来做进一步优化：只有第一次使用CAS将**线程ID**设置到对象的Mark Word头，之后发现这个线程ID是自己的就表示没有竞争，不用重新CAS.

- 撤销偏向需要将持锁线程升级为轻量级锁，这个过程中所有线程需要暂停(STW)
- 访问对象的hashCode也会撤销偏向锁，无锁状态下对象头存放的是hashCode，但是加了偏向锁以后，对象头存放的是线程id，hashCode被放到了加锁线程中去，所以如果需要访问对象的hashCode，那么需要撤销偏向锁
- 如果对象虽然被多个线程访问，但没有竞争，这时偏向了线程T1的对象仍有机会重新偏向T2，重偏向会重置对象的Thread ID（重偏向）
- 撤销偏向和重偏向都是批量进行的，以类为单位
- 如果撤销偏向到达某个阈值，整个类的所有对象都会变为不可偏向的
- 可以主动使用 `-XX:-UseBiasedLocking` 禁用偏向锁

### 其他优化

- 减少上锁时间

- 减少锁的粒度（将一个锁拆分成多个锁提高并发度）

- 锁粗化（多次循环进入同步块不如同步块内多次循环），下面的例子中：JVM会把多次append的加锁操作粗化为一次（因为是对同一个对象进行加锁，没必要重入多次）

  ```java
  new StringBuffer().append("a").append("b").append("c");
  ```

- 锁消除（JVM会进行代码的逃逸分析，例如某个加锁对象是方法内局部变量，不会被其它线程所访问到，这时候就会被即时编译器忽略掉所有同步操作。）
- 读写分离