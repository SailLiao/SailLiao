---
title: Java方法执行流程解析
date: 2020-12-24 20:50:27
tags: Java
cover: https://sailliao.oss-cn-beijing.aliyuncs.com/img/wallhaven-g7gvo3.png
---

Java程序运行时，必须经过编译和运行两个步骤。首先将后缀名为.java的源文件进行编译，最终生成后缀名为.class的字节码文件。
然后Java虚拟机将编译好的字节码文件加载到内存（这个过程被称为类加载，是由加载器完成的），然后虚拟机针对加载到内存的java类进行解释执行，显示结果。

# 类的加载过程

并不是说随便将一个文件的后缀修改为 .class 就能被加载运行，类的加载过程非常复杂，简单来说分为以下几个部分

> 加载
> 验证
> 准备
> 解析
> 初始化

## 加载

加载的作用主要是从jar包或者war包里面找到并且将类的二进制数据 .class 文件加载到Java的方法区。所以有可能你的项目需要加载的类非常多，造成方法区的扩容，这个时候是会 Stop The Word 的。

## 验证

肯定不是只要是.class文件的文件都能被加载，有一个验证的过程，可以通过将 notepad++ 安装16进制的插件来查看 .class 文件，例如前面8个字节是 cafe baby, 紧跟着的 2 + 2 个字节是版本号之类的。

## 准备

从现在开始，将为一些变量分配内存，并且将其初始化为默认值，此时，实例对象还没有分配内存，这些都是在方法区上进行的。

```Java
// 代码1
public class A {
    static int a ;
    public static void main(String[] args) {
        System.out.println(a);
    }
}
// 代码2
public class A {
    public static void main(String[] args) {
        int a ;
        System.out.println(a);
    }
}
```

例如上面的代码，代码1 可以正常输出0，代码2会无法通过编译。
这是因为局部变量不会像成员变量那样，在准备阶段会被赋值，成员变量会有两次赋值的机会，一次是在准备阶段，一次是在初始化阶段（执行构造方法之类的）

## 解析

解析是非常非常非常重要的一环，负责把**符号引用**转变为**直接引用**

### 符号引用

我们再使用 javap 查看字节码的时候有时候会发现类似这种

```bash
Ljava/lang/String;
```

可以理解，在编译时候，我们并不知道这个实际内存地址

### 直接引用

字面意思，直接指向目标的指针

解析阶段负责把整个类激活，过程主要包括
1. 类或接口的解析
2. 类方法解析
3. 接口方法解析
4. 字段解析

在这个阶段常发生的异常
1. java.lang.NoSuchFieldError 根据继承关系从下往上，找不到相关字段时的报错。
2. java.lang.IllegalAccessError 字段或者方法，访问权限不具备时的错误。
3. java.lang.NoSuchMethodError 找不到相关方法时的错误。

解析阶段就保证了相互引用的完整性，**把组合与继承推到了运行时**

## 初始化

如果前面的阶段都正常的话，接下来就该初始化成员变量，到了这一步，才真正开始执行一些字节码。
猜想一下，下面的代码，会输出什么？

```Java
public class A{
	static int a = 0;
	static {
		a = 1;
		b = 1;
	}
	static int b = 0;
	public stati cvoid main(String[] args){
		System.out.println(a);
		System.out.println(b);
	}
}
```
输出的是 1 0 

还有另外的如果有父类，会先执行父类的初始化方法

# 执行过程

以下面的程序为例

```Java
public class A {
    private B b = new B();

    public static void main(String[] args) {
        A a = new A();
        long num = 4321;
        long ret = a.b.test(num);
        System.out.println(ret);
    }
}

class B {
    private int a = 1234;
    static long C = 1111;

    public long test(long num) {
        long ret = this.a + num + C;
        return ret;
    }
}
```

经过 javac 和 javap 之后可以得到字节码

## javac

1. javac -g:lines 强制生成 LineNumberTable。
2. javac -g:vars  强制生成 LocalVariableTable。
3. javac -g 生成所有的 debug 信息。


## javap

javap 是 JDK 自带的反解析工具。它的作用是将 .class 字节码文件解析成可读的文件格式。

在使用 javap 时我一般会添加 -v 参数，尽量多打印一些信息。同时，我也会使用 -p 参数，打印一些私有的字段和方法。使用起来大概是这样

```bash
javap -p -v HelloWorld
```

## 对象的创建

### 对象的创建方式有

1. 使用 Class 的 newInstance 方法。
2. 使用 Constructor 类的 newInstance 方法。
3. 反序列化。
4. 使用 Object 的 clone 方法。

当虚拟机遇到一条 new 指令时，首先会检查这个指令的参数能否在常量池中定位一个符号引用。然后检查这个符号引用的类字节码是否加载、解析和初始化。如果没有，将执行对应的类加载过程。


## 执行过程

拿我们上面的代码来说，执行 A 代码，在调用 private B b = new B() 时，就会触发 B 类的加载。

![](https://s0.lgstatic.com/i/image3/M01/61/AD/CgpOIF4ezuOAK_6bAACFY5oeX-Y174.jpg)

A 和 B 会被加载到元空间的方法区，进入 main 方法后，将会交给执行引擎执行。
这个执行过程是在栈上完成的，其中有几个重要的区域，包括虚拟机栈、程序计数器等。
接下来我们详细看一下虚拟机栈上的执行过程。

编译
```bash
javac -g:lines -g:vars A.java
```

使用 javap 查看字节码
```bash
javap -p -v A

Classfile /D:/桌面/A.class
  Last modified 2020-12-25; size 619 bytes
  MD5 checksum 27fd5b9739725d25e9d2a22176642fba
public class A
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #12.#30        // java/lang/Object."<init>":()V
   #2 = Class              #31            // B
   #3 = Methodref          #2.#30         // B."<init>":()V
   #4 = Fieldref           #5.#32         // A.b:LB;
   #5 = Class              #33            // A
   #6 = Methodref          #5.#30         // A."<init>":()V
   #7 = Long               4321l
   #9 = Methodref          #2.#34         // B.test:(J)J
  #10 = Fieldref           #35.#36        // java/lang/System.out:Ljava/io/PrintStream;
  #11 = Methodref          #37.#38        // java/io/PrintStream.println:(J)V
  #12 = Class              #39            // java/lang/Object
  #13 = Utf8               b
  #14 = Utf8               LB;
  #15 = Utf8               <init>
  #16 = Utf8               ()V
  #17 = Utf8               Code
  #18 = Utf8               LineNumberTable
  #19 = Utf8               LocalVariableTable
  #20 = Utf8               this
  #21 = Utf8               LA;
  #22 = Utf8               main
  #23 = Utf8               ([Ljava/lang/String;)V
  #24 = Utf8               args
  #25 = Utf8               [Ljava/lang/String;
  #26 = Utf8               a
  #27 = Utf8               num
  #28 = Utf8               J
  #29 = Utf8               ret
  #30 = NameAndType        #15:#16        // "<init>":()V
  #31 = Utf8               B
  #32 = NameAndType        #13:#14        // b:LB;
  #33 = Utf8               A
  #34 = NameAndType        #40:#41        // test:(J)J
  #35 = Class              #42            // java/lang/System
  #36 = NameAndType        #43:#44        // out:Ljava/io/PrintStream;
  #37 = Class              #45            // java/io/PrintStream
  #38 = NameAndType        #46:#47        // println:(J)V
  #39 = Utf8               java/lang/Object
  #40 = Utf8               test
  #41 = Utf8               (J)J
  #42 = Utf8               java/lang/System
  #43 = Utf8               out
  #44 = Utf8               Ljava/io/PrintStream;
  #45 = Utf8               java/io/PrintStream
  #46 = Utf8               println
  #47 = Utf8               (J)V
{
  private B b;
    descriptor: LB;
    flags: ACC_PRIVATE

  public A();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: new           #2                  // class B
         8: dup
         9: invokespecial #3                  // Method B."<init>":()V
        12: putfield      #4                  // Field b:LB;
        15: return
      LineNumberTable:
        line 1: 0
        line 2: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      16     0  this   LA;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=6, args_size=1
         0: new           #5                  // class A
         3: dup
         4: invokespecial #6                  // Method "<init>":()V
         7: astore_1
         8: ldc2_w        #7                  // long 4321l
        11: lstore_2
        12: aload_1
        13: getfield      #4                  // Field b:LB;
        16: lload_2
        17: invokevirtual #9                  // Method B.test:(J)J
        20: lstore        4
        22: getstatic     #10                 // Field java/lang/System.out:Ljava/io/PrintStream;
        25: lload         4
        27: invokevirtual #11                 // Method java/io/PrintStream.println:(J)V
        30: return
      LineNumberTable:
        line 5: 0
        line 6: 8
        line 7: 12
        line 8: 22
        line 9: 30
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      31     0  args   [Ljava/lang/String;
            8      23     1     a   LA;
           12      19     2   num   J
           22       9     4   ret   J
}


javap -p -v B

Classfile /D:/桌面/B.class
  Last modified 2020-12-25; size 446 bytes
  MD5 checksum 21b0c5e29d492eb13cc7593cdf956061
class B
  minor version: 0
  major version: 52
  flags: ACC_SUPER
Constant pool:
   #1 = Methodref          #7.#24         // java/lang/Object."<init>":()V
   #2 = Fieldref           #6.#25         // B.a:I
   #3 = Fieldref           #6.#26         // B.C:J
   #4 = Long               1111l
   #6 = Class              #27            // B
   #7 = Class              #28            // java/lang/Object
   #8 = Utf8               a
   #9 = Utf8               I
  #10 = Utf8               C
  #11 = Utf8               J
  #12 = Utf8               <init>
  #13 = Utf8               ()V
  #14 = Utf8               Code
  #15 = Utf8               LineNumberTable
  #16 = Utf8               LocalVariableTable
  #17 = Utf8               this
  #18 = Utf8               LB;
  #19 = Utf8               test
  #20 = Utf8               (J)J
  #21 = Utf8               num
  #22 = Utf8               ret
  #23 = Utf8               <clinit>
  #24 = NameAndType        #12:#13        // "<init>":()V
  #25 = NameAndType        #8:#9          // a:I
  #26 = NameAndType        #10:#11        // C:J
  #27 = Utf8               B
  #28 = Utf8               java/lang/Object
{
  private int a;
    descriptor: I
    flags: ACC_PRIVATE

  static long C;
    descriptor: J
    flags: ACC_STATIC

  B();
    descriptor: ()V
    flags:
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: sipush        1234
         8: putfield      #2                  // Field a:I
        11: return
      LineNumberTable:
        line 12: 0
        line 13: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      12     0  this   LB;

  public long test(long);
    descriptor: (J)J
    flags: ACC_PUBLIC
    Code:
      stack=4, locals=5, args_size=2
         0: aload_0
         1: getfield      #2                  // Field a:I
         4: i2l
         5: lload_1
         6: ladd
         7: getstatic     #3                  // Field C:J
        10: ladd
        11: lstore_3
        12: lload_3
        13: lreturn
      LineNumberTable:
        line 17: 0
        line 18: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      14     0  this   LB;
            0      14     1   num   J
           12       2     3   ret   J

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=2, locals=0, args_size=0
         0: ldc2_w        #4                  // long 1111l
         3: putstatic     #3                  // Field C:J
         6: return
      LineNumberTable:
        line 14: 0
}
```

注意这一行
```bash
1: invokespecial #1                  // Method java/lang/Object."<init>":()V
```
可以看到对象的初始化，首先是调用了 Object 类的初始化方法。注意这里是 <init> 而不是 <cinit>。
B 中的这一样
```
#2 = Fieldref           #6.#25         // B.a:I
```
其实是拼接了 #6 和 #25 的内容

```bash
#6 = Class              #27            // B
#27 = NameAndType       #8:#9         // a:I
...
#8 = Utf8               a
#9 = Utf8               I
```
:I 这种字符其实是有意义的

* B 基本类型 byte
* C 基本类型 char
* D 基本类型 double
* F 基本类型 float
* I 基本类型 int
* J 基本类型 long
* S 基本类型 short
* Z 基本类型 boolean
* V 特殊类型 void
* L 对象类型，以分号结尾，如 Ljava/lang/Object;
* [Ljava/lang/String; 数组类型，每一位使用一个前置的"["字符来描述

## test 函数执行过程

```bash
public long test(long);
    descriptor: (J)J
    flags: ACC_PUBLIC
    Code:
        stack=4, locals=5, args_size=2
            0: aload_0
            1: getfield      #2                  // Field a:I
            4: i2l
            5: lload_1
            6: ladd
            7: getstatic     #3                  // Field C:J
        10: ladd
        11: lstore_3
        12: lload_3
        13: lreturn
        LineNumberTable:
        line 17: 0
        line 18: 12
        LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      14     0  this   LB;
            0      14     1   num   J
            12       2     3   ret   J
```

### stack=4
表示操作数栈深度是4，JVM 运行时，会根据这个数值，来分配栈帧中操作栈的深度。

### locals=5
它的单位是 Slot（槽），可以被重用。其中存放的内容，包括：
* this
* 方法参数
* 异常处理器的参数
* 方法体中定义的局部变量

### args_size=2
比较好理解，就是方法的参数个数，因为每个方法都有一个隐藏参数 this，所以这里的数字是 2。

## 字节码执行过程

```bash
 0: aload_0
```
把第 1 个引用型局部变量推到操作数栈，这里的意思是把 this 装载到了操作数栈中。
对于 static 方法，aload_0 表示对方法的第一个参数的操作。
```bash
1: getfield      #2                  // Field a:I
```
将栈顶的指定的对象的第 2 个实例域（Field）的值，压入栈顶。#2 就是指的我们的成员变量 a。
```bash
4: i2l
```
将栈顶 int 类型的数据转化为 long 类型，这里就涉及我们的隐式类型转换了。图中的信息没有变动，不再详解介绍。
```bash
5: lload_1
```
将第一个局部变量入栈。也就是我们的参数 num。这里的 l 表示 long，同样用于局部变量装载。你会看到这个位置的局部变量，一开始就已经有值了。
```bash
6: ladd
```
把栈顶两个 long 型数值出栈后相加，并将结果入栈。
```bash
7: getstatic     #3                  // Field C:J
```
根据偏移获取静态属性的值，并把这个值 push 到操作数栈上。
```bash
10: ladd
```
再次执行 ladd。
```bash
11: lstore_3
```
把栈顶 long 型数值存入第 4 个局部变量。
还记得我们上面的图么？slot 为 4，索引为 3 的就是 ret 变量。
```bash
12: lload_3
```
正好与上面相反。上面是变量存入，我们现在要做的，就是把这个变量 ret，压入虚拟机栈中。
```bash
13: lreturn
```
从当前方法返回 long。

到此为止，我们的函数就完成了相加动作，执行成功了。

# 文章来源
[java方法执行流程解析](https://www.cnblogs.com/minikobe/p/11512628.html)
[第04讲：动手实践：从栈帧看字节码是如何在 JVM 中进行流转的](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=31#/detail/pc?id=1028)