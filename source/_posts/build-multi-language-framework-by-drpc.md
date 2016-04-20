title: 使用DRPC构建分布式多语言编程架构
date: 2016-03-26
categories: 
- Storm
tags:
- 多语言
- C#
- .NET
- storm
- DRPC
- Topology

---

Distributed RPC（DRPC）作为Storm基于Thrift协议的RPC实现，已在之前的文章中被多次提及；在一个多开发语言的环境中，RPC是必不可少的一环，常见的RPC实现方式除了thrift外也还有很多，甚至基于Http协议的RESTful API也可以算作是其中的一员。本文将为您解读：为什么在这么多的RPC方式中选择使用DRPC来构建多语言编程架构，而不是使用Thrift或者其它方式？

<!--more-->

如果你开始阅读这篇文章，我相信你至少同意一家公司出现多种编程语言是一个再正常不过的事情。随着编程语言的发展，大多数语言都会有其最佳使用场景的概念，有些偏向快速开发，有些则更注重效率；不同的部门使用各自熟悉的或合适的语言开发，最终需要协同使用时，有人力的公司可以选择用一种语言重写一遍，那么另一些公司呢？

一开始，你写这个模块可能只是满足自己程序的需要，甚至只是做一个可行性的实验；后来大家发现很好用，越来越多的人使用这个模块，你不得不为了方便别人使用进一步封装，优化代码性能，考虑如何部署成服务，怎么实现负载均衡......
 
自己创造的代码被广泛使用确实让我信心爆棚，但越来越多的时间花在我原本并不擅长的方向......有没有一种方案让我只需要专心的写代码，多语言调用、部署、负载均衡、分布式什么都我都不需要去管？

答案当然是肯定的啦，下面我就给大家介绍基于 DRPC 构建的分布式多语言编程模式，个人才疏学浅，如果疏漏，请轻拍！

 ![DRPC-flow](http://www.tnidea.com/media/image/drpc-flow.jpg)

DRPC的整个过程与一般的RPC并没有区别，它的实现也是基于Thrift协议；让它变得与众不同的原因便是让你能方便的使用Storm集群的计算能力，当然这也意味着你的整个计算集群都得基于Storm。如果你对大数据的技术有过了解，即使没有听说过Storm，也一定听说过Hadoop吧，Storm类似于Hadoop中的MapReduce，它最大的改变便是将MapReduce中批处理的方式换成了实时的方式，另一个号称实时的框架spark streaming则只是把批处理变得更小更敏捷罢了。更多关于这三者比较的信息可以查看我之前的文章：[开源分布式计算系统框架比较](http://www.tnidea.com/compare-with-distributed-computation-system.html "开源分布式计算系统框架比较")。

编写运行在Storm框架上的DRPC模块是非常容易的，你只需要将你原来编写的逻辑代码包裹在Storm Bolt的封装代码中即可。当然，也会有一些限制，首先是需要将复杂的逻辑尽量拆解，将一个需要运行很久的代码直接封装可能会导致Bolt运行超时；其次，目前的DRPC版本仅支持一个字符串的输入输出，这也意味着复杂的对象需要额外的序列化与反序列化开销；第三，目前并非所有语言均已支持Storm，幸运的是完成对Storm的支持并不困难，下面给大家列出我整理的一些支持语言的DEMO（需要支持DRPC仅需将输入与输出参数的末尾增加一个ID）。

- [C# Demo](https://github.com/ziyunhx/storm-net-adapter/blob/master/samples/StormSample/Counter.cs)
- [Java Demo](https://github.com/apache/storm/blob/master/examples/storm-starter/src/jvm/storm/starter/bolt/PrinterBolt.java)
- [Python Demo](https://github.com/apache/storm/blob/master/examples/storm-starter/multilang/resources/splitsentence.py)
- [NodeJS Demo](https://github.com/apache/storm/blob/master/examples/storm-starter/multilang/resources/splitsentence.js)
- [Ruby Demo](https://github.com/apache/storm/blob/master/examples/storm-starter/multilang/resources/splitsentence.rb)

如果你需要完成对一种目前尚不支持的语言的适配，可以查看 [Storm Multi-Language Protocol](http://storm.apache.org/releases/0.9.6/Multilang-protocol.html)。

另一个需要关心的便是在其它语言中的调用问题，它的实现相比支持Storm更加简单，毕竟只是对Thrift的进一步封装，同样提供我搜集的开源项目地址：

- C#/.Net	[Storm.Net.Apapter](https://github.com/ziyunhx/storm-net-adapter)
- Java	[Storm 官方项目](http://storm.apache.org/)
- Python	[storm-drpc-client](https://pypi.python.org/pypi/storm-drpc-client)
- Php	[php-drpc](https://github.com/mithunsatheesh/php-drpc)

如果需要使用到生产环境中，性能和安全性也是需要考虑的问题。使用DRPC会带来一定性能损耗，但是不要担心，不论是官方的测试报告还是我们实际使用的结论都表明这一损耗在分布式集群中是可接受的，对于大部分的用户甚至无法感知；它的负载能力同样不俗，而且DRPC本身也是支持分布式的。DRPC目前并没有做许多额外的安全性的防护，我们目前也是只能通过切换默认端口号，不开放外网访问或者使用IP白名单，程序中验证访问权限的方式来完成。相信在今后这一块也会受到官方的重视，本文的例子还是以0.9.6版本为基础，官方的开发版本已经到了2.0。

如果你已经下定决心想要尝试下，我已经给你准备了一个Docker快速编排的镜像，你可以查看我之前的文章或者直接依据开源项目的步骤操作，虽然现在在Docker中使用JDK已经违反了协议，还望大家放我一马啊。

- [使用Docker快速部署Storm环境](http://www.tnidea.com/deploy-storm-by-docker.html)
- [storm-mono-docker](https://github.com/ziyunhx/storm-mono-docker)

另外，现在微信公众号已开通评论和打赏功能，有什么建议或者觉得文章对您有帮助，请移步公众号支持一下哦！