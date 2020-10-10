---
layout: post
title:  "HTTP允许重复name的header吗"
date:   2020-10-10 18:24:26 +0800
categories: http
---

答: 可以

    Multiple message-header fields with the same field-name MAY be
    present in a message if and only if the entire field-value for that
    header field is defined as a comma-separated list [i.e., #(values)].
    It MUST be possible to combine the multiple header fields into one
    "field-name: field-value" pair, without changing the semantics of the
    message, by appending each subsequent field-value to the first, each
    separated by a comma. The order in which header fields with the same
    field-name are received is therefore significant to the
    interpretation of the combined field value, and thus a proxy MUST NOT
    change the order of these field values when a message is forwarded.


当且仅当这个name所对应的value是一组由逗号分隔的值. 并且这样的多个header可以合并到一个header中也不改变这个请求的语义

常见的例子: set-cookie, 经常见到一个response包含多个set-cookie的指令

按这个规定, request header中的 accept-encoding: gzip, deflate 似乎可以分成两个accept-encoding来发送