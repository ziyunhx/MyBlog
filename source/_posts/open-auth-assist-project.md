title: Open Auth辅助库（使用ImitateLogin实现登录）
date: 2015-08-10
categories: 
- 爬虫
tags:
- csharp
- C#
- .NET
- ImitateLogin
- OpenAuth
- open-auth-assist

---

 网络上越来越多的公司进行着自己的平台化策略，其中绝大多数都已Web API的方式对外提供服务，为了方便的使用这些服务，你不得不引用许多相关的类库，但是API的本质其实仅仅是一些约定的网络请求，我们大多数情况仅仅使用API提供的少数几个功能，因此，我稍微修改了下微博的c#的类库，加入了[ImitateLogin](https://github.com/ziyunhx/imitate-login "ImitateLogin")库来模拟登录，形成了[open-auth-assist](https://github.com/ziyunhx/open-auth-assist "open-auth-assist")库。

<!--more-->
 [open-auth-assist](https://github.com/ziyunhx/open-auth-assist)的目的是将现有的API的类库使用一种通用的方式来代替，同时又不增加太多的额外工作。另外这个项目也可以算作我的另一个开源项目[ImitateLogin](https://github.com/ziyunhx/imitate-login)的一个Demo。

 这个类库的绝大多数代码都源于 [weiboSDK](http://weibosdk.codeplex.com/ "WeiboSDK") 这个项目，由于作者不准备再更新，而且没有继续提供模拟登录的功能，所以我拿过来修改了下开源出来（已获得原作者同意），目前仅完成了微博部分的实现，由于这个项目依赖于[ImitateLogin](https://github.com/ziyunhx/imitate-login)，因此只有[ImitateLogin](https://github.com/ziyunhx/imitate-login)完成的网站才会增加支持；[ImitateLogin](https://github.com/ziyunhx/imitate-login)本身并没有太多的技术难度，仅仅需要熟悉网络请求和一些耐心来解决各种客户端加密，所以如果大家有时间，也希望能一起为这个项目贡献一些代码，谢谢！

 下面将简单介绍下如何使用[open-auth-assist](https://github.com/ziyunhx/open-auth-assist)来实现微博API的调用。

 首先，我们需要实例化一个OpenAuthAssist类：

{% codeblock lang:csharp %}
 var openAuth = new SinaWeiboClient("1402038860", "62e1ddd4f6bc33077c796d5129047ca2", "http://qcyn.sina.com.cn");
{% endcodeblock %}

 例子中使用的appkey使用了原作者例子中的key。

 接下来我们登录需要进行操作的用户：

{% codeblock lang:csharp %}
 openAuth.DoLogin("username", "password");
{% endcodeblock %}

 然后我们来使用Weibo提供的获取用户时间轴的API来展示如何使用Get：

{% codeblock lang:csharp %}
 var result = openAuth.HttpGet("statuses/friends_timeline.json", new Dictionary<string, object>
			{
				{"count", 5},
				{"page", 1},
				{"base_app" , 0}
			});
{% endcodeblock %}

 我们发送一条微博来展示Post方法的调用：

{% codeblock lang:csharp %}
 var result2 = openAuth.HttpPost("statuses/update.json", new Dictionary<string, object>
			{
				{"status" , string.Format("post from OpenAuth.Assist! @{0:HH:mm:ss}", DateTime.Now)}
			});
{% endcodeblock %}

 接下来，好好享受吧！
