title: ImitateLogin新增插件机制以及又一个社交网站的支持
date: 2016-1-9
categories: 
- 爬虫
tags:
- social network
- thrift
- 微信
- C#
- .NET
- wechat
- 插件机制

---

我的文章里已经多次介绍 [imitate-login](https://github.com/ziyunhx/imitate-login) ，这是我最近一直在维护的一个使用c#模拟社交网站登录的开源项目，现在新增了对插件的支持以及一个新的网站（由于某种原因，会在文章结束部分介绍；而且仅会出现在博客中）。希望喜欢的读者可以通过 **Star & fork** 来支持我，我也会据此来决定时间的分配。
 
 <!--more-->
 
说点无关的东西，最近把博客从 GitHub Page 迁移到了 Azure，为了减轻域名与服务器的费用，在文章的正文开头和结尾放了点小广告，如果影响了你的阅读体验，请自行使用 Adblock Plus。 

好了，言归正常！[imitate-login](https://github.com/ziyunhx/imitate-login) 现在已经提供了对插件的支持；目前有两个部分使用到了插件机制，登录自身实现以及登录过程中的验证码识别过程；其中登录过程仅支持 MEF（Managed Extensibility Framework）模式，而验证码识别过程支持 Thrift RPC (Apache Thrift)、HTTP RESTful (POST/GET)、MEF 三种方式。下面将介绍这三种插件的开发与配置方式，所有代码均已经在 [Extensions](https://github.com/ziyunhx/imitate-login/tree/master/Extensions) 。

### Thrift RPC ###

在 Imitate Login 的库中有通过 Thrift 文件生成的类 ThriftOperation，如果你使用其它语言开发，请通过 Thrift 生成对应的类，下面将介绍使用C#来开发插件。

首先，创建一个控制台应用程序，新增一个类继承 ThriftOperation.Iface 并实现，这里直接 return 一下：

{% codeblock lang:csharp %}
class demo : ThriftOperation.Iface
{
    public string Operation(OperationObj operationObj)
    {
        return "1234";
    }
}
{% endcodeblock %}

然后在主函数里增加一个创建 Thrift 服务端得方法：

{% codeblock lang:csharp %}
static void Main(string[] args)
{
    int port = 7801;

    string str = ConfigurationManager.AppSettings["ServerPort"];
    if (!string.IsNullOrEmpty(str))
        int.TryParse(str, out port);

    if (args != null && args.Length == 1)
    {
        int.TryParse(args[0], out port);
    }

    Start(port);
}

public static void Start(int port)
{
    TServerSocket serverTransport = new TServerSocket(port, 0, false);
    ThriftOperation.Processor processor = new ThriftOperation.Processor(new demo());
    TServer server = new TSimpleServer(processor, serverTransport);
    Console.WriteLine("Starting server on port {0} ...", port);
    server.Serve();
}
{% endcodeblock %}

在使用时，你需要先启动该插件程序，然后将下面配置部分合并放到程序运行目录的 extension.conf 文件：

{% codeblock %}
{
    "ExtendType": 3,
    "SupportSite": [2],
    "Path": null,
    "Host": "127.0.0.1",
    "Port": 7801,
    "UrlFormat": null,
    "HttpMethod": null
}
{% endcodeblock %}

你可以使用 [PluginConfigBuild](https://github.com/ziyunhx/imitate-login/tree/master/Tools/PluginConfigBuild) 工具来生成配置文件，此处不再解释具体细节。

### HTTP RESTful ###

另外一种插件方式即使用通用的 Http RESTful API 来实现，如果通过 GET 方法，你仅能传入一个枚举用来表明网站以及一个字符串作为参数；如果你通过 POST 方法，需要通过 Thrift 获得一个 OperationObj 类的定义，当然 C# 可以通过直接引用 [imitate-login](https://github.com/ziyunhx/imitate-login) 库来获得。API 的编写方法不再累述，接下来你需要将以下配置部分合并放到程序运行目录的 extension.conf 文件：

{% codeblock %}
{
    "ExtendType": 2,
    "SupportSite": [6],
    "Path": null,
    "Host": null,
    "Port": 0,
    "UrlFormat": "http://localhost:2920/Mail/SendMail?loginSite={0}&imageUrl={1}",
    "HttpMethod": "GET"
}
{% endcodeblock %}

### MEF ###

MEF 是微软在 .NET 4.0 以后原生提供的一种插件模式；使用该方法需要用到 IMEFOperation 类，你需要通过引用 [imitate-login](https://github.com/ziyunhx/imitate-login) 得到，demo 代码如下：

{% codeblock lang:csharp %}
[Export(typeof(IMEFOperation))]
[ExportMetadata("loginSite", LoginSite.Baidu)]
public class demo : IMEFOperation
{
    public string Operate(string imageUrl = "", Image image = null, params string[] param)
    {
        return "1234";
    }
}
{% endcodeblock %}

这种方式需要在配置文件中指定插件的存放位置，位置支持相对运行目录或绝对目录；本例为将该插件生成的 dll 拷贝到程序运行目录下的 Extensions 目录中，配置文件如下：

{% codeblock %}
{
    "ExtendType": 1,
    "SupportSite": [5, 1],
    "Path": "Extensions",
    "Host": null,
    "Port": 0,
    "UrlFormat": null,
    "HttpMethod": null
}
{% endcodeblock %}

所有的配置文件均可以通过 [PluginConfigBuild](https://github.com/ziyunhx/imitate-login/tree/master/Tools/PluginConfigBuild) 工具来生成，其中 SupportSite 为支持的登录网站的枚举数组。

----

以下内容将仅在博客中展示！


<script async src="//pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
<!-- 博客正文广告 -->
<ins class="adsbygoogle"
     style="display:block"
     data-ad-client="ca-pub-6541442242935379"
     data-ad-slot="9453052931"
     data-ad-format="auto"></ins>
<script>
(adsbygoogle = window.adsbygoogle || []).push({});
</script>

为了展示插件机制，我将微信网页版的登录增加到了库中；该方式为登录时将二维码通过邮件发送到指定的邮箱，人工使用手机微信扫描后登录微信。该方法仅供展示插件机制，无生成环境使用价值，请轻喷！

修改 [MailNotication](https://github.com/ziyunhx/imitate-login/tree/master/Extensions/MailNotication) 插件的 web.config 中的发信邮箱与收信邮箱设置，在 IIS 中部署好WebAPI，配置好插件配置文件，即可测试微信登录功能。

----

好了，接下来需要什么自己动手试试吧！