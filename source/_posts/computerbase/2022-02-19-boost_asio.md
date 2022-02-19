---
title: boost.asio分析
date: 2022-02-19
updated: 2022-02-19
tags: 
  - boost.Asio
categories:
  - network
aplayer: true
---

### 结构

```
boost::asio：这是核心类和函数所在的地方。重要的类有io_service和streambuf。类似read, read_at, read_until方法，它们的异步方法，它们的写方法和异步写方法等自由函数也在这里。
boost::asio::ip：这是网络通信部分所在的地方。重要的类有address, endpoint, tcp, udp和icmp，重要的自由函数有connect和async_connect。要注意的是在boost::asio::ip::tcp::socket中间，socket只是boost::asio::ip::tcp类中间的一个typedef关键字。
boost::asio::error：这个命名空间包含了调用I/O例程时返回的错误码
boost::asio::ssl：包含了SSL处理类的命名空间
boost::asio::local：这个命名空间包含了POSIX特性的类
boost::asio::windows：这个命名空间包含了Windows特性的类
```

