title: Storm 1.0.1发布，.NET 适配也已到来
date: 2016-5-12
categories: 
- Storm
tags:
- csharp
- C#
- .NET
- storm
- Topology

---

 Apache Storm 1.0.0刚发布不久，1.0.1版本也在几天前到来；该版本主要是完成一些BUG修复和小的改进，通过一段时间新版本的使用，特将个人感受和一些遇到的问题归纳如下；另外 .NET 版本的 Storm 适配器也已经发布，源码在 [storm-net-adapter](https://github.com/ziyunhx/storm-net-adapter "storm-net-adapter")，如果你希望便捷的体验Storm 1.0.1，可以通过 Docker 来部署，地址在：[storm-mono-docker](https://github.com/ziyunhx/storm-mono-docker)，该镜像已经集成了Mono，你也可以查看我之前的文章来详细了解。

<!--more-->
 
 下图是使用 Docker 部署的 .NET 版的 Wordcount 的 Storm UI：
 
 ![Storm UI](https://www.tnidea.com/media/image/storm-ui-docker-101.png)
 
 通过图片我们可以看到，Topology 和 Supervisor 都增加了内存的占用字段，Nimbus 也支持多主机配置了，诟病多年的单 Nimbus 造成的稳定性隐患也终于得到解决。原有的配置项从 `nimbus.host` 换成了 `nimbus.seeds`，但实际测试如果你没有修改过来的话也只会出现警告，并不会崩溃。
 
 但是 Windows 版本的安装就会有些状况了，首先是 log4g2 的配置问题，原本逻辑的将 %STORM_HOME% 拼接在前面的逻辑并没有生效，你需要在 conf/storm.yaml 中配置日志配置文件的路径，类似：
 
    storm.log4j2.conf.dir: "X:/Storm/apache-storm-last/log4j2"
 
 然后你需要使用管理员权限运行命令，相关的资料你可以查看 [windows-users-guide](https://github.com/apache/storm/blob/master/docs/windows-users-guide.md "windows-users-guide")。
 
 另外 Storm 的安全机制也有了很大的提升，详细信息我会在后续专门翻译介绍，你也可以查看 [SECURITY](http://storm.apache.org/releases/1.0.1/SECURITY.html "SECURITY") 了解。storm jar 的远程提交在上面提供的 Docker 镜像中暂时还无法使用，可能和安全机制的提交也有关系。
 
 总体来说，Storm 的发展速度和方向都是符合我的个人预期的，新版本的文档目前还不够完善，建议大家先折腾一段时间踩踩坑再上生产环境。学习的话就直接从 1.0 版本开始吧。
 
 好了，本文就先介绍到这里。欢迎大家关注我的公众号，现在开始接受外部投稿，如果你有 STORM SPARK 等流式框架或者 Tensflow 等机器学习框架以及 NLP 自然语言处理相关的文章愿意在本公众号发布，欢迎在公众号和我留言！