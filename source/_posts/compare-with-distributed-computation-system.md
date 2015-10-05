title: 开源分布式计算系统框架比较
date: 2015-05-03
categories: 
- Storm
tags: 
- Storm
- Spark Streaming
- Hadoop MapReduce
- 流式框架
- 分布式

---

 分布式计算在许多领域都有广泛需求，目前流行的分布式计算框架主要有 Hadoop MapReduce, Spark Streaming, Storm； 这三个框架各有优势，现在都属于 Apache 基金会下的顶级项目，下文将对三个框架的特点与适用场景进行分析，以便开发者能快速选择适合自己的框架进行开发。

<!--more-->

 Hadoop MapReduce 是三者中出现最早，知名度最大的分布式计算框架，最早由 Google Lab 开发，使用者遍布全球（[Hadoop PoweredBy](http://wiki.apache.org/hadoop/PoweredBy "Hadoop PoweredBy")）；主要适用于大批量的集群任务，由于是批量执行，故时效性偏低，原生支持 Java 语言开发 MapReduce ，其它语言需要使用到 Hadoop Streaming 来开发。Spark Streaming 保留了 Hadoop MapReduce 的优点，而且在时效性上有了很大提高，中间结果可以保存在内存中，从而对需要迭代计算和有较高时效性要求的系统提供了很好的支持，多用于能容忍小延时的推荐与计算系统。Storm 一开始就是为实时处理设计，因此在实时分析/性能监测等需要高时效性的领域广泛采用，而且它理论上支持所有语言，只需要少量代码即可完成适配器。

 下面的表格是对三者部分特性的比较，描述时间为 2015-5-3，三个项目均处于快速迭代中，文中描述特性会随时产生变化，如果与官方文档产生出入以官方文档为准。


| 比较项	        | Storm    | Spark Streaming         | Hadoop MapReduce     |
| ------------: | ----------- |-----------| -----------|
| 血统   		| Twitter       | UC Berkeley AMP lab | Google Lab	 |
| 开源时间		| 2011.9.16     | 2011.5.24     | 2007.9.4          |
| 当前版本 		| 0.9.4         | 1.3.1         | 2.7.0          |
| 相关资料		| 多				| 多				| 极多			 |
| 依赖环境		| Zookeeper、Java、Python | hadoop client、Scala | Java、ssh          |
| 技术语言		| Java、Clojure          | Scala         | Java         |
| 支持语言		| Any           | Scala、Java、Python | Java & Others	 |
| 延时   		| 实时			| 秒级			| 较高 		 |
| 网络带宽		| 一般			| 一般			| 一般 		 |
| 硬盘IO 		| 一般			| 少			| 较少			 |
| 集群支持		| 好				| 超过1000节点	| 数千个节点			 |
| 吞吐量 		| 较好			| 好				| 好 			 |
| 使用公司   	| 淘宝、百度、Twitte、Groupon、雅虎 | Intel、腾讯、淘宝、中移动、Google	| EBay、Facebook、Google、IBM      |
| 适用场景		| 实时的小数据块的分析计算    | 较大数据块又需要高时效性的小批量计算				| 低时效性的大批量计算        		|


表格说明：

 - 开源时间以 github 上最早的 commit 或者官网上最早发布版本的时间为准。
 
 - 当前版本与特性描述截止 2015-5-3。
 
 - 相关资料量通过比较官方文档、搜索引擎、论坛等途径得出。
 
 - 部分比较数据来源于实践或相关文章（未找到出处）。

本文会保持更新，如果数据发现有出入，欢迎指正。

参考资料：

 - [Hadoop 官网](http://hadoop.apache.org/index.html "hadoop")
 
 - [Spark Streaming](http://spark.apache.org/streaming/ "Spark Streaming")
 
 - [Storm 官网](http://storm.apache.org/ "storm")
  