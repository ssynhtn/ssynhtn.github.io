---
layout: post
title:  "Tallest Character In Android"
date:   2018-04-12 15:53:25 +0800
categories: android
---

有一个任务看上去很简单: 居中显示一个字母  
那就用一个TextView, gravity=center就好了  
可是如果文字比较大的时候, 就会感觉这个文字并没有完全居中? 就像是这样  
![TextView Gravity Center]({{ "/assets/tall-char/center_fail.png" | absolute_url }})  
搜索一下就会发现, 其实可以准确地居中一个字母  
但是貌似有两种很不一样的解决方式  
一种通过Paint.getTextBounds()获取字母的bounding rect, 还有一种方法是利用FontMetrics中的ascent, descent等属性  

第一种方法是可行的  
{% highlight java %}
String text = "j";
paint.getTextBounds(text, 0, text.length(), rect);
float baseX = getWidth() / 2.0f - rect.exactCenterX();
float baseY = getHeight() / 2.0f - rect.exactCenterY();
canvas.drawText(text, baseX, baseY, paint);
{% endhighlight %}

就像这样  
![TextView Center Using Bounds]({{ "/assets/tall-char/center_good.png" | absolute_url }})

第二种方法就很可疑了  
FontMetrics对ascent这个field的注释是这样的  
The recommended distance above the baseline for singled spaced text.  
用它这也能居中文字才怪了, 不同的字母的ascent和descent都很不一样, 要居中不同的字母不可能用FontMetrics.xxx就能完美解决的

通过getTextBounds后, -rect.top实际是一个字母的高度, 哪个字母最高呢...  
如果在Character.MIN_VALUE ~ MAX_VALUE这个范围查找, 会发现是〱, 这是一个日文字母, 一般用于纵向排版
如果在所有的code point中查找, 会发现最高的字母是U+1d11f, 是一个音乐符号

![Tallest Character U+1d11f]({{ "/assets/tall-char/tall_char.png" | absolute_url }})
