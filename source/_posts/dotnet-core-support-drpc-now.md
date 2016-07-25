title: .NET Core 现已支持DRPC，同时带来Apache Thrift
date: 2016-7-25
categories: 
- Storm
tags:
- csharp
- .NET Core
- dotnet
- storm

---

 上篇文章为大家带来了新版本的 Storm 适配器，今天来弥补一下上次匆忙发布带来的遗憾。是的，DRPC for .net Core 来了，当然，为了实现这个功能，一个精简版本的 Apache Thrift for .net core 也产生了；这个类库根据 [Poadmap for adding new language bindings](https://thrift.apache.org/docs/HowToNewLanguage) 完成，为了不带来误解，该项目暂时不开源，仅在 Nugut 中供 [storm-net-adapter](https://github.com/ziyunhx/storm-net-adapter) 使用，如果你也暂时需要它，可以通过 Nuget 搜索 Tnidea.Thrift 获得。

<!--more-->
 
 [storm-net-adapter](https://github.com/ziyunhx/storm-net-adapter) 最新版本为 2.0.5，现在支持 .Net4.0+, .Net Core。下面给大家介绍如何快速开始创建一个 DRPC Topology：

 ## 1. 创建.Net DRPC Topology项目 ##

 - 使用命令创建新的 .Net Core 项目：

{% codeblock %}
mkdir StormSample
cd StormSample
dotnet new
{% endcodeblock %}

 - 在项目中添加 Storm.Net.Adapter 引用：

{% codeblock %}
"dependencies": {
    "Storm.Net.Adapter": "2.0.5"
}
{% endcodeblock %}

 - 创建一个 SimpleDRPC 类：

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
 
 ## 2. 使用Java封装Storm Topology ##
 
{% codeblock lang:java %}
package org.apache.storm.starter;

import org.apache.storm.Config;
import org.apache.storm.LocalCluster;
import org.apache.storm.LocalDRPC;
import org.apache.storm.StormSubmitter;
import org.apache.storm.drpc.DRPCSpout;
import org.apache.storm.drpc.LinearDRPCTopologyBuilder;
import org.apache.storm.drpc.ReturnResults;
import org.apache.storm.spout.ShellSpout;
import org.apache.storm.task.ShellBolt;
import org.apache.storm.topology.IRichBolt;
import org.apache.storm.topology.IRichSpout;
import org.apache.storm.topology.OutputFieldsDeclarer;
import org.apache.storm.topology.TopologyBuilder;
import org.apache.storm.tuple.Fields;

import java.util.Map;

/**
 * This topology demonstrates Storm's stream groupings and multilang capabilities.
 */
public class DrpcTestTopologyCsharp {
	public static class SimpleDRPC extends ShellBolt implements IRichBolt {

		public SimpleDRPC() {
			super("dotnet", "StormSample.dll", "SimpleDRPC");
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

 - 注意调用代码中的 StormSimple.dll：

{% codeblock %}
super("dotnet", "StormSimple.dll", "SimpleDRPC");
{% endcodeblock %}

 - 打包 StormSample ，并将依赖项一起拷贝到 multilang/resources 。

 - 使用 Maven 打包 Java 项目，通过 storm jar 命令提交：

{% codeblock %}
$ storm jar storm-starter-1.0.1.jar org.apache.storm.starter.DrpcTestTopologyCsharp simpledrpc
{% endcodeblock %}

 ## 3. 使用C#调用刚刚提交的DRPC Topology ##

{% codeblock lang:csharp %}
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Storm;

namespace Storm.DRPC.Demo
{
    public class Program
    {
        public static void Main(string[] args)
        {
            DRPCClient client = new DRPCClient("host", 3772);
            string result = client.execute("simpledrpc", "hello word");
            Console.WriteLine(result);
            Console.WriteLine("Please input a word and press enter, if you want quit it, press enter only!");
            do
            {
                string input = Console.ReadLine();
                if (string.IsNullOrEmpty(input))
                    break;

                Console.WriteLine(client.execute("simpledrpc", input));
            }
            while (true);
        }
    }
}
{% endcodeblock %}

 至此就全部结束了！所有的代码都在 [storm-net-adapter](https://github.com/ziyunhx/storm-net-adapter) 可以找到，你可以通过 Star 与 Fork 该项目来支持我！