---
layout: post
title: "域名系统(DNS)"
description: ""
category: network
tags: []
---
{% include JB/setup %}

## DNS基础

DNS的名字空间和Unix的文件系统相似，也具有层次结构，如下图所示：

![](/images/network/dns-hierarchy.png)

每个节点有最多63各字符串的标识，根节点是没有任何标识的特殊节点。
命名标识一律不区分大小写，命名树上任何一个节点的域名就是将从该节点到最高层的域名串连起来，中间使用"."来分隔这些域名。
域名树中的每一个节点必须有一个唯一的域名，但域名树中的不同节点可使用相同的标识。

以"."结尾的域名成为绝对域名或完全合格的域名FQDN(Full Qualified Domain Name)，例如ruleless.net.。
如果一个域名不以"."结尾，则认为该域名是不完全的，如何使域名完整依赖于使用的DNS软件。

顶级域被分为三个部分：

  1. 用作逆向DNS解析的特殊域(arpa)
  2. 7个3字符长的普通域
  3. 2字符串长度的国家域

7个普通域的划分含义如下：

  1. **com** 商业组织
  2. **edu** 教育机构
  3. **gov** 其他美国政府部门
  4. **int** 国际组织
  5. **mil** 美国军事网点
  6. **net** 网络
  7. **org** 其他组织

## DNS报文格式

DNS定义了一个用于查询和响应的报文格式，如下：

![](/images/network/DNSHeader.png)

标识字段由客户端程序设置并由服务器返回结果，客户程序通过它来确定响应和查询是否匹配。
16bit的标志字段被划分为若干个子字段，如下：

![](/images/network/DNSHeaderFlag.png)

各字段含义如下：

  + QR: 0表示查询报文，1表示响应报文
  + opcode: 0表示标准查询，1表示反向查询，2表示请求服务器状态
  + AA: 表示"授权回答"
  + TC: 表示"可截断的"，使用UDP时，它表示当应答的总长度超过512字节时，只返回前512个字节
  + RD: 表示"期望递归"。该bit能在一个查询中设置，并在响应中返回。
    这个标志告诉名字服务器必须处理这个查询，也称一个递归查询。
	如果该位为0，且被请求的名字服务器没有一个授权回答，它就返回一个能解答该查询的其他名字服务器列表。
	这称为迭代查询。
  + RA: 表示"可用递归"。如果名字服务器支持递归查询，则在响应中将该bit设为1
  + (zero): 必须填0
  + rcode: 返回字段码。0表示没有差错，3表示名字差错。名字差错只能从一个授权名字服务器返回，它表示在查询中指定的域名不存在。

随后的4个16bit字段表示后续变长字段中包含的条目数。对于查询报文，问题数通常为1，而其他3项均为0。
类似的，对于应答报文，回答数至少是1。