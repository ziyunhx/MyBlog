title: .NET使用DRPC远程调用运行在Storm上的拓扑
date: 2015-08-12
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

 Distributed RPC（DRPC）是Storm构建在Thrift协议上的RPC的实现，DRPC使得你可以通过多种语言远程的使用Storm集群的计算能力。DRPC并非Storm的基础特性，但它确实非常有用。DRPC的整个过程与一般的RPC没有区别，客户端只需要调用一个远程的方法并等待返回结果。主要工作已经被DRPC Server封装，服务端在这个过程中完成了以下步骤：

<!--more-->
 - 从客户端接收一个RPC请求；
 - 将请求发送到storm topology；
 - 从storm topology接收结果；
 - 将结果发回给等待的客户端。

 ![image](http://blog.tnidea.com/media/image/drpc-workflow.png)

 [storm-net-adapter](https://github.com/ziyunhx/storm-net-adapter "storm-net-adapter")现在已经完成了对DRPC的支持，因此你可以使用dotNet编写代码远程调用任何支持语言编写的支持DRPC的Topology，当然你也可以使用dotNet编写Topology供其它语言通过DRPC调用。

 DRPC是[storm-net-adapter](https://github.com/ziyunhx/storm-net-adapter "storm-net-adapter")新增加的特性，因此需要使用最新的类库，你可以使用源代码自行编译，或者下载最新的[Release](https://github.com/ziyunhx/storm-net-adapter/releases "Release")，还可以使用Nuget获取最新版本。

    PM> Install-Package Storm.Net.Adapter

 推荐大家使用Nuget获取，方便管理依赖项。下面将介绍如何通过DRPC调用运行在Storm集群的方法，在这之前，你需要已经熟悉Storm环境的搭建与集群部署，不了解的可以先看我之前的文章。为了尽可能的简单，我们使用了Storm官方的BasicDRPCTopology，这个是一个简单的使用JAVA编写的DRPC Topology，它的功能仅仅是在传入的单词后面增加一个感叹号。

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
import backtype.storm.LocalDRPC;
import backtype.storm.StormSubmitter;
import backtype.storm.drpc.LinearDRPCTopologyBuilder;
import backtype.storm.topology.BasicOutputCollector;
import backtype.storm.topology.OutputFieldsDeclarer;
import backtype.storm.topology.base.BaseBasicBolt;
import backtype.storm.tuple.Fields;
import backtype.storm.tuple.Tuple;
import backtype.storm.tuple.Values;

/**
 * This topology is a basic example of doing distributed RPC on top of Storm. It implements a function that appends a
 * "!" to any string you send the DRPC function.
 * <p/>
 * See https://github.com/nathanmarz/storm/wiki/Distributed-RPC for more information on doing distributed RPC on top of
 * Storm.
 */
public class BasicDRPCTopology {
  public static class ExclaimBolt extends BaseBasicBolt {
    @Override
    public void execute(Tuple tuple, BasicOutputCollector collector) {
      String input = tuple.getString(1);
      collector.emit(new Values(tuple.getValue(0), input + "!"));
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
      declarer.declare(new Fields("id", "result"));
    }

  }

  public static void main(String[] args) throws Exception {
    LinearDRPCTopologyBuilder builder = new LinearDRPCTopologyBuilder("exclamation");
    builder.addBolt(new ExclaimBolt(), 3);

    Config conf = new Config();

    if (args == null || args.length == 0) {
      LocalDRPC drpc = new LocalDRPC();
      LocalCluster cluster = new LocalCluster();

      cluster.submitTopology("drpc-demo", conf, builder.createLocalTopology(drpc));

      for (String word : new String[]{ "hello", "goodbye" }) {
        System.out.println("Result for \"" + word + "\": " + drpc.execute("exclamation", word));
      }

      cluster.shutdown();
      drpc.shutdown();
    }
    else {
      conf.setNumWorkers(2);
      StormSubmitter.submitTopologyWithProgressBar(args[0], conf, builder.createRemoteTopology());
    }
  }
}
{% endcodeblock %}

 相关代码已经集成到[storm-starter](https://github.com/ziyunhx/storm-net-adapter/tree/master/storm-starter)中，下面我们还需要修改一下Storm的配置文件：

    drpc.servers:
     - "drpc1.foo.com"
     - "drpc2.foo.com"
 
 将drpc1.foo.com替换成你接下来要启动drpc服务机器的IP或者域名，你可以只保留一条，也可以继续增加服务的数量。

 在你刚刚填写的IP所在服务器上启动drpc服务：

    $ storm drpc

 使用storm命令提交Topology：
 
	$ storm jar storm-starter-*.jar storm.starter.BasicDRPCTopology drpc-test

 然后我们就可以在Csharp上编写代码调用了：

{% codeblock lang:csharp %}
DRPCClient client = new DRPCClient("localhost", 3772);
string result = client.execute("exclamation", "hello word");
{% endcodeblock %}

 替换 localhost 为你的drpc服务器的地址，exclamation为你在java中设置的LinearDRPCTopologyBuilder的名称；我也在项目中新增了一个控制台程序[Storm.DRPC.Demo](https://github.com/ziyunhx/storm-net-adapter/tree/master/samples/Storm.DRPC.Demo)以便大家用于测试！
 
