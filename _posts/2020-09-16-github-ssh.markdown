---
layout: post
title:  "github添加ssh"
date:   2020-09-16 18:24:26 +0800
categories: bash
---

1， 生成rsa pair, 至于rsa密钥是啥， 嗯其实不是很清楚

    ssh-keygen -t rsa

默认生成的文件是.ssh目录下的id_rsa和id_rsa.pub, 如果需要生成第二对的话， 中间提示文件目录的时候输入dir/filename就可以了

2， 到github的ssh设置中新建key， 将.pub文件的内容复制进去

3， 如果是指定了名称的key， 那么要手动添加文件

    ssh-add .ssh/private_key_file

4, .gitconfig中的代理设置貌似只能设置http连接的代理， ssh使用代理有自己的设置, 文件为.ssh/config

    Host github.com
    HostName github.com
    User git
    # 走 HTTP 代理
    # ProxyCommand socat - PROXY:127.0.0.1:%h:%p,proxyport=1087
    # 走 socks5 代理（如 Shadowsocks）
      ProxyCommand nc -v -x 127.0.0.1:1086 %h %p

5, 检测ssh连接

    ssh -T git@github.com

如果失败的话可以加上v参数查看是什么原因