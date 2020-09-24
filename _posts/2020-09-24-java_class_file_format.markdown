---
layout: post
title:  "Java class文件格式"
date:   2020-09-24 18:24:26 +0800
categories: jvm
---


<pre>
咖啡宝贝 0xcafebabe u4
minor u2
major u2  // java1 45/0x2D, java8 52/0x34
constant_pool_count u2
list of cp_info:  // 个数为constant_pool_count-1, 为啥, 因为index 0的为null/保留, 不记录, 为啥index 0的为null/保留, 因为设计者想要用0来标记不指向任何一个cp_info, 至于哪里用到了这个, 不是很清楚
  cp_info:
    tag/type u1
    data // type specific
class access flags u2
class name index u2 // constant_pool中的index
super class name index u2 // 接口的super class也用java.lang.Object占个位
interface class name count u2
interface class name indexes
field counts u2
list of field info:
  field access flags u2
  field name index u2
  field descriptor index u2
  field attributes count u2
  field attributes:
    attribute name index u2
    attribute data length u4
    attribute data
method counts u2
list of method info:
  access flags u2
  method name index u2
  method descriptor index u2
  method attributes count u2
  method attributes:
    attribute name index u2 // 比如指向Code
    attribute data length u4  // 如果name为Code, 这里记录的长度是指令长度
    attribute data  // 如果name是Code, 这里是实际指令
attributes count u2
  attribute name index u2
  attribute data length u4
  attribute data

</pre>