title: 搭建.Net开发Storm拓扑的环境
date: 2015-05-20
categories: 
- Storm
tags: 
- Storm
- zookeeper
- 流式框架
- 分布式
- 部署

---

 上篇博客比较了目前流行的计算框架特性，如果你是 Java 开发者，那么根据业务场景选择即可；但是如果你是 .Net 开发者，那么三者都不能拿来即用，至少在这篇文章出现之前是如此。基于上篇文章的比较发现，Storm 应该是对多语言支持比较好的框架了，但即便如此，官方也没有提供 .Net 的适配器，网上也找不到第三方的开源库。So，[Storm.Net.Adapter](https://github.com/ziyunhx/storm-net-adapter "Storm.Net.Adapter") 出现了，一个使用 Csharp 开发的 针对 Apache Storm 的适配器！

<!--more-->
 本文是“Storm系列”的第一篇，后期会根据时间情况继续更新，欢迎大家关注文章末尾处的微信公众号：dotNet大数据，如果你愿意提供素材或建议，欢迎通过 About 中的邮箱与我联系。


## 安装Storm与依赖环境 ##

### 安装Zookeeper ###

 - 获取最新 Zookeeper 程序包：[官网](http://zookeeper.apache.org/ "zookeeper")

 - 解压程序包，拷贝 conf 下 zoo_sample.cfg 为 zoo.cfg，修改相关配置

 - Windows 环境下直接执行 bin\zkServer.cmd；Linux 下执行 `bin/zkServer.sh start`

### 安装Python, Java与Maven ###

 - 下载 Python 2.x 安装

 - 下载 JAVA 6+ 安装，必须安装 JDK 版，否则使用 Maven 时会出错

 - 下载 Maven 并安装

### 下载Storm ###

 - 获取最新 Storm 程序包：[官网](http://storm.apache.org/downloads.html "Storm")

 - 解压后修改 conf 下的 storm.yaml 里的相关配置

### 配置环境变量 ###

 - 配置 Storm_Home 与 Java_Home; 目录最好不要有空格

 - classPath 里增加 `.;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\toos.jar;`

 - path 里增加 `%STORM_HOME%\bin;%JAVA_HOME%\bin;`

 - 将 Maven 的目录也加到 path 方便使用

### Storm启动 ###

 - 启动 Zookeeper

 - 运行 `storm nimbus` （如果未将 Storm 加到 path，需要先切换到 Storm 的 bin 目录，下同）

 - 运行 `storm supervisor` （集群环境下，非主可以仅执行该句）

 - 运行 `storm ui`，通过 http://localhost:8080/ 监控 Storm 运行状况

## 使用 Storm.Net.Adapter ##

### 获取 Storm.Net.Adapter ###

 目前有以下几种方式获取最新的 Storm.Net.Adapter 库

 - 通过源代码编译自己的版本： [GitHub](https://github.com/ziyunhx/storm-net-adapter "Storm.Net.Adapter")

 - 下载编译好的版本加入引用： [Release](https://github.com/ziyunhx/storm-net-adapter/releases "Storm.Net.Adapter Release")

 - 使用 NuGet 获取最新版本（推荐）：`PM> Install-Package Storm.Net.Adapter`

### 创建示例项目 ###

 - 在项目中引用 Storm.Net.Adapter，创建 Spout （基于ISpout）和 Bolt （基于IBolt或IBasicBolt），都需要 `using Storm;`

 - 创建一个使用 Maven 管理的 Java 项目，增加 dotNet 程序对应的 Topology

 - Windows（.Net Framework）平台下，你可以通过下面的方式来调用你的 Spout 或 Bolt：

		super("cmd", "/k", "CALL", "StormSimple.exe", "generator");

 - Linux, Mac OSX, Windows（mono）平台下，你可以通过下面的方式来调用你的 Spout 或 Bolt：

		super("mono", "StormSimple.exe", "generator");

### 打包与发布 ###

 - 拷贝编译好的 dotNet 程序到 resources 目录下，使用下面的 Maven 命令打包你的 Topology：

    	$ mvn package

 - 通过 Storm 命令行工具提交你创建好的 Topology：

		$ storm jar storm-starter-*-jar-with-dependencies.jar storm.starter.WordCountTopologyCsharp wordcount

## Storm系列文章 ##

（一）：[搭建dotNet开发Storm拓扑的环境](http://www.tnidea.com/deploy-storm-environment.html "搭建dotNet开发Storm拓扑的环境")

（二）：[使用Csharp创建你的第一个Storm拓扑（wordcount）](http://www.tnidea.com/you-first-csharp-storm-topology.html "使用Csharp创建你的第一个Storm拓扑")

（三）：[创建Maven项目打包提交wordcount到Storm集群](http://www.tnidea.com/deploy-wordcount-topology "创建Maven项目打包提交wordcount到Storm集群")