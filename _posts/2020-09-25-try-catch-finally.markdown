---
layout: post
title:  "try catch finally的执行顺序和返回结果"
date:   2020-09-25 18:24:26 +0800
categories: java
---

一个历史悠久的无聊面试题, 网上文章很多

经过一番考究发现原则很简单

先把官方文档贴一下

    14.20.2. Execution of try-finally and try-catch-finally
    A try statement with a finally block is executed by first executing the try block. Then there is a choice:

    If execution of the try block completes normally, then the finally block is executed, and then there is a choice:

    If the finally block completes normally, then the try statement completes normally.

    If the finally block completes abruptly for reason S, then the try statement completes abruptly for reason S.

    If execution of the try block completes abruptly because of a throw of a value V, then there is a choice:

    If the run-time type of V is assignment compatible with a catchable exception class of any catch clause of the try statement, then the first (leftmost) such catch clause is selected. The value V is assigned to the parameter of the selected catch clause, and the Block of that catch clause is executed. Then there is a choice:

    If the catch block completes normally, then the finally block is executed. Then there is a choice:

    If the finally block completes normally, then the try statement completes normally.

    If the finally block completes abruptly for any reason, then the try statement completes abruptly for the same reason.

    If the catch block completes abruptly for reason R, then the finally block is executed. Then there is a choice:

    If the finally block completes normally, then the try statement completes abruptly for reason R.

    If the finally block completes abruptly for reason S, then the try statement completes abruptly for reason S (and reason R is discarded).

    If the run-time type of V is not assignment compatible with a catchable exception class of any catch clause of the try statement, then the finally block is executed. Then there is a choice:

    If the finally block completes normally, then the try statement completes abruptly because of a throw of the value V.

    If the finally block completes abruptly for reason S, then the try statement completes abruptly for reason S (and the throw of value V is discarded and forgotten).

    If execution of the try block completes abruptly for any other reason R, then the finally block is executed, and then there is a choice:

    If the finally block completes normally, then the try statement completes abruptly for reason R.

    If the finally block completes abruptly for reason S, then the try statement completes abruptly for reason S (and reason R is discarded).

是不是觉得很长很复杂?

实际上我们只要注意到这么几件事情

1, completes normally和completes abruptly的区别

所谓completes abruptly, 猛地结束, 指的就是return和throw这两种语句, 这两种语句我们都可以把它看成这段代码有了一个结果

2, try和catch可以看成一个整体

3, finally是永远会执行, 即使try或者catch中抛了没有被捕捉到的exception

4, finally也是可以有结果的(return 或者throw), 正常情况下这是应该避免的情况, 因为finally的设计意图是做最后的清理, 例如close一个stream

5, 如果finally有结果, 它的结果会覆盖try/catch的结果

6, 这是相对来说最难理解的一点: 如果try/catch中return了x这个变量, finally中对变量x的值进行了改变(但是没有return x), 这个改变不影响整个try/catch/finally的返回值, 换句话说, try/catch中的return语句捕获了当时的x的值

对于最后一点, 我一开始觉得很奇怪, finally毕竟也是在整个方法返回前执行的, 既然它改变了x的值, 而且return的就是x, 那么return值应该被改变了. 

其实理解这个很简单, Java编译器很容易做到在try的return中记录当时的return值, 而在finally中对x的赋值虽然改变了x, 但是它不是整个方法的return值. 实际上javac就是这么做的, 这么做的原因还是回到出发点, 在finally中改变返回结果不是finally的设计意图, 更何况我们没有在finally中return x

对于3这这种确实在finally中return的情况, Java语言允许这样做, 但是idea会给一个警告"'return' inside 'finally' block "