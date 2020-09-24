---
layout: post
title:  "无用知识: 一个Java类最多可以有多少个成员变量"
date:   2020-09-23 18:24:26 +0800
categories: jvm
---

Android中, 单个dex文件有一个65536个方法数的限制: [https://developer.android.com/studio/build/multidex?hl=zh-cn](https://developer.android.com/studio/build/multidex?hl=zh-cn)

原因貌似是dex文件对每个方法都指定了一个id, 用了2个byte来保存这些id, 因此有了这个有点小的限制, 也许Android系统设计者本身就没有预见到会发展得这么成功

今天在查看Java的.class文件的结构的时候, 发现了类似的东西: class文件中有一个“常量池”, 这个常量池中的个数constant_pool_count也是用2个byte来保存的, 其中index 0保留, 从1开始. 所以这个常量池中元素个数的上限也是65534个

这个常量池其实和我第一直觉上理解的"常量池“不太一样. 从字面意义上, 我会觉得类似于"hello"这样的字面值常量会保存在里面, 事实上也是这样的. 但是除此之外, 类的名称, 字段/方法的名称, 字段的类型/方法的签名的全都保存在这个常量池当中

一开始我觉得挺奇怪的, 为什么class文件会保留字段的具体名称, 既然计算机只认识0和1, 显然没有必要保留名称, 不过想了下这样做是必须的, class文件是相互独立的, class文件如果要编译成本地代码在硬件上运行, 其实还需要经过一个link阶段, 在link之前不同的class文件之间的方法调用和字段访问都应该用名称

举个例子, 假如说class A依赖于jdk中的class B, 那么当我们更新jdk的时候, 如果class B中新增了方法, 如果对B的方法的调用是用一个id或index来标记的, 没有办法保证B中的方法的id/index是保持稳定的, 只有名称/方法签名是稳定的

另一个原因是, 如果要实现反射的话, 那保存字段和方法的名称就是必须的了

以这个类为例: 

    public class Large0 {
      int a0 = 1;
    }

    javac Large0.java
    javap -v Large0.class

它的类文件中, 常量池的部分为:

    Constant pool:
    #1 = Methodref          #4.#13         // java/lang/Object."<init>":()V
    #2 = Fieldref           #3.#14         // Large0.a0:I
    #3 = Class              #15            // Large0
    #4 = Class              #16            // java/lang/Object
    #5 = Utf8               a0
    #6 = Utf8               I
    #7 = Utf8               <init>
    #8 = Utf8               ()V
    #9 = Utf8               Code
    #10 = Utf8               LineNumberTable
    #11 = Utf8               SourceFile
    #12 = Utf8               Large0.java
    #13 = NameAndType        #7:#8          // "<init>":()V
    #14 = NameAndType        #5:#6          // a0:I
    #15 = Utf8               Large0
    #16 = Utf8               java/lang/Object

其中字段a0保存在index=2的FieldRef类型, 之所以会生成这个FieldRef类型, 是因为代码中对a0进行了初始赋值为1, 在编译成字节码的时候, 这个初始化会进入构造方法中, 这里对这个成员进行了引用  
FieldRef类型占用5个byte, 第一个byte注明该常量的类型为字段引用(9), 后面跟着两个各自占用2个byte的index, 指向字段所在的这个类(#3, Large0)和字段的NameAndType(#4)这两个常量, 其中NameAndType类型的常量又指向两个UTF8类型的常量(#5, #6)  


所以一个int a0 = 1;这样的申明, 会带来: 一个Const_fieldRef, 一个Const_Class(共享), 一个Const_NameAndType, 一个记录name的UTF8, 一个记录type(共享)的UTF8
这样它独占的的常量就有3个

所以一个类中用到的成员变量的一个上限是65535 / 3 = 21845

实际测试一下  
通过一个shell文件生成任意长度的Large.java文件

    filename="$1.java"
    echo -n > $filename
    echo "public class $1 {" >> $filename
    for ((i = 0; i < $2; i++))
    do
      echo "int a$i = 1;" >> $filename
    done
    echo "}" >> $filename

然后编译这个java文件  
在手动二分查找(=,-)之后发现这样的a成员变量最多是13106个, 13107个的时候就报“代码过长”错误了

对13106个ivar的这个class文件使用javap进行反汇编后发现它实际上有39331个常量, 并没有到达上限

    grep "#\d* =" large_13106.javap | tail
    #39322 = NameAndType        #26209:#13111 // a13098:I
    #39323 = NameAndType        #26210:#13111 // a13099:I
    #39324 = NameAndType        #26211:#13111 // a13100:I
    #39325 = NameAndType        #26212:#13111 // a13101:I
    #39326 = NameAndType        #26213:#13111 // a13102:I
    #39327 = NameAndType        #26214:#13111 // a13103:I
    #39328 = NameAndType        #26215:#13111 // a13104:I
    #39329 = NameAndType        #26216:#13111 // a13105:I
    #39330 = Utf8               Large
    #39331 = Utf8               java/lang/Object

所以应该是有别的限制  
实际上javac报的错误是“代码过长/code too large, 它指的不是源代码长度过长, 而是一个Java方法编译后, 它的Code部分超过了长度限制
Code的长度限制code_length是用一个u4来记录的, 也就是最大为2^32-1, 但是jvm规范同时又规定一个方法最大长度不超过2^16  
不管这个设定为什么浪费了2个字节, 如果看&ltinit&gl这个构造方法, 会发现它在执行了

    aload_0
    invokespecial #1
后(占用4个byte的), 后面对每个成员变量进行了初始化

      aload_0
      iconst_1
      putfield      #10012             // Field a10010:I

这是因为我写shell脚本的时候给每个成员变量都初始化了成了1  
每个初始化都占用了5个byte, 这样(65535 - 4) / 5 = 13106.2  
而且前面

所以如果把成员变量的初始化去掉, 最多可以有65521个成员变量, 如果再多一个, javac报的错误是“常量过多/too many constants”, 这里的常量应该是指常量池的常量