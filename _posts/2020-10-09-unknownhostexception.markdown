---
layout: post
title:  "UnknownHostException意味着什么"
date:   2020-09-28 18:24:26 +0800
categories: java
---

很多时候在断网的时候, 一个http请求会以一个UnknownHostException结束  
一个最原始的处理方法是把exception的msg以toast的形式抛出, 我经常这样干, 我觉得这样挺好的, 什么问题都一目了然  
可是这样做的话, 99.999%的概率在用户反馈群中会有人报告"app报错了/弹出了乱码", 然后不明所以的项目经理也会问这是发生了什么错误  
所以还是要处理成错误页面, or即使是弹吐司, 也要弹一个不明所以的“网络错误”

UnknownHostException是什么呢, 它的文档说:  

    Thrown to indicate that the IP address of a host could not be determined.

所以它其实是整个请求的第一步: 域名解析, 出了错误

如果我们使用Java中最原始的的URL类来发出http请求, 如下:

    URL url = new URL("http://www.google.com");
    URLConnection conn = url.openConnection();
    InputStream in = conn.getInputStream();

这个错误的发生路径是:

    URLConnection.getInputStream()
    URLConnection.URLtoSocketPermission(URL)
    URLConnection.getHostAndPort(URL)
    InetAddress.getByName(String host)
    ...一堆内部impl调用
    native Inet6AddressImpl.lookupAllHostAddr(String host) // 如果机器支持ipv6的话

native部分的代码在Inet6AddressImpl.c的Java_java_net_Inet6AddressImpl_lookupAllHostAddr函数中, 略过其中不知道在干什么的c层的东西, 它最主要做的是:

    error = getaddrinfo(hostname, NULL, &hints, &res);

    if (error) {
      NET_ThrowUnknownHostExceptionWithGaiError(env, hostname, error);
    }

其中getaddrinfo是c中获取ip地址的函数  


     The getaddrinfo() function is used to get a list of IP addresses and port
     numbers for host hostname and service servname.

NET_ThrowUnknownHostExceptionWithGaiError是定义在net_util_md.c中的一个函数, 基本上就是

    jobject x = JNU_NewObjectByName(env, "java/net/UnknownHostException", "(Ljava/lang/String;)V", s);
    if (x != NULL) (*env)->Throw(env, x);

所以就是这样了~