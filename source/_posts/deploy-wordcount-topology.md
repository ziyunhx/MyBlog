title: 创建Maven项目打包提交wordcount到Storm集群
date: 2015-06-20
categories: 
- Storm
tags:
- Storm
- wordcount
- csharp
- C#
- .NET

---

 在上一篇博客中，我们通过[Storm.Net.Adapter](https://github.com/ziyunhx/storm-net-adapter "Storm.Net.Adapter")创建了一个使用Csharp编写的Storm Topology - wordcount。本文将介绍如何编写Java端的程序以及如何发布到测试的Storm环境中运行。

<!--more-->
 如果你觉得对你有帮助，欢迎Star和Fork，让更多人看到来帮助完善这个项目。

 **STEP1**: 克隆storm官方示例项目 [storm-starter](https://github.com/apache/storm/tree/master/examples/storm-starter)：
 
 ````
 $ git clone git://github.com/apache/storm.git && cd storm/examples/storm-starter
 ````
 
 **STEP2**: 增加csharp的多语言支持：
 
 将上一篇博客 [使用Csharp创建你的第一个Storm拓扑](https://www.tnidea.com/you-first-csharp-storm-topology.html "使用Csharp创建你的第一个Storm拓扑") 中完成的项目编译，把生产的组件拷贝到 `/multilang/resources/` 文件夹中。
 
 **STEP3**：使用JAVA创建Topology：

 在 `/src/jvm/storm/starter/` 新增 WordCountTopologyCsharp.java

{% codeblock lang:java %}
/**
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package storm.starter;

import backtype.storm.Config;
import backtype.storm.LocalCluster;
import backtype.storm.StormSubmitter;
import backtype.storm.spout.ShellSpout;
import backtype.storm.task.ShellBolt;
import backtype.storm.topology.IRichBolt;
import backtype.storm.topology.IRichSpout;
import backtype.storm.topology.OutputFieldsDeclarer;
import backtype.storm.topology.TopologyBuilder;
import backtype.storm.tuple.Fields;

import java.util.Map;

/**
 * This topology demonstrates Storm's stream groupings and multilang capabilities.
 */
public class WordCountTopologyCsharp {
	public static class Generator extends ShellSpout implements IRichSpout {

		public Generator() {
			super("cmd", "/k", "CALL", "StormSimple.exe", "generator");
			
		}

		@Override
		public void declareOutputFields(OutputFieldsDeclarer declarer) {
			declarer.declare(new Fields("word"));
		}

		@Override
		public Map<String, Object> getComponentConfiguration() {
			return null;
		}
	}	
	
	public static class Splitter extends ShellBolt implements IRichBolt {

		public Splitter() {
			super("cmd", "/k", "CALL", "StormSimple.exe", "splitter");
		}

		@Override
		public void declareOutputFields(OutputFieldsDeclarer declarer) {
			declarer.declare(new Fields("word", "count"));
		}

		@Override
		public Map<String, Object> getComponentConfiguration() {
			return null;
		}
	}
	
	public static class Counter extends ShellBolt implements IRichBolt {
		
		public Counter(){
			super("cmd", "/k", "CALL", "StormSimple.exe", "counter");
		}
		
		@Override
		public void declareOutputFields(OutputFieldsDeclarer declarer) {
			declarer.declare(new Fields("word", "count"));
		}

		@Override
		public Map<String, Object> getComponentConfiguration() {
			return null;
		}
	}
	

	public static void main(String[] args) throws Exception {

		TopologyBuilder builder = new TopologyBuilder();

		builder.setSpout("generator", new Generator(), 1);

		builder.setBolt("splitter", new Splitter(), 1).fieldsGrouping("generator",
				new Fields("word"));
		
		builder.setBolt("counter", new Counter(), 1).fieldsGrouping("splitter",
				new Fields("word", "count"));

		Config conf = new Config();
		conf.setDebug(true);

		if (args != null && args.length > 0) {
			conf.setNumWorkers(3);

			StormSubmitter.submitTopologyWithProgressBar(args[0], conf,
					builder.createTopology());
		} else {
			conf.setMaxTaskParallelism(3);

			LocalCluster cluster = new LocalCluster();
			cluster.submitTopology("WordCount", conf, builder.createTopology());

			Thread.sleep(10000);

			cluster.shutdown();
		}
	}
}
{% endcodeblock %}

本例是在window平台使用.Net执行，如果你使用Mono，或者在其它平台通过Mono运行，请将 

`super("cmd", "/k", "CALL", "StormSimple.exe", "xxxxxx");` 

替换为 

`super("mono", "StormSimple.exe", "xxxxxx");` 


 **STEP4**：编译并提交Topology：
 
 - 初始化安装storm所需依赖：`$ mvn clean install -DskipTests=true`
 - 使用Maven打包storm拓扑：`$ mvn package`
 - 搭建好运行环境并提交：

 `$ storm jar storm-starter-*-jar-with-dependencies.jar storm.starter.WordCountTopologyCsharp wordcount`

 storm集群的搭建请参考系列文章第一篇 [搭建dotNet开发Storm拓扑的环境](https://www.tnidea.com/deploy-storm-environment.html "搭建dotNet开发Storm拓扑的环境")
 
 ![image](https://www.tnidea.com/media/image/storm-3-01.png)
 
 ![image](https://www.tnidea.com/media/image/storm-3-02.png)

