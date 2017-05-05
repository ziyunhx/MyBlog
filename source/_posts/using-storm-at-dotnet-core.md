title: .NET Core 也能玩转 Storm
date: 2016-7-11
categories: 
- Storm
tags:
- csharp
- .NET Core
- dotnet
- storm

---

 .Net Core 自发布以来广受关注，基于其开源与跨平台的特性，可以预见其在 web 开发领域越来越受青睐。现在，Apache Storm 的 .Net Core 版本的适配器正式发布，你现在也可以使用 .Net Core 开发 Topology，实现分布式跨平台的实时计算。

<!--more-->
 
 [storm-net-adapter](https://github.com/ziyunhx/storm-net-adapter) 现在已经更新到 2.0.4，增加了对 .Net Core 的支持，并对之前存在的例如输出复杂日志时的 BUG 进行了修复；由于目前 Thrift 的适配暂时还未完成，因此还无法使用 DRPC 的特性。同时，Storm 集群的 Docker 的镜像编排也已经集成了 .Net Core ，你可以通过 [storm-mono-docker](https://github.com/ziyunhx/storm-mono-docker) 获取到它。
 
 .Net Core 玩转 Storm 的首要条件当然是安装 .Net Core 的开发环境啦，你可以通过以下地址查看如何安装：[.Net Core](https://www.microsoft.com/net/core) 。

 然后使用命令创建新的 .Net Core 项目：

	mkdir StormSample
	cd StormSample
    dotnet new

 在项目中添加 Storm.Net.Adapter 引用：

    "dependencies": {
        "Storm.Net.Adapter": "2.0.1"
    }
 
 接下来增加业务代码，继承 ISpout 和 IBolt，实现接口方法，在 Main 方法里实现调用方法，具体代码可以参照 [StormSample](https://github.com/ziyunhx/storm-net-adapter/tree/master/src/Samples/StormSample) 。

 在 Java 端增加调用 .Net 代码的 Topology；

 如果需要运行在 Windows(.Net Framework) 平台下通过如下代码调用：

    super("cmd", "/k", "CALL", "StormSimple.exe", "generator");

 在 Linux, Mac OSX, Windows(mono) 下通过如下代码调用：

    super("mono", "StormSimple.exe", "generator");

 在 Linux, Mac OSX, Windows(.net core) 下通过如下代码调用：

    super("dotnet", "StormSimple.dll", "generator");

 使用 Maven 打包 Java 项目，通过 storm jar 命令提交：

    $ storm jar storm-starter-*.jar org.apache.storm.starter.WordCountTopologyCsharp wordcount

 你就可以看到如下熟悉的界面咯：

 ![Storm UI](https://www.tnidea.com/media/image/storm-ui-docker-101.png)

 接下来需要完善与实现的任务还很多，项目的发展需要得到广泛的关注与支持；如果你觉得这个项目对您有帮助，欢迎在 GitHub 给出 Star ，项目地址：[storm-net-adapter](https://github.com/ziyunhx/storm-net-adapter) ，或者转发文章支持一下！