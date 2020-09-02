---
layout: post
title:  "LinkedList的实现"
date:   2020-08-25 18:20:26 +0800
categories: java
---

网上总是有很多类似于“HashMap实现原理”这样的文章，没办法，谁让面试的时候他们总问这些问题呢

实际上我们从阅读这些分析源码的博客文章中获得的知识已经属于四手知识，不是二手，甚至不是三手
1, 首先有人发明了hashtable，并证明了这种数据结构的性能特点
2, 然后Java库的开发者将它以Java语言实现和完善了一遍(HashTable), 再一遍(HashMap), 又一遍(java8 HashMap)
3, 接着那个博客作者详详细细地阅读，运行，解析了HashMap的源码
4, 而可怜的你则在面试之前或工作之余捧着博客仔细阅读这些"精华"知识

在我的经验中，自己实现的东西总是最好的，所以我们应该自己实现一遍
今天就从简单的List开始
ArrayList之前我写过了，今天写LinkedList吧

1， 首先我们选择一个类名，我选择GoodLinkedList，它实现List<E>接口, 我选择直接实现这个接口而不是继承AbstractList，因为我只是想要实现一个链表而已, 继承那个类还要去看它那些方法是需要覆盖的,以为那些默认的方法性能上和链表结构不符, 所以为了免去研究这个基类的任务,就直接一点吧

    public class GoodLinkedList<E> implements List<E>

2， 首先把空方法都实现了, 点击类名上的红线, 使用Alt+Enter快捷键完成
3， 加上链表的Node<E>类，目前似乎单向的链表就足够了
    static class Node<E> {
        E value;
        Node<E> next;
    }

4, 我们保存两个实例变量，head，和size，因为不然的话连获取一个长度都要遍历一遍，那个就不太好了
    Node<E> head;
    int size;
如果我们选择int来保存长度，也就意味着这个链表最多就只能保存2^31-1个元素 ～ 2^31 = ~ 2* 10^9个也就是2G个
对于绝大多数应用这够不够我不知道，据说在64bit的机器上一个Java对象占用内存最低为16个byte（我们暂时把这个当成已知的知识好了），那这就是32GB的内存，对于大多数机器来说这完全足够了
不过Java是一个兼容性很强，前瞻性也很强的语言，所以偷看一下jdk的实现吧，也是int，那就安心了，至于官方的实现为什么用了transient，那又不安心了。实际上transient是在将对象序列化成数据的时候不保存这个size，而jdk的序列化用了自定义的流程，这个size会反序列化的时候在添加元素的时候自动不停++，所以就不需要保存一遍了，但是为什么jdk的链表在自定义序列化过程呢，又不安了。

我们按照idea给我自动添加的方法顺序来实现这个类吧
从逻辑上来说, 先去实现add方法似乎会比较合理,但是链表本身是一个很基础的类,所以乱着来也完全没有问题

5, 实现size()方法:
    @Override
    public int size() {
        return size;
    }
6, 还有isEmpty:
    @Override
    public boolean isEmpty() {
        return size == 0;
    }

7, 接着是contains方法
    @Override
    public boolean contains(Object o) {
        Node<E> node = this.head;
        while (node != null) {
            if (Objects.equals(node.value, o)) {
                return true;
            }

            node = node.next;
        }
        return false;
    }

8, 接着是iterator方法
Iterator本身不是链表结构包含的东西, 而是Java的List接口要求实现的. 
Iterator本身是一个接口, 所以我们要么返回一个匿名内部类对象, 要么要写一个专门的类实现这个接口, 因为Iterator的两个方法hasNext和next要更新状态, 所以它是需要实例变量的, 所以还是定义一个类比较合适
iterator这个方法不能返回一个单例, 否则第二次获取这个iterator它就已经在尾部了
本质上来说这个Iterator只需要拿到head对象就可以遍历了, 这样看的话可以用静态的内部类, 但是Iterator还有一个remove方法, 要实现它必须对原list进行改动, 比如如果我们移除第一个元素, 那么List.head变量需要改变, 所以这里必须用非静态的(很显然把List传给Iterator有点多此一举了)

to be continued...

