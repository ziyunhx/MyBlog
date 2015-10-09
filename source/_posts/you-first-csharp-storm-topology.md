title: 使用Csharp创建你的第一个Storm拓扑（wordcount）
date: 2015-06-04
categories: 
- Storm
tags: 
- Storm
- wordcount
- csharp
- C#
- .NET

---

 WordCount在大数据领域就像学习一门语言时的hello world，得益于Storm的开源以及[Storm.Net.Adapter](https://github.com/ziyunhx/storm-net-adapter "Storm.Net.Adapter")，现在我们也可以像Java或Python一样，使用Csharp创建原生支持的Storm Topologies。下面我将通过介绍wordcount来展示如何使用Csharp开发Storm拓扑。

<!--more-->
 上篇博客已经介绍了如何部署Storm开发环境，本文所讲述demo已包含在[Storm.Net.Adapter](https://github.com/ziyunhx/storm-net-adapter "Storm.Net.Adapter")中，如果你觉得对你有帮助，欢迎Star和Fork，让更多人看到来帮助完善这个项目。

 首先，我们创建一个控制台应用程序（使用控制台是方便调用） StormSimple；使用Nuget添加添加Storm.Net.Adapter（该类库的namespace为Storm）。

![wordcount project](http://www.tnidea.com/media/image/wordcount-project.png)

 **STEP1**：通过继承ISpout创建一个Spout：Generator，实现ISpout的四个方法：

{% codeblock lang:csharp %}
void Open(Config stormConf, TopologyContext context);
void NextTuple();
void Ack(long seqId);
void Fail(long seqId);
{% endcodeblock %}

 在实现这4个方法之前，我们还需要创建一些变量和方法来初始化这个类：
   
{% codeblock lang:csharp %}
private Context ctx;

public Generator(Context ctx)
{
	Context.Logger.Info("Generator constructor called");
	this.ctx = ctx;

	// Declare Output schema
	Dictionary<string, List<Type>> outputSchema = new Dictionary<string, List<Type>>();
	outputSchema.Add("default", new List<Type>() { typeof(string) });
	this.ctx.DeclareComponentSchema(new ComponentStreamSchema(null, outputSchema));
}
{% endcodeblock %}

 我使用了一个私有变量`ctx`来保存实例化时传入的Context对象，Context有一个静态的Logger，用于日志的发送，我们无需实例化即可使用它。根据日志级别不同，包含 Trace Debug Info Warn Error 五个级别，另外我们在实例化方法里还需要定义输入和输出的参数的数量和类型，本例子中输入为`null`，输出为一个字符串。另外我们还创建一个方法来直接返回实例化后的类：

{% codeblock lang:csharp %}
/// <summary>
///  Implements of delegate "newPlugin", which is used to create a instance of this spout/bolt
/// </summary>
/// <param name="ctx">Context instance</param>
/// <returns></returns>
public static Generator Get(Context ctx)
{
    return new Generator(ctx);
}
{% endcodeblock %}

 其中Open在该类第一次任务调用前执行，主要用于预处理和一些配置信息的传入，大多数情况下，我们并不需要做什么；NextTuple方法用于生成Tuple，会不断被调用，因此如果没什么任务要向下发送，可以使用`Thread.Sleep(50);`来减少CPU的消耗（具体休息时间与Topology设置有关，只要不超过超时时间就没有问题）。

 本例子中NextTuple主要用于从一个包含英语句子的数组中随机取出一条句子，并把它发送到下一个环节，为了能够保证所有的任务都被成功执行一遍，我们将发送的消息缓存起来，并且限制正在执行中的任务数量为20。

{% codeblock lang:csharp %}
private const int MAX_PENDING_TUPLE_NUM = 20;
private long lastSeqId = 0;
private Dictionary<long, string> cachedTuples = new Dictionary<long, string>();

private Random rand = new Random();
string[] sentences = new string[] {
                                  "the cow jumped over the moon",
                                  "an apple a day keeps the doctor away",
                                  "four score and seven years ago",
                                  "snow white and the seven dwarfs",
                                  "i am at two with nature"};
/// <summary>
/// This method is used to emit one or more tuples. If there is nothing to emit, this method should return without emitting anything. 
/// It should be noted that NextTuple(), Ack(), and Fail() are all called in a tight loop in a single thread in C# process. 
/// When there are no tuples to emit, it is courteous to have NextTuple sleep for a short amount of time (such as 10 milliseconds), so as not to waste too much CPU.
/// </summary>
public void NextTuple()
{
    Context.Logger.Info("NextTuple enter");
    string sentence;

    if (cachedTuples.Count <= MAX_PENDING_TUPLE_NUM)
    {
        lastSeqId++;
        sentence = sentences[rand.Next(0, sentences.Length - 1)];
        Context.Logger.Info("Generator Emit: {0}, seqId: {1}", sentence, lastSeqId);
        this.ctx.Emit("default", new List<object>() { sentence }, lastSeqId);
        cachedTuples[lastSeqId] = sentence;
    }
    else
    {
        // if have nothing to emit, then sleep for a little while to release CPU
        Thread.Sleep(50);
    }
    Context.Logger.Info("cached tuple num: {0}", cachedTuples.Count);

    Context.Logger.Info("Generator NextTx exit");
}
{% endcodeblock %}

 `this.ctx.Emit` 即用来把Topology发送给下一个Bolt。

 Ack()和Fail()方法分别在整个Topology执行成功和Topology失败时被调用。本例中Ack主要是移除缓存，Fail主要是用于取出缓存数据并重新发送Tuple。

{% codeblock lang:csharp %}
/// <summary>
/// Ack() will be called only when ack mechanism is enabled in spec file.
/// If ack is not supported in non-transactional topology, the Ack() can be left as empty function. 
/// </summary>
/// <param name="seqId">Sequence Id of the tuple which is acked.</param>
public void Ack(long seqId)
{
    Context.Logger.Info("Ack, seqId: {0}", seqId);
    bool result = cachedTuples.Remove(seqId);
    if (!result)
    {
        Context.Logger.Warn("Ack(), remove cached tuple for seqId {0} fail!", seqId);
    }
}

/// <summary>
/// Fail() will be called only when ack mechanism is enabled in spec file. 
/// If ack is not supported in non-transactional topology, the Fail() can be left as empty function.
/// </summary>
/// <param name="seqId">Sequence Id of the tuple which is failed.</param>
public void Fail(long seqId)
{
    Context.Logger.Info("Fail, seqId: {0}", seqId);
    if (cachedTuples.ContainsKey(seqId))
    {
        string sentence = cachedTuples[seqId];
        Context.Logger.Info("Re-Emit: {0}, seqId: {1}", sentence, seqId);
        this.ctx.Emit("default", new List<object>() { sentence }, seqId);
    }
    else
    {
        Context.Logger.Warn("Fail(), can't find cached tuple for seqId {0}!", seqId);
    }
}
{% endcodeblock %}

 至此，一个Spout就算完成了，下面我们继续分析Bolt。

 **STEP2**：通过继承IBasicBolt创建Bolt：Splitter、Counter。

 Splitter是一个通过空格来拆分英语句子为一个个独立的单词，Counter则用来统计各个单词出现的次数。我们只详细分析Splitter，Counter类仅贴出全部源码。

 和Generator相同，我们首先也要构造一个实例化方法方便使用者传参和调用：

{% codeblock lang:csharp %}
private Context ctx;
private int msgTimeoutSecs;

public Splitter(Context ctx)
{
    Context.Logger.Info("Splitter constructor called");
    this.ctx = ctx;

    // Declare Input and Output schemas
    Dictionary<string, List<Type>> inputSchema = new Dictionary<string, List<Type>>();
    inputSchema.Add("default", new List<Type>() { typeof(string) });
    Dictionary<string, List<Type>> outputSchema = new Dictionary<string, List<Type>>();
    outputSchema.Add("default", new List<Type>() { typeof(string), typeof(char) });
    this.ctx.DeclareComponentSchema(new ComponentStreamSchema(inputSchema, outputSchema));

    // Demo how to get stormConf info
    if (Context.Config.StormConf.ContainsKey("topology.message.timeout.secs"))
    {
        msgTimeoutSecs = Convert.ToInt32(Context.Config.StormConf["topology.message.timeout.secs"]);
    }
    Context.Logger.Info("msgTimeoutSecs: {0}", msgTimeoutSecs);
}

/// <summary>
///  Implements of delegate "newPlugin", which is used to create a instance of this spout/bolt
/// </summary>
/// <param name="ctx">Context instance</param>
/// <returns></returns>
public static Splitter Get(Context ctx)
{
    return new Splitter(ctx);
}
{% endcodeblock %}

 在这个实例化方法中，我们增加了一个没有使用的变量`msgTimeoutSecs`用来展示如何获取Topology的配置。

 由于继承了IBasicBolt，我们需要实现以下两个方法：

{% codeblock lang:csharp %}
void Prepare(Config stormConf, TopologyContext context);
void Execute(StormTuple tuple);
{% endcodeblock %}

 这和IBolt是一致的，IBasicBolt和IBolt的区别仅仅在于后者需要自己处理何时向Storm发送Ack或Fail，IBasicBolt则不需要关心这些，如果你的Execute没有抛出异常的话，总会在最后向Storm发送Ack，否则则发送Fail。Prepare则是用于执行前的预处理，此例子里同样什么都不需要做。

{% codeblock lang:csharp %}
/// <summary>
/// The Execute() function will be called, when a new tuple is available.
/// </summary>
/// <param name="tuple"></param>
public void Execute(StormTuple tuple)
{
    Context.Logger.Info("Execute enter");

    string sentence = tuple.GetString(0);

    foreach (string word in sentence.Split(' '))
    {
        Context.Logger.Info("Splitter Emit: {0}", word);
        this.ctx.Emit("default", new List<StormTuple> { tuple }, new List<object> { word, word[0] });
    }

    Context.Logger.Info("Splitter Execute exit");
}

public void Prepare(Config stormConf, TopologyContext context)
{
    return;
}
{% endcodeblock %}

 Counter和上述的代码类似：
 
{% codeblock lang:csharp %}
using Storm;
using System;
using System.Collections.Generic;

namespace StormSample
{
    /// <summary>
    /// The bolt "counter" uses a dictionary to record the occurrence number of each word.
    /// </summary>
    public class Counter : IBasicBolt
    {
        private Context ctx;

        private Dictionary<string, int> counts = new Dictionary<string, int>();

        public Counter(Context ctx)
        {
            Context.Logger.Info("Counter constructor called");

            this.ctx = ctx;

            // Declare Input and Output schemas
            Dictionary<string, List<Type>> inputSchema = new Dictionary<string, List<Type>>();
            inputSchema.Add("default", new List<Type>() { typeof(string), typeof(char) });

            Dictionary<string, List<Type>> outputSchema = new Dictionary<string, List<Type>>();
            outputSchema.Add("default", new List<Type>() { typeof(string), typeof(int) });
            this.ctx.DeclareComponentSchema(new ComponentStreamSchema(inputSchema, outputSchema));
        }

        /// <summary>
        /// The Execute() function will be called, when a new tuple is available.
        /// </summary>
        /// <param name="tuple"></param>
        public void Execute(StormTuple tuple)
        {
            Context.Logger.Info("Execute enter");

            string word = tuple.GetString(0);
            int count = counts.ContainsKey(word) ? counts[word] : 0;
            count++;
            counts[word] = count;

            Context.Logger.Info("Counter Emit: {0}, count: {1}", word, count);
            this.ctx.Emit("default", new List<StormTuple> { tuple }, new List<object> { word, count });

            Context.Logger.Info("Counter Execute exit");
        }

        /// <summary>
        ///  Implements of delegate "newPlugin", which is used to create a instance of this spout/bolt
        /// </summary>
        /// <param name="ctx">Context instance</param>
        /// <returns></returns>
        public static Counter Get(Context ctx)
        {
            return new Counter(ctx);
        }

        public void Prepare(Config stormConf, TopologyContext context)
        {
            return;
        }
    }
}
{% endcodeblock %}

 **STEP3**：修改Program.cs来方便使用Java调用。

{% codeblock lang:csharp %}
using Storm;
using System;
using System.Linq;

namespace StormSample
{
    class Program
    {
        static void Main(string[] args)
        {
            if (args.Count() > 0)
            {
                string compName = args[0];

                try
                {
                    if ("generator".Equals(compName))
                    {
                        ApacheStorm.LaunchPlugin(new newPlugin(Generator.Get));
                    }
                    else if ("splitter".Equals(compName))
                    {
                        ApacheStorm.LaunchPlugin(new newPlugin(Splitter.Get));
                    }
                    else if ("counter".Equals(compName))
                    {
                        ApacheStorm.LaunchPlugin(new newPlugin(Counter.Get));
                    }
                    else
                    {
                        throw new Exception(string.Format("unexpected compName: {0}", compName));
                    }
                }
                catch (Exception ex)
                {
                    Context.Logger.Error(ex.ToString());
                }
            }
            else
            {
                Context.Logger.Error("Not support local model.");
            }
        }
    }
}
{% endcodeblock %}

 我们在Main方法里使用参数来确定具体调用的是哪个Spout/Bolt，ApacheStorm是一个包含主要方法的类，之所以不使用Storm只是因为命名空间占用了它。Csharp端的代码到此就全部结束了，Java端的代码与部署发布将在下一篇详细介绍，敬请期待！下面让我们来看一看整个Topology的流程吧！

 ![WordCount Topology](http://www.tnidea.com/media/image/wordcount-topology.png)

