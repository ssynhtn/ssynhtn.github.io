---
layout: post
title:  "bash中获取大小端"
date:   2020-08-29 18:21:26 +0800
categories: shell
---

首先明确一下一般说大小端是指byte层面上的, 他们都说因为内存地址是byte level的, 不过很明显这个问题在bit level也是存在的

little endian: 较低位的数字放在地址较低的内存上  
big endian: 较低位的数字放在地址较高的内存上

pros & cons  
little endian: 逻辑合理  
big endian: 如果我们按照习惯从左到右排列内存, 左为低地址右为高地址(我们想象数组的时候一般都是这样), 那么按照书写顺序, 高位的数字应该是在左边的, 也就是低地址内存

我以前觉得big endian才简单, 现在想想还是little endian好啊

今天我发现hexdump的输出会发映出大小端: 

    if [ `printf '\x00\x01' | hexdump -d | head -1 | sed 's/[0-9]*\s*\([0-9]*\).*/\1/'` = 00256 ] ; then echo small ; else echo big ; fi

首先解释一下上面这个命令是怎么工作的
1, printf '\xnn' 是按照16进制输出一个byte, 因此这里输出了数值为0和1的byte(不是字符'0'和'1')  
2, hexdump 的-d选项会以2个byte为单位, 将这两个byte组成的数按十进制输出，显示为５个digit(不足5位用0填充)  
3, head -1 是取第一行, hexdump每行会在前面显示当前byte的offset, 最后会多输出一行显示总byte数　　  
4, sed中\1取了第二个连续数字组成的substring, 也就是byte array[0, 1]的数值大小  
5, 在小端的硬件上, 这个数值应该是00256, 在大端设备上应该是00001


等我想明白之后发现上面这一整个逻辑其实和普通的C++代码判断大小端的逻辑是一样的

    #include <iostream>

    int main() {
        u_int8_t arr[2];
        arr[0] = 0;
        arr[1] = 1;

        u_int16_t res = *(u_int16_t*)arr;

        if (res == 256) {
            std::cout << "little endian" << std:: endl;
        } else {
            std::cout << "big endian" << std::endl;
        }

        return 0;
    }