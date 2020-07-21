---
layout: post
title:  "Equals"
date:   2018-04-08 22:10:25 +0800
categories: jekyll update
---
在Java中，假如a是个变量，b = (a == a)
是否b一定为true？
turns out the answer is not
{% highlight java %}
public static void main(String[] args) {
  float v = Float.NaN;
  System.out.println("equals? " + (v == v));
}
{% endhighlight %}

这个结果是由float的spec规定的，NaN和任何东西用任何operator（除了!=）比较，结果都是false
也许你会和我一样觉得它和自己比较结果应该是true
但是NaN不代笔任何一个数，0.0f/0.0f是NaN，1.0f/0.0f也是NaN，so
0.0f/0.0f== 1.0f/0.0f结果为false貌似确实更合理