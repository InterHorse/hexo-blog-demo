---
title: JavaMail 发送邮件阻塞问题解决——设置 smtp 超时时间
date: 2019-04-11 19:20:34
updated: 2019-04-11 19:20:34
tags: JavaMail
categories: Java
---
# 一. 起因
最近发现项目中有关发送邮件的模块偶尔会阻塞住，导致整个线程阻塞。诡异的是没有捕获到任何异常日志，程序莫名其妙就卡在了 sendMail 上。

后来想到发送邮件的内容过大，可能由于这个原因导致，所以找了一下有关 JavaMail 超时设置的资料。现做整理，顺便聊聊一些小坑。

# 二. JavaMail smtp 超时参数
|参数|类型  | 描述|
|--|--|--|
|  mail.smtp.connectiontimeout|int  |Socket connection timeout value in milliseconds. This timeout is implemented by java.net.Socket. Default is infinite timeout. |
|mail.smtp.timeout|int|Socket read timeout value in milliseconds. This timeout is implemented by java.net.Socket. Default is infinite timeout.|
|mail.smtp.writetimeout|int|Socket write timeout value in milliseconds. This timeout is implemented by using a java.util.concurrent.ScheduledExecutorService per connection that schedules a thread to close the socket if the timeout expires. Thus, the overhead of using this timeout is one thread per connection. Default is infinite timeout.|

源自 JavaMail API，文末有链接。

# 三. 参数简介
- mail.smtp.connectiontimeout：连接时间限制，单位毫秒。是关于与邮件服务器建立连接的时间长短的。默认是无限制。
- mail.smtp.timeout：邮件接收时间限制，单位毫秒。这个是有关邮件接收时间长短。默认是无限制。
- mail.smtp.writetimeout：邮件发送时间限制，单位毫秒。有关发送邮件时内容上传的时间长短。默认同样是无限制。

大部分博客、资料都提到了前两个属性，而容易忽略最后一个。因为我是用来发送邮件的，所以上传邮件的内容过大是导致发送模块阻塞的原因。而设置 writetimeout 时间后，再超时时就会报异常，捕获处理下就可以了。

# 四. 在 SpringBoot 中的配置
我所用的是 SpringBoot 工程，之前也发表过一篇有关 SpringBoot 配置邮件的博客：[SpringBoot 配置邮件服务](https://blog.csdn.net/Colton_Null/article/details/84653785)

只需要在原有配置的基础上，加上如下设置即可。

```yaml
spring:
  mail:
      mail:
        smtp:
          timeout: 10000
          connectiontimeout: 10000
          writetimeout: 10000
```

----
站在前人的肩膀上前行，感谢以下资料的支持。

- [JavaMail API documentation](https://javaee.github.io/javamail/docs/api/)
- [JavaMail的大坑](https://blog.csdn.net/j16421881/article/details/78513218)