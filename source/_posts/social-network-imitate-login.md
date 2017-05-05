title: 开源一个社交网站模拟登录的库
date: 2015-06-27
categories: 
- 爬虫
tags:
- social network
- thrift
- csharp
- C#
- .NET

---

 网站的登录是抓取某些网站的必须步骤，大多数情况我们都是使用一个真实的浏览器去提交我们的登录信息，但是在代码中嵌套浏览器不仅会带来性能损耗，还会带来崩溃的风险。因此就有了这个使用httpRequest来模拟登录的库 **[imitate-login](https://github.com/ziyunhx/imitate-login)** ，目前仅有微博网页版和微博Wap版的实现，其它计划实现会根据项目关注度来决定（**Star & fork**）是否更新以及更新时间。
 
 <!--more-->
 **如果这个项目侵犯了您的权益，请及时与我联系（可通过留言或邮件）！我会在收到的一周内协商处理。**
 
 这个类库仅对外提供一个方法： 
 
 `LoginResult Login(1: string userName, 2: string password, 3: LoginSite loginSite);` 
 
 这个方法位于ImitateLogin的LoginHelper类中，使用之前需要先对其进行实例化。通过传入 用户名、密码以及登录的网站，返回一个包含登录结果状态、描述信息以及Cookies字典的类。
 
 这个类库并没有提供对验证码的支持，微博可以通过设置登录保护来避免验证码的出现：
 
 位于 设置->账号安全->登录保护
 
 ![image](https://www.tnidea.com/media/image/imitate-login-weibo-setting.png)
 
 这个项目支持使用 Apache Thrift 来实现多语言环境下的RPC调用，首先安装Thrift，然后使用以下命令创建目标语言下的接口：
 
 `thrift --gen <language> ImitateLogin.thrift`
 
将上面命令中的`<language>`替换为你所使用的语言。Thrift 目前支持以下参数所代表的语言：
 
 - as3
 - c_glib
 - cpp
 - csharp
 - delphi
 - erl
 - go
 - hs
 - java
 - js
 - ocaml
 - perl
 - php
 - py
 - rb

然后在csharp端添加服务端得代码：
 
{% codeblock lang:csharp %}
public void Start() 
{ 
	TServerSocket serverTransport = new TServerSocket(7901, 0, false); 
	Login.Processor processor = new Login.Processor(new LoginHelper()); 
	TServer server = new TSimpleServer(processor, serverTransport); 
	Console.WriteLine("Starting server on port 7901 ..."); 
	server.Serve(); 
}
{% endcodeblock %}

 然后在其它语言（例如JAVA）中实现客户端的方法：
 
{% codeblock lang:java %}
TTransport transport = new TSocket("localhost", 7901);
transport.open();
TProtocol protocol = new TBinaryProtocol(transport);
Login.Client client = new Login.Client(protocol);
client.Login("username", "password", LoginSite.Weibo);
{% endcodeblock %}

将上述语句中的 username 和 password 替换为真实用于登录的微博账户。
 
你可以在包含Mono或.Net Framework的环境下运行 **[imitate-login](https://github.com/ziyunhx/imitate-login)** 类库。该类库里包含一个使用Gtk+创建的测试窗体程序，如果你希望使用它，需要额外安装 Gtk+ for Mono.
 
该类库已经完成的社交网站支持：

 - Weibo
 - WeiboWap

计划完成的支持：

 - Taobao
 - QQ
 - Facebook
 - Twitter
 - Google

计划支持部分会根据项目关注度来决定（**Star & fork**）是否更新以及更新时间。