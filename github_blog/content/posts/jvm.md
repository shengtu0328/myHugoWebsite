---
title: "JVM学习"
date: 2022-10-11T18:23:30+08:00
draft: false
categories: ["tech"]
---

## 一、什么是JVM

### 定义

Java Virtual Machine，JAVA程序的**运行环境**（JAVA二进制字节码的运行环境）

### 功能

**解释和运行**  对字节码文件中的指令，实时的解释成机器码，让计算机执行

**内存管理**    自动为对象、方法等分配内存，自动的垃圾回收机制

**即时编译**   对热点代码进行优化，提升执行效率 ( java 语言需要将字节码指令实时地通过jvm解释成机器码，才可以交给计算机运行，随着程序的执行，需要反复地去进行，

每次运行都要经历  解释成机器码----->运行机器码 这两部

 但有了jit后  直接把热点代码的机器码放进内存中， 少了解释成机器码文件这一步   )

![](jit.png)

### 好处

- 一次编写，到处运行 (屏蔽了字节码和操作系统和的差异，对外提供一致的操作环境，从而可以跨平台，如在windows，linux运行、)

- 自动内存管理，垃圾回收机制 (c,c++之前没有自动垃圾回收)

- 数组下标越界检查 （c语言也没有?）

- 多态（虚方法表实现？)

 ![](001.png)

### jvm参数

- 栈内存大小

  ```
  -Xss1024k
  ```

- 堆内存大小	
  ```
  -Xmx4G
  ```
- 永久代大小 1.8 以前
  ```
  -XX:MaxPermSize=8m
  ```
- 元空间大小 1.8 以后
  ```
  -XX:MaxMetaspaceSize=8m
  ```
- 堆空间回收不足开关 +是启用，-是不启用
  ```
  -XX:-UseGCOverheadLimit
  ```
  ```
  +XX:-UseGCOverheadLimit
  ```

- StringTable 桶个数

  ```
  -XX:StringTableSize=20000
  ```
- 禁用显示垃圾回收，如System.gc() 注意：System.gc()是full gc，新生代，老年代。
  ```
  -XX:+DisableExplicitGC 显式的
  ```

## 二、内存结构
 ![](002.png)
 - 方法区:存放了类

 - 堆:存放了对象

 - 虚拟机栈&程序器计数器&本地方法栈: 对象调用方法时会用到

 - 解释器:每行代码执行

 - 即时的编译器:热点代码执行

 - 垃圾回收:回收在堆中不被引用的对象

 - 本地方法接口:操作系统的功能方法

   

### 1、程序计数器
  ![](005.png)
#### 作用

   用于保存JVM中下一条所要执行的指令的地址
#### 特点

   - 线程私有
     - CPU会为每个线程分配时间片，当当前线程的时间片使用完以后，CPU就会去执行另一个线程中的代码
     - 程序计数器是**每个线程**所**私有**的，每个线程都有自己的程序计数器，当另一个线程的时间片用完，又返回来执行当前线程的代码时，通过程序计数器可以知道应该执行哪一句指令
     
   - 不会存在内存溢出.
### 2、虚拟机栈
####    定义
- 每个**线程**运行需要的内存空间，称为**虚拟机栈**
- 每个栈由多个**栈帧**组成，对应着每次调用**方法**时所占用的内存
- 每个线程只能有**一个活动栈帧**，对应着**当前正在执行的方法**
- Student s=new Student()；s这个变量也是在栈里，new Student()对象是在堆里
####  演示

代码

```
public class Main {
	public static void main(String[] args) {
		method1();
	}

	private static void method1() {
		method2(1, 2);
	}

	private static int method2(int a, int b) {
		int c = a + b;
		return c;
	}
}
```
可以参考idea debug模式下的 Frames

  ![](008_stack.png)

####  问题辨析

- 垃圾回收是否涉及栈内存？
  - **不需要**。因为虚拟机栈中是由一个个栈帧组成的，在方法执行完毕后，对应的栈帧就会被弹出栈。所以无需通过垃圾回收机制去回收内存。
  
- 栈内存的分配越大越好吗？
  - 不是。因为**物理内存是一定的**，栈内存越大，可以支持更多的递归调用，但是可执行的线程数就会越少。
  - 线程数=物理内存/栈内存
  - -Xss1024k （栈内存JVM默认参数配置，默认大小是1M）
  
- 方法内的局部变量是否是线程安全的？
  - 如果方法内**局部变量没有逃离方法的作用范围**，则是**线程安全**的
  - 如果如果**局部变量引用了对象**或者静态变量，并**逃离了方法的作用范围**，则需要考虑线程安全问题


#### 内存溢出

  **Java.lang.stackOverflowError** 栈内存溢出

  **发生原因**

  - 虚拟机栈中，**栈帧过多**（无限递归，第三方库出现的问题如json序列化对象互相引用）

  - 每个栈帧**所占用过大**

    可以通过idea中Configurations中设置VM options 设置jvm参数：如-Xss256k

      ![](012.png)

#### 线程运行诊断

   CPU占用过高

- Linux环境下运行某些程序的时候，可能导致CPU的占用过高，这时需要定位占用CPU过高的线程
  - **top**
  
    查看**进程**的CPU和内存使用情况,会显示进程id(PID),但还定位不到线程，
  
  - **ps H -eo pid,tid,%cpu | grep 我是刚才通过top查到的进程号**
  
    查看**线程**占用CPU和内存使用情况
  
    **H** 是显示进程里所有线程树的意思
  
    **-eo**是输出结果 ，**pid:**进程id，**tid:**线程id，,**%cpu**:cpu占用率 
  
  - **jstack 我是一个进程id** 
  
    通过查看进程中的线程的id，刚才通过ps命令看到的tid来**对比定位**，注意jstack查找出的线程id是**16进制的**，**需要转换**

### 3、本地方法栈

​    ![](016_nativeMethod.png)

  一些带有**native关键字**的方法就是需要JAVA去调用本地的C或者C++方法，因为JAVA有时候没法直接和操作系统底层交互，所以需要用到本地方法



### 4、堆

   Heap 堆

​    ![](017_堆.png)

#### 定义

通过new关键字**创建的对象**都会被放在堆内存

#### 特点

- **所有线程共享**，堆内存中的对象都需要**考虑线程安全问题**
- 有垃圾回收机制
- jvm参数：-Xmx4G   可以通过修改jvm堆内存的参数大小，尽早暴露问题

#### 堆内存溢出

**java.lang.OutofMemoryError** ：java heap space. 堆内存溢出

比如一个size很大的list，就会发生堆内存溢出

#### 堆内存诊断

1. jps 工具

- 查看当前系统中有哪些 java 进程

-  jps

2. jmap 工具

- 查看堆内存占用情况 jmap - heap 进程id，只能是某一时刻

- jmap -heap 5750

3. jconsole 工具

- 图形界面的，多功能的监测工具，可以连续监测

- jconsole

  ​    ![](020_jconsole.png)
4. java VisualVm
- jvisualvm 图形界面，更加强大，类似idea debug模式下看到的信息

     ![](021_jvisualvm.png)



### 5、方法区

   ![](022_方法区.png)

https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.5.4

The Java Virtual Machine has a ***method area*** that is **shared among all Java Virtual Machine threads**. The method area is analogous（相似的） to the storage area for compiled code of a conventional language or analogous to the "text" segment in an operating system process. It stores per-class structures such as the run-time constant pool（**运行时常量池**), field （ ）and method data, and the code for methods(**方法**) and constructors（**构造方法**）, including the special methods ([§2.9](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.9)) used in class and instance initialization and interface initialization.

The method area is created on virtual machine start-up. Although the method area is logically part of the heap, simple implementations may choose not to either garbage collect or compact it. This specification does not mandate the location of the method area or the policies used to manage compiled code. The method area may be of a fixed size or may be expanded as required by the computation and may be contracted if a larger method area becomes unnecessary. The memory for the method area does not need to be contiguous.

#### 定义

- 方法区是所有线程共享的，存储了运行时常量池，类模板。

- 方法区逻辑上定义为在堆中。

  1.8以前叫永久代，用的堆。

  1.8以后用叫元空间，用的操作系统内存。StringTable串池还是在堆中

- 方法区也会内存溢出



   ![](022_方法区定义.png)



#### 内存溢出

jvm参数： -XX:MaxMetaspaceSize=8m

1.8 以前会导致永久代内存溢出

* 演示永久代内存溢出 java.lang.OutOfMemoryError: PermGen space 
*  -XX:MaxPermSize=8m

1.8 之后会导致元空间内存溢出 

* 演示元空间内存溢出 java.lang.OutOfMemoryError: Metaspace 

* -XX:MaxMetaspaceSize=8m

```
/**
 * 演示元空间内存溢出 java.lang.OutOfMemoryError: Metaspace
 * 元空间 大小参数设置
 * -XX:MaxMetaspaceSize=8m
 */
public class Demo1_8 extends ClassLoader { // 可以用来加载类的二进制字节码
    public static void main(String[] args) {
        int j = 0;
        try {
            Demo1_8 test = new Demo1_8();
            for (int i = 0; i < 10000; i++, j++) {
                // ClassWriter 作用是生成类的二进制字节码
                ClassWriter cw = new ClassWriter(0);
                //参数定义: 版本号， public， 类名, 包名, 父类， 接口
                cw.visit(Opcodes.V1_8, Opcodes.ACC_PUBLIC, "Class" + i, null, "java/lang/Object", null);
                // 返回 byte[]
                byte[] code = cw.toByteArray();
                // 执行了类的加载
                test.defineClass("Class" + i, code, 0, code.length); // Class 对象
            }
        } finally {
            System.out.println(j);
        }
    }
}
```



工作中加载类的场景

* spring

* mybatis

#### 常量池

##### 定义

- 常量池
  
  就是一张表，虚拟机指令根据这张常量表找到要执行的类名、方法名、参数类型、字面量（即参数，字符串，布尔类型，整形）信息
  
- 运行时常量池
   常量池是*.class文件中的，当该**类被加载以后**，它的常量池信息就会放入运行时常量池（也就是内存中），并把里面的符号地址变为真实地址。(即#1，#2变成物理地址)



##### 举例说明

用java代码反编译举个例子

 ```
  // 二进制字节码（类基本信息，常量池，类方法定义，包含了虚拟机指令）
  public class HelloWorld {
      public static void main(String[] args) {
          System.out.println("hello world");
      }
  }
 ```

* javap -v HelloWorld.class  

  反编译

```
========================类基本信息========================
  Last modified 2022-10-14; size 567 bytes
  MD5 checksum 8efebdac91aa496515fa1c161184e354
  Compiled from "HelloWorld.java"
public class cn.itcast.jvm.t5.HelloWorld
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
  
========================常量池========================
Constant pool:
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #21.#22        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #23            // hello world
   #4 = Methodref          #24.#25        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #26            // cn/itcast/jvm/t5/HelloWorld
   #6 = Class              #27            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lcn/itcast/jvm/t5/HelloWorld;
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
  #26 = Utf8               cn/itcast/jvm/t5/HelloWorld
  #27 = Utf8               java/lang/Object
  #28 = Utf8               java/lang/System
  #29 = Utf8               out
  #30 = Utf8               Ljava/io/PrintStream;
  #31 = Utf8               java/io/PrintStream
  #32 = Utf8               println
  #33 = Utf8               (Ljava/lang/String;)V
  
    
========================类方法定义========================
{
  public cn.itcast.jvm.t5.HelloWorld();       //构造方法
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 4: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcn/itcast/jvm/t5/HelloWorld;

  public static void main(java.lang.String[]);    //main 方法
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // 获取System.out这个静态变量
         3: ldc           #3                  // String hello world 加载一个参数
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V 虚方法调用
         8: return
      LineNumberTable:
        line 6: 0
        line 7: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
}
SourceFile: "HelloWorld.java"

```

| 翻译下就是       |                                                              |
| ---------------- | ------------------------------------------------------------ |
| getstatic #2     | 去常量池(Constant pool）找 #2 的变量,最终找到了System.out这个静态成员变量 |
|                  | #2 = Fieldref 是一个成员变量的引用                           |
|                  | #2->#21.#22                                                  |
|                  | #21 = Class              #28                                 |
|                  | #22 = NameAndType        #29:#30                             |
|                  | #28 = Utf8               java/lang/System                    |
|                  | #29 = Utf8               out                                 |
|                  | #30 = Utf8               Ljava/io/PrintStream;               |
|                  |                                                              |
| ldc           #3 | #3 = String             #23                                  |
|                  | #23 = Utf8               hello world                         |
|                  |                                                              |
| invokevirtual #4 | 找到 java/io/PrintStream对象的println方法 参数是:(Ljava/lang/String;)V，并调用 |
|                  | #4 = Methodref          #24.#25                              |
|                  | #24 = Class              #31                                 |
|                  | #25 = NameAndType        #32:#33                             |
|                  | #31 = Utf8               java/io/PrintStream                 |
|                  | #32 = Utf8               println                             |
|                  | #33 = Utf8               (Ljava/lang/String;)V               |



#### StringTable（串池）

常量池->运行时常量池-

执行语句 String str = "abc" 时，首先查看字符串池中是否存在字符串"abc" ，如果存在则直接将"abc"地址赋给str ，如果不存在则先在字符串池中新建一个字符串"abc"，然后再将其赋给str。

String str = new String(“aa”)：至少会创建一个对象，也有可能创建两个。用到new关键字，肯定会在堆中创建一个String对象，如果字符池中已经存在”abc”,则不会在字符串池中创建一个String对象，如果不存在，则会在字符串常量池中也创建一个对象。



直到用到了该符号，才会创建对应的字符串对象放入字符串常量池

##### 举例说明(依据字节码)

java代码

```
// StringTable [ "a", "b" ,"ab" ]  hashtable 结构，不能扩容
public class Demo1_22 {
    // 常量池中的信息，都会被加载到运行时常量池中， 这时 a b ab 都是常量池中的符号，还没有变为 java 字符串对象
    // ldc #2 会把 a 符号变为 "a" 字符串对象
    // ldc #3 会把 b 符号变为 "b" 字符串对象
    // ldc #4 会把 ab 符号变为 "ab" 字符串对象

    public static void main(String[] args) {
        String s1 = "a"; // 懒惰的
        String s2 = "b";
        String s3 = "ab";
        String s4 = s1 + s2; //通过字节码可以知道，实际上是 new StringBuilder().append("a").append("b").toString() 和  new String("ab")
        System.out.println(s3 == s4);// false  因为s3在串池，s4在堆中
        String s5 = "a" + "b";  // javac 在编译期间的优化，结果已经在编译期确定为ab
        System.out.println(s3 == s5);//true 因为都是在串池

    }
}

```

java字节码

```
Classfile /G:/xrqProjects/jvm/代码/jvm/out/production/jvm/cn/itcast/jvm/t1/stringtable/Demo1_22.class
  Last modified 2022-10-17; size 1057 bytes
  MD5 checksum b94bb46c9f03f10beb6d57556ca42557
  Compiled from "Demo1_22.java"
public class cn.itcast.jvm.t1.stringtable.Demo1_22
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #12.#36        // java/lang/Object."<init>":()V
   #2 = String             #37            // a
   #3 = String             #38            // b
   #4 = String             #39            // ab
   #5 = Class              #40            // java/lang/StringBuilder
   #6 = Methodref          #5.#36         // java/lang/StringBuilder."<init>":()V
   #7 = Methodref          #5.#41         // java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
   #8 = Methodref          #5.#42         // java/lang/StringBuilder.toString:()Ljava/lang/String;
   #9 = Fieldref           #43.#44        // java/lang/System.out:Ljava/io/PrintStream;
  #10 = Methodref          #45.#46        // java/io/PrintStream.println:(Z)V
  #11 = Class              #47            // cn/itcast/jvm/t1/stringtable/Demo1_22
  #12 = Class              #48            // java/lang/Object
  #13 = Utf8               <init>
  #14 = Utf8               ()V
  #15 = Utf8               Code
  #16 = Utf8               LineNumberTable
  #17 = Utf8               LocalVariableTable
  #18 = Utf8               this
  #19 = Utf8               Lcn/itcast/jvm/t1/stringtable/Demo1_22;
  #20 = Utf8               main
  #21 = Utf8               ([Ljava/lang/String;)V
  #22 = Utf8               args
  #23 = Utf8               [Ljava/lang/String;
  #24 = Utf8               s1
  #25 = Utf8               Ljava/lang/String;
  #26 = Utf8               s2
  #27 = Utf8               s3
  #28 = Utf8               s4
  #29 = Utf8               s5
  #30 = Utf8               StackMapTable
  #31 = Class              #23            // "[Ljava/lang/String;"
  #32 = Class              #49            // java/lang/String
  #33 = Class              #50            // java/io/PrintStream
  #34 = Utf8               SourceFile
  #35 = Utf8               Demo1_22.java
  #36 = NameAndType        #13:#14        // "<init>":()V
  #37 = Utf8               a
  #38 = Utf8               b
  #39 = Utf8               ab
  #40 = Utf8               java/lang/StringBuilder
  #41 = NameAndType        #51:#52        // append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
  #42 = NameAndType        #53:#54        // toString:()Ljava/lang/String;
  #43 = Class              #55            // java/lang/System
  #44 = NameAndType        #56:#57        // out:Ljava/io/PrintStream;
  #45 = Class              #50            // java/io/PrintStream
  #46 = NameAndType        #58:#59        // println:(Z)V
  #47 = Utf8               cn/itcast/jvm/t1/stringtable/Demo1_22
  #48 = Utf8               java/lang/Object
  #49 = Utf8               java/lang/String
  #50 = Utf8               java/io/PrintStream
  #51 = Utf8               append
  #52 = Utf8               (Ljava/lang/String;)Ljava/lang/StringBuilder;
  #53 = Utf8               toString
  #54 = Utf8               ()Ljava/lang/String;
  #55 = Utf8               java/lang/System
  #56 = Utf8               out
  #57 = Utf8               Ljava/io/PrintStream;
  #58 = Utf8               println
  #59 = Utf8               (Z)V
{
  public cn.itcast.jvm.t1.stringtable.Demo1_22();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 4: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcn/itcast/jvm/t1/stringtable/Demo1_22;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=6, args_size=1
         0: ldc           #2                  // String a
         2: astore_1
         3: ldc           #3                  // String b
         5: astore_2
         6: ldc           #4                  // String ab  s3 == s5 为true!!!!!!!!!!!!!!!!!!!!!
         8: astore_3
         9: new           #5                  // class java/lang/StringBuilder
        12: dup
        13: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
        16: aload_1
        17: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        20: aload_2
        21: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        24: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        27: astore        4
        29: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
        32: aload_3
        33: aload         4
        35: if_acmpne     42
        38: iconst_1
        39: goto          43
        42: iconst_0
        43: invokevirtual #10                 // Method java/io/PrintStream.println:(Z)V
        46: ldc           #4                  // String ab    s3 == s5 为true!!!!!!!!!!!!!!!!!!!!!
        48: astore        5
        50: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
        53: aload_3
        54: aload         5
        56: if_acmpne     63
        59: iconst_1
        60: goto          64
        63: iconst_0
        64: invokevirtual #10                 // Method java/io/PrintStream.println:(Z)V
        67: return
      LineNumberTable:
        line 11: 0
        line 12: 3
        line 13: 6
        line 14: 9
        line 15: 29
        line 16: 46
        line 17: 50
        line 19: 67
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      68     0  args   [Ljava/lang/String;
            3      65     1    s1   Ljava/lang/String;
            6      62     2    s2   Ljava/lang/String;
            9      59     3    s3   Ljava/lang/String;
           29      39     4    s4   Ljava/lang/String;
           50      18     5    s5   Ljava/lang/String;
      StackMapTable: number_of_entries = 4
        frame_type = 255 /* full_frame */
          offset_delta = 42
          locals = [ class "[Ljava/lang/String;", class java/lang/String, class java/lang/String, class java/lang/String, class java/lang/String ]
          stack = [ class java/io/PrintStream ]
        frame_type = 255 /* full_frame */
          offset_delta = 0
          locals = [ class "[Ljava/lang/String;", class java/lang/String, class java/lang/String, class java/lang/String, class java/lang/String ]
          stack = [ class java/io/PrintStream, int ]
        frame_type = 255 /* full_frame */
          offset_delta = 19
          locals = [ class "[Ljava/lang/String;", class java/lang/String, class java/lang/String, class java/lang/String, class java/lang/String, class java/lang/String ]
          stack = [ class java/io/PrintStream ]
        frame_type = 255 /* full_frame */
          offset_delta = 0
          locals = [ class "[Ljava/lang/String;", class java/lang/String, class java/lang/String, class java/lang/String, class java/lang/String, class java/lang/String ]
          stack = [ class java/io/PrintStream, int ]
}
SourceFile: "Demo1_22.java"

```

##### 特征

- 常量池中的字符串仅是符号，只有**在被用到时才会转化为对象**
- 利用串池的机制，来避免重复创建字符串对象
- 字符串**变量**拼接的原理是**StringBuilder**
- 字符串**常量**拼接的原理是**编译器优化**
- 可以使用**intern方法**，主动将串池中还没有的字符串对象放入串池中
- **注意**：无论是串池还是堆里面的字符串，都是对象

##### intern方法 1.8

调用字符串对象的intern方法，会将该字符串对象**的引用**尝试放入到串池中

- 如果串池中没有该字符串对象，则放入

- 如果有该字符串对象，则不会放入

  无论放入是否成功，都会返回**串池中**的字符串对象



JDK 1.7后，intern方法还是会先去查询常量池中是否有已经存在，如果存在，则返回常量池中的引用，这一点与之前没有区别，区别在于，如果在常量池找不到对应的字符串，则不会再将字符串拷贝到常量池，而只是在常量池中生成一个对原字符串的引用。简单的说，就是往常量池放的东西变了：原来在常量池中找不到时，复制一个副本放到常量池，1.7后则是将在堆上的地址引用复制到常量池。

```
例子1：先创建堆字符串，再intern()
public static void main(String[] args) {
    //串池：【"a" "b"】,堆：【new String("a")，new String("b"), new String("ab")】
    String s = new String("a") + new String("b");

    //调用intern方法，这时串池中没有"ab"，则会将s这个对象引用放入到串池中，此时堆内存与串池中的"ab"是同一个字符串对象的引用
    String s2 = s.intern();

    //与串池中的ab对象做比较 返回true
    System.out.println(s2 == "ab");

    //比较的是引用 返回true  jdk1.6这里是false
    System.out.println(s == "ab");
}
```

```
例子2：先创建串池字符串，后创建堆中字符串，再intern()
public static void main(String[] args) {
    //串池：【"ab"】
    String x = "ab";

    //串池：【"ab","a" "b"】,这一步只是添加了【"a" "b"】
    //并且在堆中创建了 s对象 ，s和x不是同一个对象，因为是串池中的“ab”先创建，堆中拼接后创建
    String s = new String("a") + new String("b");

    //串池中已经有ab了，所以s被没有被放进去，返回的是串池中的对象
    String s2 = s.intern();

    //s2和x都是串池中的对象 返回true
    System.out.println(s2 == x);

    //s是堆中的对象 x串池中对象 返回false
    System.out.println(s == x);
}
```

##### intern方法 1.6

调用字符串对象的intern方法，会将该字符串对象尝试放入到串池中

- 如果串池中没有该字符串对象，**会将该字符串对象复制一份**，再放入到串池中
- 如果有该字符串对象，则放入失败

无论放入是否成功，都会返回**串池中**的字符串对象

**注意**：此时无论调用intern方法成功与否，串池中的字符串对象和堆内存中的字符串对象**都不是同一个对象**，上面例子1中  System.out.println(s== "ab");返回false

#####  垃圾回收

StringTable在内存紧张时，会发生垃圾回收



在jvm1.6中，StringTable是常量池的一部分。

在jvm1.8中，StringTable存在了堆中。

为什么要变呢？因为永久代是在full gc的时候，才会gc。触发的时间比较慢。导致StringTable 回收效率不高。但StringTable 使用频繁，如java程序中会大量使用字符串常量。回收效率不高会占用大量内存，有可能会导致永久代内存不足。在堆中只要minor gc就会触发gc。

注意：

1.6永久代和堆逻辑上是相互隔离的，但它们使用的物理内存是连续的。

1.8元空间不再与堆连续，而且是存在于本地内存（Native memory）

https://blog.csdn.net/ju_362204801/article/details/122379813

##### StringTable调优

- StringTable是由类似juc里的HashTable实现的，所以可以**适当增加桶的个数**，从而减少哈希碰撞，来减少字符串放入串池所需要的时间。

```
-XX:StringTableSize=20000
```

- 如果有大量的字符串，并且有不少是重复的，可以通过intern方法减去重后入池，减少内存占用。因为在串池中，相同的字符串在串池只会存储一份。



#### 直接内存

- 属于操作系统，常见于NIO操作时，**用于数据缓冲区**
- 分配回收成本较高，但读写性能高
- 不受JVM内存回收管理

- java 想要读取文件时，需要从用户态切换到内核态，通过内核态将文件先读取到系统缓冲区，再读取到java缓冲区。


![](042_直接内存_Io.png)

- 
  直接内存是操作系统和Java代码都可以访问的一块区域，无需将代码从系统内存复制到Java堆内存，从而提高了效率。比如nio中的ByteBuffer.allocateDirect(16)。


![](042_直接内容_nio.png)

-  class java.nio.HeapByteBuffer          -java 堆内存，读写效率较低 ，就和普通javabean 对象一会受到GC影响
-  class java.nio.DirectByteBuffer        -直接内存 ，读写效率高（少一次拷贝），不会受到java GC的影响，分配内存效率低,但是也会有内存溢出的限制。



##### 释放原理

java.nio.DirectByteBuffer 直接内存的回收不是通过JVM的垃圾回收来释放的，而是最终通过**unsafe.freeMemory**来手动释放

- 使用了Unsafe类来完成直接内存的分配回收，回收需要主动调用freeMemory方法
- ByteBuffer的实现内部使用了Cleaner（虚引用）来检测ByteBuffer。一旦ByteBuffer被垃圾回收，那么会由ReferenceHandler来调用Cleaner的clean方法调用freeMemory来释放内存
- 如果java中 ByteBuffer对象没有垃圾回收，和它关联的直接内存对象不会自动回收，需要通过**unsafe.freeMemory**来手动释放

通过 ByteBuffer.allocateDirect 得到DirectByteBuffer

```
ByteBuffer bb = ByteBuffer.allocateDirect(1024 * 1024)
```




```
 DirectByteBuffer(int cap) {                   // package-private

        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);

        long base = 0;
        try {
            base = unsafe.allocateMemory(size);// 分配了DirectByteBuffer对象的直接内存！！！！！！！！！！！！！！！
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        //Cleaner是个虚引用类型 Cleaner extends PhantomReference<Object>，当它所关联的对象被gc时，会触发它的clean方法。也就是this对象触发gc时，会调用Cleaner 的clean方法。
        //本例中就是调用Deallocator里的 run方法 用unsafe 方式进行直接内存的回收
        //在后台有个专门检测虚引用对象的ReferenceHandler线程中 执行clean方法。
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));//
        att = null;



    }
```



回调任务对象Deallocator会释放DirectByteBuffer对象的直接内存！

```
 #Cleaner的 回调任务对象 
 private static class Deallocator
        implements Runnable
    {

        private static Unsafe unsafe = Unsafe.getUnsafe();

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
            unsafe.freeMemory(address);// 释放了DirectByteBuffer对象的直接内存！！！！！！！！！！！！！！！
            address = 0;
            Bits.unreserveMemory(size, capacity);      
        }

    }
```

## 三、垃圾回收

### 1、如何判断对象可以回收

#### 引用计数法

只要这个对象被其他对象引用就 计数加一，不再被引用就减一。0代表可以回收。

弊端：循环引用时，两个对象的计数都为1，导致两个对象都无法被释放

#### 可达性分析算法（JVM使用的）

- JVM中的垃圾回收器通过**可达性分析**来探索所有存活的对象

- 扫描 **堆** 中的对象，看能否沿着GC Root对象为起点的引用链找到该对象，如果**找不到，则表示可以回收**

  例子：抓住葡萄的根，连着的葡萄就是不可垃圾回收的对象，落下的葡萄就是可垃圾回收的对象

#### 可以作为GC Root的对象

  -指肯定不能被垃圾回收 的对象
  - 虚拟机栈（栈帧中的本地变量表）中引用的对象。　
  - 方法区中类静态属性引用的对象
  - 方法区中常量引用的对象
  - 本地方法栈中JNI（即一般说的Native方法）引用的对象
  - 被synchronized关键字持有的对象

#### 五种引用



##### 引用队列的作用

回收软引用或者弱引用本身

回收虚引用本身，并且释放直接内存

回收终结器引用，对象被垃圾回收，会加入引用队列，但是还不能被直接垃圾回收

##### 软引用或者弱引是怎么被回收的

就比如ByteBuffer对象会被jvm gc回收掉，但包装ByteBuffer对象的SoftReference对象不会被马上回收，要配合引用队列回收


![](054_5种引用.png)

##### 强引用

只有所有 GC Roots 对象都不通过【强引用】引用该对象，该对象才能被垃圾回收

- 如上图B、C对象都不引用A1对象时，A1对象才会被回收

  

<u>以下引用的前提:没有强引用引用该对象</u>

##### 软引用

仅有软引用引用该对象时，在垃圾回收后，内存仍不足时会再次出发垃圾回收，回收**软引用对象所引用的对象**

- 可以配合引用队列来释放软引用自身

- 当GC Root指向软引用对象，软引用对象引用了一个对象，在触发了垃圾回收,且内存不足时，会**回收软引用所引用的对象**

 就如 list --> SoftReference --> byte[]

- 如上图如果B对象不再引用 且 A2对象且内存不足时，软引用所引用的A2对象就会被回收
- 反射中的一些方法就用到了软引用

软引用的使用

```
import java.io.IOException;
import java.lang.ref.SoftReference;
import java.util.ArrayList;
import java.util.List;

/**
 * 演示软引用
 * -Xmx20m -XX:+PrintGCDetails -verbose:gc
 */
public class Demo2_3 {

    private static final int _4MB = 4 * 1024 * 1024;



    public static void main(String[] args) throws IOException {
        /*List<byte[]> list = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            list.add(new byte[_4MB]);
        }

        System.in.read();*/
        soft();


    }

    public static void soft() {
        // list --> SoftReference --> byte[]

        List<SoftReference<byte[]>> list = new ArrayList<>();
        for (int i = 0; i < 5; i++) {
            SoftReference<byte[]> ref = new SoftReference<>(new byte[_4MB]);
            System.out.println(ref.get());
            list.add(ref);
            System.out.println(list.size());

        }
        System.out.println("循环结束：" + list.size());
        for (SoftReference<byte[]> ref : list) {
            System.out.println(ref.get());
        }
    }
}
    
    
Connected to the target VM, address: '127.0.0.1:65198', transport: 'socket'
[B@6f75e721
1
[B@69222c14
2
[B@606d8acf
3
[GC (Allocation Failure) [PSYoungGen: 1777K->464K(6144K)] 14065K->12760K(19968K), 0.0007427 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[B@782830e
4
[GC (Allocation Failure) --[PSYoungGen: 4672K->4672K(6144K)] 16968K->16968K(19968K), 0.0007738 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
[Full GC (Ergonomics) [PSYoungGen: 4672K->4471K(6144K)] [ParOldGen: 12296K->12289K(13824K)] 16968K->16760K(19968K), [Metaspace: 2843K->2843K(1056768K)], 0.0047325 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[GC (Allocation Failure) --[PSYoungGen: 4471K->4471K(6144K)] 16760K->16760K(19968K), 0.0007396 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure) [PSYoungGen: 4471K->0K(6144K)] [ParOldGen: 12289K->361K(8192K)] 16760K->361K(14336K), [Metaspace: 2843K->2843K(1056768K)], 0.0072134 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[B@470e2030
5
循环结束：5
null
null
null
null
[B@470e2030
Heap
 PSYoungGen      total 6144K, used 4378K [0x00000007bf980000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 5632K, 77% used [0x00000007bf980000,0x00000007bfdc6808,0x00000007bff00000)
  from space 512K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007bff80000)
  to   space 512K, 0% used [0x00000007bff80000,0x00000007bff80000,0x00000007c0000000)
 ParOldGen       total 8192K, used 361K [0x00000007bec00000, 0x00000007bf400000, 0x00000007bf980000)
  object space 8192K, 4% used [0x00000007bec00000,0x00000007bec5a6f0,0x00000007bf400000)
 Metaspace       used 2850K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 304K, capacity 386K, committed 512K, reserved 1048576K
Disconnected from the target VM, address: '127.0.0.1:65198', transport: 'socket'
    
    
```



如果在垃圾回收时发现内存不足，在回收软引用所指向的对象时，**软引用本身不会被清理**

如果想要**清理软引用**，需要使**用引用队列**，软引用也是占内存的，虽然不大。

**大概思路为：**查看引用队列中有无软引用，如果有，则将该软引用从存放它的集合中移除（这里为一个list集合）

```
import java.lang.ref.Reference;
import java.lang.ref.ReferenceQueue;
import java.lang.ref.SoftReference;
import java.util.ArrayList;
import java.util.List;

/**
 * 演示软引用, 配合引用队列
 * 
 * -Xmx20m
 */
public class Demo2_4 {
    private static final int _4MB = 4 * 1024 * 1024;

    public static void main(String[] args) {
        List<SoftReference<byte[]>> list = new ArrayList<>();

        // 引用队列
        ReferenceQueue<byte[]> queue = new ReferenceQueue<>();

        for (int i = 0; i < 5; i++) {
            // 关联了引用队列， 当软引用所关联的 byte[]被回收时，自动触发，软引用自己会加入到 queue 中去
            SoftReference<byte[]> ref = new SoftReference<>(new byte[_4MB], queue);
            System.out.println(ref.get());
            list.add(ref);
            System.out.println(list.size());
        }

        // 从队列中获取无用的 软引用对象，并手动移除
        Reference<? extends byte[]> poll = queue.poll();
        while( poll != null) {
            list.remove(poll);
            poll = queue.poll();
        }

        System.out.println("===========================");
        for (SoftReference<byte[]> reference : list) {
            System.out.println(reference.get());
        }

    }
}
```

##### 弱引用

只有弱引用引用该对象时，在垃圾回收时，**无论内存是否充足**，都会回收**弱引用对象所引用的对象**

- 当GC Root指向弱引用对象，弱引用对象引用了一个对象，在触发了垃圾回收,无论内存是否充足，都会**回收弱引用所引用的对象**
- 可以配合引用队列来释放弱引用自身
- 如上图如果B对象不再引用A3对象，则A3对象会被回收

**弱引用的使用和软引用类似**，只是将 **SoftReference 换为了 WeakReference**



```

/**
 * 演示弱引用
 * -Xmx20m -XX:+PrintGCDetails -verbose:gc
 */
public class Demo2_5 {
    private static final int _4MB = 4 * 1024 * 1024;

    public static void main(String[] args) {
        //  list --> WeakReference --> byte[]
        List<WeakReference<byte[]>> list = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            WeakReference<byte[]> ref = new WeakReference<>(new byte[_4MB]);
            list.add(ref);
            for (WeakReference<byte[]> w : list) {
                System.out.print(w.get()+" ");
            }
            System.out.println();

        }
        System.out.println("循环结束：" + list.size());
    }
}

```

##### **虚引用**（Cleaner 类和 reference handler）

**虚引用和终结器引用 必须配合 引用队列使用。**

必须配合引用队列使用，主要配合 ByteBuffffer 使用，被引用对象回收时，会将虚引用入队，由 Reference Handler 线程调用虚引用相关方法释放直接内存

- 创建ByteBuffer对象时，就会创建一个Cleaner虚引用对象，并把ByteBuffer对象直接内存地址传给 Cleaner虚引用对象。当引用的对象ByteBuffer被垃圾回收以后，虚引用对象Cleaner就会被放入引用队列中，然后调用Cleaner的clean方法来释放直接内存

   虚引用作用的一个体现是**释放直接内存所分配的内存**，

- 如上图，B对象不再引用ByteBuffer对象，ByteBuffer就会被回收。但是直接内存中的内存还未被回收。这时需要将虚引用对象Cleaner放入引用队列中，然后调用它的clean方法来释放直接内存

##### 终结器引用 

无需手动编码，但其内部配合引用队列使用，在垃圾回收时，终结器引用入队（被引用对象暂时没有被回收），再由 Finalizer 线程通过终结器引用找到被引用对象并调用它的 finalize方法，第二次 GC 时才能回收被引用对象。

**只有执行了 finalize方法 ，此对象才能被垃圾回收，不然这个虽然不被引用了，但是一直没有被回收，已被废弃**

- 所有的类都继承自Object类，Object类有一个finalize方法。当某个对象重写了finalize方法，不再被其他的对象所引用时，会先创建终结器引用对象并放入引用队列中。会有一个优先级很低的 finalize线程查看这个队列，然后根据终结器引用对象找到它所引用的对象，然后调用该对象的finalize方法。调用以后，该对象就可以被垃圾回收了

- 如上图，B对象不再引用A4对象。这时终结器对象就会被放入引用队列中，在引用队列会根据它，找到它所引用的对象。然后调用被引用对象的finalize方法，该对象就可以被垃圾回收了

  

##### 引用队列

- 软引用和弱引用**可以配合**引用队列
  - 在**弱引用**和**虚引用**所引用的对象被回收以后，会自动将这些引用放入引用队列中(如果代码里定义了引用队列的话），方便一起回收这些软/弱引用对象
- 虚引用和终结器引用**必须配合**引用队列
  - 虚引用和终结器引用在使用时会关联一个引用队列，不需要自定义代码实现



### 2、垃圾回收算法

#### 标记-清除

![](057_标记清除.png)

**定义**：标记清除算法顾名思义，是指在虚拟机执行垃圾回收的过程中，先采用标记算法确定可回收对象，然后垃圾收集器根据标识清除相应的内容，给堆内存腾出相应的空间

- 这里的腾出内存空间并不是将内存空间的字节清0，而是记录下这段内存的起始结束地址，下次分配内存的时候，会直接**覆盖**这段内存

- **优点**：速度较快
- **缺点**：**容易产生大量的内存碎片**，可能无法满足大对象的内存分配（如数组，需要连续的存储空间），一旦导致无法分配对象，那就会导致jvm启动gc，一旦启动gc，我们的应用程序就会暂停，这就导致应用的响应速度变慢



#### 标记-整理

![](058_标记整理.png)



- **优点**：速度慢
- **缺点**：不会有内存碎片

标记-整理 会将不被GC Root引用的对象回收，清除其占用的内存空间。然后整理剩余的对象，可以有效避免因内存碎片而导致的问题，但是因为整体需要消耗一定的时间，对象发生移动，对象的引用地址需要变更，所以效率较低

#### 复制

![](059_复制1.jpg)





![](059_复制2.jpg)
- **优点**：不会有内存碎片

- **缺点**：需要占用双倍内存空间

  将内存分为等大小的两个区域，FROM和TO（TO一开始为空）。先将被GC Root引用的对象从FROM 拷贝到TO中，再回收不被GC Root引用的对象。然后交换FROM和TO。这样也可以避免内存碎片的问题，但是会占用双倍的内存空间。



#### 总结
实际这几种垃圾回收算法都会使用

### 3、分代回收(就是指堆内存)

根据对象的生命周期的不同，进行不同的垃圾回收策略。



1. 对象首先分配在伊甸园区域

![](062_01.jpg)

2. 当新生代的内存不足时，就会进行一次垃圾回收，这时的回收叫做 **Minor GC**。Minor GC 会将**伊甸园和幸存区FROM**存活的对象**用复制算法**放入幸存区 TO中， 并让其寿命加1，再交换两个幸存区。

   minor gc 会引发 stop the world，暂停其它用户的线程（因为对象的地址被移动了，所以要暂停其他线程），等垃圾回收结束

   ![](062_02.jpg)

   ![](062_03.jpg)

3. 再次创建对象，若新生代的伊甸园又满了，则会**再次触发 Minor GC**，不仅会回收伊甸园中的垃圾，**还会回收幸存区中的垃圾**，再将活跃对象复制到幸存区TO中。回收以后会交换两个幸存区，并让幸存区中的对象**寿命加1**

   ![](062_05.jpg)

4. 当对象寿命超过阈值时，会晋升至老年代，最大寿命是15（4bit）

   ![](062_06.jpg)

5. 当老年代空间不足，会先尝试触发 minor gc，如果之后空间仍不足，那么触发 full gcc，STW的时间更长。如果老年代的空也不够，就触发OutOfMemory

   ![](062_07.jpg)



### 4、JVM参数

堆初始大小 -Xms

堆最大大小 -Xmx 或 -XX:MaxHeapSize=size

新生代大小 -Xmn （指初始最大同时指定）或    (-XX:NewSize=size + -XX:MaxNewSize=size )

幸存区比例（动态） -XX:InitialSurvivorRatio=ratio 和 -XX:+UseAdaptiveSizePolicy

幸存区比例（默认是8，如果新生代有10m，代表8m是伊甸园的） -XX:SurvivorRatio=ratio  

晋升阈值(新生代晋升老年代，默认和垃圾回收器有关，如15,如6) -XX:MaxTenuringThreshold=threshold

晋升详情 -XX:+PrintTenuringDistribution

GC详情 -XX:+PrintGCDetails -verbose:gc

FullGC 前 MinorGC -XX:+ScavengeBeforeFullGC



启动一个main方法，什么都不写

```
// -Xms20M -Xmx20M -Xmn10M -XX:+UseSerialGC -XX:+PrintGCDetails -verbose:gc -XX:-ScavengeBeforeFullGC
```

可以看到新生代-Xmn分配了10M，但是只有9M，因为默认8(eden:1(from):1(to) ，幸存区to的1m默认是不能用的。 

 def new generation ：新生代

 tenured generation ：老年代

[Full GC (Ergonomics) 就是 full gc，[GC (Allocation Failure) 就是minor gc

```
Heap
 def new generation   total 9216K, used 1896K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
  eden space 8192K,  23% used [0x00000007bec00000, 0x00000007bedda120, 0x00000007bf400000)
  from space 1024K,   0% used [0x00000007bf400000, 0x00000007bf400000, 0x00000007bf500000)
  to   space 1024K,   0% used [0x00000007bf500000, 0x00000007bf500000, 0x00000007bf600000)
 tenured generation   total 10240K, used 0K [0x00000007bf600000, 0x00000007c0000000, 0x00000007c0000000)
   the space 10240K,   0% used [0x00000007bf600000, 0x00000007bf600000, 0x00000007bf600200, 0x00000007c0000000)
 Metaspace       used 3131K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 344K, capacity 388K, committed 512K, reserved 1048576K

```

#### GC 分析

##### 大对象处理策略

当遇到一个**较大的对象**时，就算新生代的**伊甸园**为空，也**无法容纳该对象**时，会将该对象**直接晋升为老年代**

##### 线程内存溢出

某个线程的内存溢出了而抛异常（out of memory），不会让其他的线程结束运行

这是因为当一个线程**抛出OOM异常后**，**它所占据的内存资源会全部被释放掉**，从而不会影响其他线程的运行，**进程依然正常**

### 4、垃圾回收器



几种垃圾回收器模型

#### 串行的 (单线程的垃圾回收器)

   堆内存较小，适合个人电脑

​    ![](069_01_串行.jpg)

####  吞吐量优先(多线程的垃圾回收器)



java -server -XX:+ScavengeBeforeFullGC -XX:+PrintGCDetails -XX:+PrintTenuringDistribution -XX:+PrintGCTimeStamps -Dspring.profiles.active=prd -Dfile.encoding=utf-8 -Djava.net.preferIPv6Addresses=false -Djava.net.preferIPv4Stack=true -Duser.timezone=Asia/Shanghai -jar -Xms2048m -Xmx2048m -Xss512m -XX:SurvivorRatio=8 -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=50 -XX:+UseParallelGC -XX:+UseParallelOldGC -XX:ParallelGCThreads=4 -XX:+UseAdaptiveSizePolicy -XX:MaxNewSize=128m -XX:NewSize=128m -Dnacos.config.server-addr=10.32.11.32:8849 -Dnacos.discovery.server-addr=10.32.11.32:8849 ${project.build.finalName}.jar

```
java -server -XX:+ScavengeBeforeFullGC -
XX:+PrintGCDetails 
-XX:+PrintTenuringDistribution 
-XX:+PrintGCTimeStamps

-Dspring.profiles.active=prd 
-Dfile.encoding=utf-8 
-Djava.net.preferIPv6Addresses=false 
-Djava.net.preferIPv4Stack=true 
-Duser.timezone=Asia/Shanghai -jar 



-Xms2048m -Xmx2048m 
-Xss512m 
-XX:SurvivorRatio=8 
-XX:+UseCMSInitiatingOccupancyOnly 
-XX:CMSInitiatingOccupancyFraction=50 
-XX:+UseParallelGC -XX:+UseParallelOldGC 
-XX:ParallelGCThreads=4 
-XX:+UseAdaptiveSizePolicy 
-XX:MaxNewSize=128m 
-XX:NewSize=128m



 -Dnacos.config.server-addr=10.32.11.32:8849
 -Dnacos.discovery.server-addr=10.32.11.32:8849
 ${project.build.finalName}.jar


```

   让**单位时间**内，STW 的时间最短 0.2 0.2 = 0.4，垃圾回收时间占程序运行时间比最低，这样就称吞吐量高

   堆内存较大，多核cpu

   但用户线程不能运行了

​    ![](069_02_吞吐量.jpg)

-XX:+UseParallelGC ~ -XX:+UseParallelOldGC

-XX:+UseAdaptiveSizePolicy    此参数代表 打开动态调整伊甸园与幸存区的比例的开关

-XX:GCTimeRatio=ratio          默认值99，吞吐量  1/(1+99）（吞吐量 = 运行用户代码时间 / ( 运行用户代码时间 + 垃圾收集时间 )），也就是。例如：虚拟机共运行100分钟，垃圾收集器花掉1分钟，那么吞吐量就是99%，增大堆的大小，可以减少gc数量，增加吞吐量。

-XX:MaxGCPauseMillis=ms    最大暂停毫秒数 即 每次gc花费的时间。堆越大，花费时间越长。

-XX:ParallelGCThreads=n



####  响应时间优先(多线程的垃圾回收器，如CMS) 

   堆内存较大，多核 cpu

   尽可能让**单次** STW 的时间最短 0.1 0.1 0.1 0.1 0.1 = 0.5s

​    ![](071_cms.png)

**并行收集**：指多条垃圾收集线程并行工作，但此时**用户线程仍处于等待状态**。

**并发收集**：指用户线程与垃圾收集线程**同时工作**（不一定是并行的可能会交替执行）。**用户程序在继续运行**，而垃圾收集程序运行在另一个CPU上，可以减少 STW。

##### CMS 收集器

Concurrent Mark Sweep，一种以**响应时间短**老年代垃圾收集器

**特点**：基于**标记-清除算法**实现。并发收集、低停顿，但是会产生内存碎片，CMS收集器的内存回收过程是与用户线程一起**并发执行**的。

**应用场景**：适用于注重服务的响应速度，希望系统停顿时间最短，给用户带来更好的体验等场景下。如web程序、b/s服务

**CMS收集器的运行过程分为下列4步：**

**初始标记**：标记GC Roots能直接到的对象。速度很快但是**仍存在Stop The World问题**

**并发标记**：进行GC Roots Tracing 的过程，找出存活对象且用户线程可并发执行,剩余的垃圾找出来？

**重新标记**：为了**修正并发标记期间**因用户程序继续运行而导致标记产生变动的那一部分对象的标记记录。仍然存在Stop The World问题

**并发清除**：对标记的对象进行清除回收



```
-XX:+UseConcMarkSweepGC ~ -XX:+UseParNewGC ~ SerialOld 
```

ConcMarkSweepGC CMS 是工作在老年代的垃圾回收器，与之配合的UseParNewGC 是工作在新生代。CMS有可能会因为内存碎片化的问题退化到SerialOld 单线程（串行的垃圾回收器使用的），垃圾回收时间会非常慢。



```
-XX:ParallelGCThreads=n ~ -XX:ConcGCThreads=threads       
```

  -XX:ParallelGCThreads=4 ~ -XX:ConcGCThreads=1  代表1个线程gc，3个线程给用户，按照这样的比例。





```
-XX:CMSInitiatingOccupancyFraction=percent 
```
这个参数就是何时触发垃圾回收。不能堆内存占100%的时候去GC，因为并发清理时候也会产生浮动垃圾，这些浮动垃圾没地方存。这样说懂了吧。



```
-XX:+CMSScavengeBeforeRemark
```

在重新标记的阶段，有可能新生代的对象引用了老年代的对象。如果重新标记要扫描整个堆，新生代的个数比较多，而且本身就有可能已经是垃圾了。就算从新生代找到老年代，就算老年代被引用了，但实际上最终还是要被回收掉。

XX:+CMSScavengeBeforeRemark，在重新标记之前，先执行一次ygc，回收掉年轻带的对象无用的对象，并将对象放入幸存带或晋升到老年代，这样再进行年轻带扫描时，只需要扫描幸存区的对象即可，一般幸存带非常小，这大大减少了扫描时间。



https://juejin.cn/post/6844903782107578382







#### G1

##### **定义**：

Garbage First

JDK 9以后默认使用，而且替代了CMS 收集器

##### 适用场景

- 同时注重吞吐量和低延迟（响应时间）
- 超大堆内存（内存大的），会将堆内存划分为多个**大小相等**的区域
- 整体上是**标记-整理**算法，两个区域之间是**复制**算法

相关 JVM 参数

-XX:+UseG1GC 

-XX:G1HeapRegionSize=size 

-XX:MaxGCPauseMillis=time



  ![](073_g1.png)

E：伊甸园 S：幸存区 O：老年代

#####  1.Young Collection

  ![](073_g1_01.png)

  ![](073_g1_02.png)

  ![](073_g1_03.png)

##### 2.Young Collection + CM

CM：并发标记

- 在 Young GC 时会**对 GC Root 进行初始标记**
- 在老年代**占用堆内存的比例**达到阈值时，对进行并发标记（不会STW），阈值可以根据用户来进行设定

![](076_g1_01.png)

https://juejin.cn/post/7010034105165299725

### 5、直接内存

  ![](directbuff-io.png)

  ![](directbuff-nio.png)

并不属于JVM中的内存结构，不由JVM进行管理。是虚拟机的系统内存

常见于NIO操作时，用于数据缓冲区，分配回收成本较高，但读写性能高，不受JVM内存回收管理

