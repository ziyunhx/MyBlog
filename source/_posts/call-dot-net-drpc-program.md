title: 使用DRPC调用.NET开发的Storm拓扑
date: 2015-09-12
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

 在上一篇文章里介绍了.Net版本的DRPC的类库，结尾处的例子使用了Java版 BasicDRPCTopology ，本文我将介绍如何使用C#来开发 Storm Topology 并供远程DRPC调用，这篇文章将作为.Net Storm系列文章的一个暂时的结束，新的文章将随着 [storm-net-adapter](https://github.com/ziyunhx/storm-net-adapter "storm-net-adapter") 新特性来更新!

<!--more-->
 在开始介绍具体代码之前，我们先回顾一下DRPC的数据流图：

 ![DRPC-workflow](https://www.tnidea.com/media/image/drpc-workflow.png)

 从图中可以看到，DRPC实际上并不会影响的具体的业务逻辑代码的编写，只是在传入参数中增加了一个 request-id 用于辨别任务，在结束时增加了一个 return-info 用来返回结果。那么，在Bolt的编写中，我们为每一个Bolt的输入输出增加一个 id 参数来标识任务，在结束时额外增加一个返回结果的参数。我们还是写一个简单的字符操作的例子。

{% codeblock lang:csharp %}
using Storm;
using System;
using System.Collections.Generic;

namespace StormSample
{
    public class SimpleDRPC : IBolt
    {
        private Context ctx;

        /// <summary>
        ///  Implements of delegate "newPlugin", which is used to create a instance of this spout/bolt
        /// </summary>
        /// <param name="ctx">Context instance</param>
        /// <returns></returns>
        public static SimpleDRPC Get(Context ctx)
        {
            return new SimpleDRPC(ctx);
        }

        public SimpleDRPC(Context ctx)
        {
            Context.Logger.Info("SimpleDRPC constructor called");
            this.ctx = ctx;

            // Declare Input and Output schemas
            Dictionary<string, List<Type>> inputSchema = new Dictionary<string, List<Type>>();
            inputSchema.Add("default", new List<Type>() { typeof(string), typeof(object) });
            Dictionary<string, List<Type>> outputSchema = new Dictionary<string, List<Type>>();
            outputSchema.Add("default", new List<Type>() { typeof(string), typeof(object) });
            this.ctx.DeclareComponentSchema(new ComponentStreamSchema(inputSchema, outputSchema));
        }

        public void Execute(StormTuple tuple)
        {
            Context.Logger.Info("SimpleDRPC Execute enter");
            
            string sentence = tuple.GetString(0) + "!";
            this.ctx.Emit("default", new List<StormTuple> { tuple }, new List<object> { sentence, tuple.GetValue(1) });
            Context.Logger.Info("SimpleDRPC Execute exit");
            ApacheStorm.Ack(tuple);
        }

        public void Prepare(Config stormConf, TopologyContext context)
        {
            return;
        }
    }
}
{% endcodeblock %}

以上代码需要引用[storm-net-adapter](https://github.com/ziyunhx/storm-net-adapter "storm-net-adapter")，详细信息可以查看我之前的文章，下面我们来使用Java定义这个DRPC Topology:

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
import backtype.storm.drpc.DRPCSpout;
import backtype.storm.drpc.LinearDRPCTopologyBuilder;
import backtype.storm.drpc.ReturnResults;
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
public class DrpcTestTopologyCsharp {
	public static class SimpleDRPC extends ShellBolt implements IRichBolt {

		public SimpleDRPC() {
			super("cmd", "/k", "CALL", "StormSample.exe", "SimpleDRPC");
			
		}

		@Override
		public void declareOutputFields(OutputFieldsDeclarer declarer) {
			declarer.declare(new Fields("id", "result"));
		}

		@Override
		public Map<String, Object> getComponentConfiguration() {
			return null;
		}
	}	
	
	public static void main(String[] args) throws Exception {
	  	TopologyBuilder builder = new TopologyBuilder();
		  
	  	DRPCSpout drpcSpout = new DRPCSpout("simpledrpc");
	    builder.setSpout("drpc-input", drpcSpout,1);

	    builder.setBolt("simple", new SimpleDRPC(), 2)
	    		.noneGrouping("drpc-input");
	    
	    builder.setBolt("return", new ReturnResults(),1)
		.noneGrouping("simple");

	    Config conf = new Config();
	    conf.setDebug(true);
	    conf.setMaxTaskParallelism(1);
	    
	    try
	    {
	    	StormSubmitter.submitTopology("drpc-q", conf,builder.createTopology());
	    }
	    catch (Exception e)
	    {
	    	e.printStackTrace();
	    }
	}
}
{% endcodeblock %}

 相关代码已经集成到[storm-starter](https://github.com/ziyunhx/storm-net-adapter/tree/master/storm-starter)中，部署并启动该Topology，然后使用C#或其它语言调用：

{% codeblock lang:csharp %}
DRPCClient client = new DRPCClient("localhost", 3772);
string result = client.execute("simpledrpc", "hello word");
{% endcodeblock %}

 下面是一些其它语言DRPC项目的搜集：

 - C#/.Net	Storm.Net.Apapter https://github.com/ziyunhx/storm-net-adapter
 - Java	Storm 官方包 http://storm.apache.org/
 - Python	storm-drpc-client https://pypi.python.org/pypi/storm-drpc-client
 - Php	php-drpc https://github.com/mithunsatheesh/php-drpc