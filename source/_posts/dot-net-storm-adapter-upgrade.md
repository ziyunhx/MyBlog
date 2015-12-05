title: Storm适配器(.Net)升级说明以及一些性能数据
date: 2015-12-5
categories: 
- Storm
tags:
- csharp
- C#
- .NET
- storm
- DRPC
- Topology

---

 [storm-net-adapter](https://github.com/ziyunhx/storm-net-adapter "storm-net-adapter") 是一个使用C#编写的 Apache Storm 适配器；用于在 .Net 环境下开发 Storm 原生支持的拓扑，以及通过DRPC来远程跨语言使用集群计算资源。距离首次介绍已经更新了3个版本，带来了数项功能与性能的改进！下面将对改进部分进行说明。

<!--more-->
 首先，在V1.2版本加入了对DRPC的支持；通过使用DRPC，你可以方便调用任何使用Storm支持语言编写的Topology，你仅需将它们部署到集群。详细介绍可以通过 [.NET使用DRPC远程调用运行在Storm上的拓扑](http://www.tnidea.com/dot-net-drpc-storm.html) 和 [使用DRPC调用.NET开发的Storm拓扑](http://www.tnidea.com/call-dot-net-drpc-program.html) 来了解。
 在刚刚发布的V1.4版本中，对DRPC的支持进一步增强，默认使用线程池来管理连接。在之前的版本中，单机单进程环境下，如果所有并发使用各自连接，并发100的环境下长时间运行会出现连接数过大导致失败；如果使用单连接加锁模式，在并发30的情况下延时以及在5000ms以上。通过使用线程池来控制（默认配置），在生产环境下，峰值200并发的条件下，使用外网访问DRPC的延时保持在200ms内；每日调用量保持在2500万次，单个小时调用量峰值在300万次。Storm集群使用Docker部署在单台服务器上，部署方式可以查看 [使用Docker快速部署Storm环境](http://www.tnidea.com/deploy-storm-by-docker.html)。下面是连续10天Storm运行的截图：
 
 ![Storm UI](http://www.tnidea.com/media/image/storm-run-status-1.png)

 ![Storm Topology](http://www.tnidea.com/media/image/storm-run-status-2.png)

 当然，你也可以通过设置 reconnect 和 maxIdle 参数来决定是否使用线程池特性以及最大线程数。
 
 在V1.3版本，解决了一些在linux环境下运行的Bug，默认demo使用mono来驱动，并将Storm版本选用了0.9.6版本。
 
 在V1.4版本中，移除了项目对于Json.Net的依赖，采用SimpleJson作为替代；现在，再也不用慢慢等待Nuget的龟速了。另外，该类库支持.Net 3.5及以上；这是由于最新的Thrift并不支持.Net 3.5以下的版本，如果你需要使用更低版本的.Net，你可以从Apache Thrift官方网站下载源码修改后自行编译。
 
 好了，在未来的版本中，我们还计划增加直接通过.Net打包发布Topology的支持，如果你有什么希望增加或完善的部分，可以通过博客留言、公众号评论、GitHub或邮件与我交流！