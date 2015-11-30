title: 使用ImitateLogin模拟登录百度
date: 2015-11-30
categories: 
- 爬虫
tags:
- social network
- thrift
- csharp
- C#
- .NET
- Baidu

---

在之前的文章中，我已经介绍过一个社交网站模拟登录的类库：[imitate-login](https://github.com/ziyunhx/imitate-login) ，这是一个通过c#的HttpWebRequest来模拟网站登录的库，之前实现了微博网页版和微博Wap版；现在，模拟百度登录的部分也已经完成。
 
 <!--more-->
 
由于个人时间的限制，加上目前有多个项目在同时进行，因此更新频率会根据项目关注度来决定（**Star & fork**）。这个类库的使用方法非常简单，仅对外提供一个方法： 
 
`LoginResult Login(1: string userName, 2: string password, 3: LoginSite loginSite);` 
 
这个方法位于ImitateLogin的LoginHelper类中，使用之前需要先对其进行实例化。通过传入 用户名、密码以及登录的网站，返回一个包含登录结果状态、描述信息和Cookies字典的类。它通过 Thrift 来实现多语言的支持。
 
下面将通过介绍模拟百度登录的实现来介绍如何进行扩充与二次开发：
 
首先，创建百度登录类 BaiduLogin.cs 继承 ILogin 接口；实现其生成的 DoLogin 方法。
 
{% codeblock lang:csharp %}
#region ILogin implementation
public LoginResult DoLogin(string UserName, string Password)
{
    throw new NotImplementedException();
}

public CookieContainer cookies { set; get;}
#endregion
{% endcodeblock %}

然后我们通过监听百度登录过程中的网络请求，梳理出修改过Cookies和最终提交登录所需的参数的请求。

Step1: 访问以下链接生成初始Cookies：https://passport.baidu.com/passApi/html/_blank.html 。

Step2: 获取最终登录提交所需的token：

{% codeblock lang:csharp %}
//1. Get the token.
string token_url = string.Format("https://passport.baidu.com/v2/api/?getapi&tpl=mn&apiver=v3&tt={0}&class=login&gid={1}&logintype=dialogLogin&callback=bd__cbs__{2}", TimeHelper.ConvertDateTimeInt(DateTime.Now), Guid.NewGuid().ToString().ToUpper(), build_callback());
string prepareContent = HttpHelper.GetHttpContent(token_url, null, cookies, referer: "https://www.baidu.com/", encode: Encoding.GetEncoding("GB2312"), cookiesDomain: "passport.baidu.com");
//string prepareJson = prepareContent.Split('(')[1].Split(')')[0];
dynamic prepareJson = JsonConvert.DeserializeObject(prepareContent.Split('(')[1].Split(')')[0]);
string token = prepareJson.data.token;
{% endcodeblock %}

其中 build_callback 为随机生成6位字母或数字的组合的方法。

Step3: 获取用于加密密码的publickey：

{% codeblock lang:csharp %}
//2. Get public key
string pubkey_url = "https://passport.baidu.com/v2/getpublickey?token={0}&tpl=mn&apiver=v3&tt={1}&gid={2}&callback=bd__cbs__{3}";
string pubkeyContent = HttpHelper.GetHttpContent(string.Format(pubkey_url, token, TimeHelper.ConvertDateTimeInt(DateTime.Now), Guid.NewGuid().ToString().ToUpper(), build_callback()), null, cookies, referer: "https://www.baidu.com/", encode: Encoding.GetEncoding("GB2312"), cookiesDomain: "passport.baidu.com");

dynamic pubkeyJson = JsonConvert.DeserializeObject(pubkeyContent.Split('(')[1].Split(')')[0]);
rsa_pub_baidu = pubkeyJson.pubkey;
string KEY = pubkeyJson.key;
{% endcodeblock %}

Step4: 模拟执行最终的登录：

{% codeblock lang:csharp %}
//3. Build post data
string login_data = "staticpage=https%3A%2F%2Fwww.baidu.com%2Fcache%2Fuser%2Fhtml%2Fv3Jump.html&charset=UTF-8&token={0}&tpl=mn&subpro=&apiver=v3&tt={1}&codestring=&safeflg=0&u=https%3A%2F%2Fwww.baidu.com%2F&isPhone=&detect=1&gid={2}&quick_user=0&logintype=dialogLogin&logLoginType=pc_loginDialog&idc=&loginmerge=true&splogin=rate&username={3}&password={4}&verifycode=&mem_pass=on&rsakey={5}&crypttype=12&ppui_logintime={6}&countrycode=&callback=parent.bd__pcbs__{7}";
login_data = string.Format(login_data, token, TimeHelper.ConvertDateTimeInt(DateTime.Now), Guid.NewGuid().ToString().ToUpper(), HttpUtility.UrlEncode(UserName), HttpUtility.UrlEncode(get_pwa_rsa(Password)), HttpUtility.UrlEncode(KEY), stopwatch.ElapsedMilliseconds, build_callback());

//4. Post the login data
string login_url = "https://passport.baidu.com/v2/api/?login";
HttpHelper.GetHttpContent(login_url, login_data, cookies, referer: "https://www.baidu.com/", cookiesDomain: "passport.baidu.com");
{% endcodeblock %}

stopwatch 是一个记录从最初执行到最终提交之前的耗时的一个计时器，get_pwa_rsa 为加密密码的方法。

Step5：验证最终的登录结果：

{% codeblock lang:csharp %}
string home_url = "https://www.baidu.com";
string result = HttpHelper.GetHttpContent(home_url, cookies: cookies, cookiesDomain: "passport.baidu.com");
//5. Verifty the login result
if (string.IsNullOrWhiteSpace(result) || result.Contains("账号存在异常") || !result.Contains("bds.comm.user=\""))
{
    return new LoginResult() { Result = ResultType.AccounntLimit, Msg = "Fail, Msg: Login fail! Maybe you account is disable or captcha is needed." };
}
{% endcodeblock %}

Step6：创建返回结果类：

{% codeblock lang:csharp %}
LoginResult loginResult = new LoginResult() { Result = ResultType.Success, Msg = "Success", Cookies = HttpHelper.GetAllCookies(cookies) };
{% endcodeblock %}

至此，模拟登录部分的代码就完成了，为了能够被其它程序调用，你还需要在 LoginSite 的枚举中新增一条来标识这个登录方法，此处增加了一个 Baidu = 5，并设置 [Description("Baidu")]。

然后在 LoginHelper.cs 的 Login 方法中的 switch (loginSite) 里增加一个 case：

{% codeblock lang:csharp %}
case LoginSite.Baidu:
    LoginClass = new BaiduLogin ();
    break;
{% endcodeblock %}

好了，大功告成！Todo List中还有淘宝、QQ、Facebook、Twitter、Google要做呢，我还想加入GitHub、Wechat...
现在，你可以帮我了吗？