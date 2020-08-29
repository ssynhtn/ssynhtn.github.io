---
layout: post
title:  "ArrayList的序列化/反序列化，版本兼容，serialVersionUID"
date:   2020-08-27 18:20:27 +0800
categories: java
---

查看ArrayList的源码的时候发现，保存元素的数组被标记为transient

    transient Object[] elementData; // non-private to simplify nested class access

有点奇怪，ArrayList是标记为Serializable的

印象中transient只用来标记不重要的和无法序列化的数据，准确说是序列化时不保存这个变量

也就是说在序列化的时候elementData是不被保存的

看了一下是因为ArrayList自定义了序列化过程，因为elementData的长度是比ArrayList的真实size要大的，尾部的null没有必要保存

关于自定义序列化的文章：
https://www.oracle.com/technical-resources/articles/java/serializationapi.html

ArrayList中自定义序列化的逻辑:

    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            int capacity = calculateCapacity(elementData, size);
            SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }

除了检查modCount以外，这个逻辑最奇怪的地方在于：专门写了size, 但是读取的时候丢弃了readInt()的结果

因为size字段本身是没有标记transient, 因此它在defaultWrite和defaultRead的时候是能够正确恢复的，那为什么要多此一举写/读一次呢？

这里的注释似乎让人容易误解这是为了和clone方法对size/capacity的处理保持行为一致

但是如果只是为了保持这个行为一致，完全没必要多write/read一次，这样做的目的其实是为了兼容最初版（jdk1.2）的ArrayList的序列化过程:

    private synchronized void writeObject(java.io.ObjectOutputStream s)
            throws java.io.IOException{
        // Write out element count, and any hidden stuff
        s.defaultWriteObject();

        // Write out array length
        s.writeInt(elementData.length);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++)
            s.writeObject(elementData[i]);
    }

    private synchronized void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in array length and allocate array
        int arrayLength = s.readInt();
        elementData = new Object[arrayLength];

        // Read in all elements in the proper order.
        for (int i=0; i<size; i++)
            elementData[i] = s.readObject();
    }


这个版本中也是自定义了序列化过程，但是在读的时候，保持了和原ArrayList的capactiy一致

而新版的序列化过程则是用size代替了capacity，显然新版是更好的


但是如果仅仅是写/读对象读顺序保持一致，是无法保证旧的被序列化的文件可以用新版的ArrayList类来反序列化的，因为序列化会记录这个类的serialVersionUID, 反序列化过程会检查当前这个类的serialVersionUID是否和文件中这个类记录的serialVersionUID一致  

可以看到新版的ArrayList中指定了这个8683452581122892189L
但是jdk1.2版的ArrayList没有明确指定serialVersionUID，当一个类没有明确指定这个数值的时候，系统会根据一个规则计算一个hash值，这个规则会包括类名，public的构造函数，字段和方法  

具体的规则在这里：
https://docs.oracle.com/javase/6/docs/platform/serialization/spec/class.html#4100

而代码在ObjectStreamClass.computeDefaultSUID这个方法中
通过拷贝一份1.2版本的ArrayList代码，拷贝一份computeDefaultSUID这个方法，并稍作修改（传入指定的"java.util.ArrayList"类名），发现1.2版的hash值计算出来确实是8683452581122892189L

我感觉ArrayDeque这个类的实现其实完全可以取代ArrayList的，但是jdk仍然是单独写了ArrayDeque

我猜测有两个原因，但是这些原因都是从方便的角度出发的：

1，ArrayDeque的capacity为了取余计算%的方便必须是2的, ArrayList的扩容方法更细致一些(大多数情况下是按1.5倍增长的)

2, 兼容序列化也许是其中一个考虑的一个地方，虽然可以做到，但是那样也会麻烦一些
