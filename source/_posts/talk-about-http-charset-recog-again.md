title: 再谈网络爬虫中的编码识别问题
date: 2016-04-19
categories: 
- 爬虫
tags:
- csharp
- C#
- .NET
- charset

---

在之前的文章中，我给大家介绍了 Nchardet 结合网页头部声明来识别网页的编码。通过较长时间段生产环境的使用，效果并不是十分理想。首先是 Nchardet 带来了极大的CPU的开销，尤其是对大规模的爬虫集群来说几乎无法接受；其次猜测的准确性距离100%还有一段距离。因此，就有了今天的这篇文章。

<!--more-->
Nchardet 是通过尝试转换编码比较错误率来实现编码猜测的，由于网页的长度与中文密度的关系，效果并不理想。之前的方法中默认采用了这种方案，导致了较大的CPU资源的开销。下面将介绍新的解析思路：

首先我们通过解析 HTTP 请求返回的 ContentType 来猜测编码，.Net的网络请求库中的 HttpClient 就是使用了这一方法，实现代码如下：

{% codeblock lang:csharp %}
string Hcharset = "";
string text = "";
if (!string.IsNullOrEmpty(text = httpWebResponse.ContentType))
{
    text = text.ToLower(CultureInfo.InvariantCulture);
    string[] array = text.Split(new char[] { ';', '=', ' ' });
    bool flag = false;
    string[] array2 = array;
    for (int i = 0; i < array2.Length; i++)
    {
        string text2 = array2[i];
        if (text2 == "charset")
            flag = true;
        else
        {
            if (flag)
                Hcharset = text2;
        }
    }
}
{% endcodeblock %}

其中 httpWebResponse 为网络请求返回的结果。

然后我们使用正则来解析用户在网页中声明的编码：

{% codeblock lang:csharp %}
string CharsetReg = @"(meta.*?charset=""?(?<Charset>[^\s""'>;]+)""?)|(xml.*?encoding=""?(?<Charset>[^\s"">;]+)""?)";

string Rcharset = "";
String cache = string.Empty;

while (true)
{
    var b = ResponseStream.ReadByte();
    if (b < 0) //end of stream
        break;
    bytes.Add((byte)b);

    if (!cache.EndsWith("</head>", StringComparison.OrdinalIgnoreCase))
        cache += (char)b;
}

Match match = Regex.Match(cache, CharsetReg, RegexOptions.IgnoreCase | RegexOptions.Multiline);
if (match.Success)
    Rcharset = match.Groups["Charset"].Value;
{% endcodeblock %}

由于两种方法取到的结果都有不准确的可能，因此当它们相同时我们就采用该种方式解析网页获得内容，避免使用 Nchardet 带来性能损耗。当两者不同时我们才使用 Nchardet 进行猜测，同样与之前的两种结果比较选择相同的方式解析。

{% codeblock lang:csharp %}
if (!string.IsNullOrEmpty(Rcharset) && !string.IsNullOrEmpty(Hcharset) && Hcharset.ToUpper() == Rcharset.ToUpper())
    encode = Encoding.GetEncoding(Hcharset);
else
{
    Ncharset = NChardetHelper.RecogCharset(bytes.ToArray(), Thrinax.Data.NChardetLanguage.CHINESE, 1024);

    if (!string.IsNullOrEmpty(Ncharset) && (Ncharset.ToUpper() == Rcharset.ToUpper() || Ncharset.ToUpper() == Hcharset.ToUpper()))
        encode = Encoding.GetEncoding(Ncharset);
}
{% endcodeblock %}

如果以上方法还是得不到相同的编码时，我们使用人工标注的编码（如果有的话）。接下来我们使用以下顺序来解析网页，直到获取到使用的编码：
人工标注的编码 > 网页自动识别 > 解析ContentType > 解析Html编码声明 。

{% codeblock lang:csharp %}
 //2，使用人工标注的编码
if (encode == null && !string.IsNullOrEmpty(encoding))
{
    try
    {
        encode = Encoding.GetEncoding(encoding);
    }
    catch { }
}

//3，使用单一方式识别出的编码，网页自动识别 > 解析ContentType > 解析Html编码声明
if (encode == null && !string.IsNullOrEmpty(Ncharset))
    encode = Encoding.GetEncoding(Ncharset);
if(encode == null && !string.IsNullOrEmpty(Hcharset))
    encode = Encoding.GetEncoding(Hcharset);
if (encode == null && !string.IsNullOrEmpty(Rcharset))
    encode = Encoding.GetEncoding(Rcharset);
{% endcodeblock %}

如果都没有的话，听天由命吧，直接使用默认编码解析。下面是该逻辑和取出内容的部分：

{% codeblock lang:csharp %}
//4，使用默认编码，听天由命吧
if (encode == null)
    encode = Encoding.Default;

Content = encode.GetString(bytes.ToArray());
ResponseStream.Close();
{% endcodeblock %}

ok，大功告成！所有代码以及里面的一些依赖都已经在 [thrinax](https://github.com/ziyunhx/thrinax) 库中了，编码识别模块通过长时间生产环境爬虫使用，统计结果显示识别准确率大于 99.99%。相比之前方式的性能提高数倍。[thrinax](https://github.com/ziyunhx/thrinax) 的定位是一个使用 .Net 提供网络抓取，信息抽取，自然语言处理的库，其中的大部分代码都会来源于社区或其它开源算法或者论文，该库只是筛选统一方便调用。如果大家有这三方面的好的推荐，可以通过 Issue 或者博客评论发给我！

本文涉及 [HttpHelper](https://github.com/ziyunhx/thrinax/blob/master/Helper/HttpHelper.cs) 与 [NChardetHelper](https://github.com/ziyunhx/thrinax/blob/master/Helper/NChardetHelper.cs)，欢迎大家 Star 与 Fork [thrinax](https://github.com/ziyunhx/thrinax) 库。