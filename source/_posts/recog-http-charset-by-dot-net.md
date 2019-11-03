title: 使用C#自动识别网页编码
date: 2016-01-27
categories: 
- 爬虫
tags:
- csharp
- C#
- .NET
- charset

---

在大规模的网络爬虫编程中，网页编码识别是必不可少的，本文将介绍如何通过C#来自动识别网页的编码。文中会分析几种可行的方法并提供源码，部分源码来源于开源代码改写而来。

<!--more-->
网页在传输中为了能够正确识别编码，一般在头部都会声明编码格式，例如：

{% codeblock %}
<meta charset="utf-8">
{% endcodeblock %}

或者像这样：

{% codeblock %}
<meta http-equiv="Content-type" content="text/html; charset=utf-8">
{% endcodeblock %}

因此，很自然的我们想到通过读取这个字段来来识别编码。下面代码中 ResponseStream 是从 HTTP 网络流里读取后转化为 Stream 的流。

{% codeblock lang:csharp %}
string CharsetReg = @"(meta.*?charset=""?(?<Charset>[^\s""'>;]+)""?)|(xml.*?encoding=""?(?<Charset>[^\s"">;]+)""?)";

Encoding encode = null;
String cache = string.Empty;

while (true)
{
    var b = (byte)ResponseStream.ReadByte();
    if (b < 0 || b == 255) //end of stream
        break;

    if (!cache.EndsWith("</head>", StringComparison.OrdinalIgnoreCase))
        cache += (char)b;
    else
        break;
}

if (httpWebResponse.CharacterSet == "ISO-8859-1" || httpWebResponse.CharacterSet == "zh-cn")
{
    Match match = Regex.Match(cache, CharsetReg, RegexOptions.IgnoreCase | RegexOptions.Multiline);
    if (match.Success)
    {
        try
        {
            charset = match.Groups["Charset"].Value;
            encode = Encoding.GetEncoding(charset);
        }
        catch { }
    }
}

if (httpWebResponse.CharacterSet != null && encode == null)
    encode = Encoding.GetEncoding(httpWebResponse.CharacterSet);
{% endcodeblock %}

这个方法仅考虑了中文与英文环境下的情况，如果还有其它编码，请自行修改。

当你满心欢喜的以为可以喝杯咖啡打个盹时，运维妹妹发来了乱码的截图。。。这是哪个实习生在网页写的GB2312，实际却是UTF-8的编码。好吧，看在就这一个的情况下，我写个参数配置一下这个网站的编码吧。

一个小时过去了。。。

已经是第7张乱码截图了，看来乱写编码已经成为了反爬虫的招数之一了！再一次打开网站，发现浏览器中并没有乱码，看来是有其它的判断方式。于是，找到了这个库 [chardet](http://www-archive.mozilla.org/projects/intl/chardet.html) ，它是mozilla自动字符编码识别程序库，我直接使用了社区中贡献的 .Net 实现版本 [Nchardet](http://www.cnblogs.com/hhh/archive/2007/01/27/632251.html)。链接是我所能搜索到的最早的文章，如果有错误还望指出！

原程序使用了一种通知模式来反馈编码结果，个人感觉与习惯使用的方式不同，于是改成了静态方法，将 HandleData 等用于识别编码的方法增加了编码返回，另外创建了一个 Nchardet 的 Helper 类：

{% codeblock lang:csharp %}
using System;
using System.Linq;
using System.Text;
using Thrinax.Data;

namespace Thrinax.Helper
{
    public class NChardetHelper
    {
        /// <summary>
        /// Recog the Encoding from byte array.
        /// </summary>
        /// <param name="bytes">the byte array.</param>
        /// <param name="language">the language.</param>
        /// <returns>charset string, will be empty when can't recog.</returns>
        public static Encoding RecogEncoding(byte[] bytes, NChardetLanguage language = NChardetLanguage.ALL)
        {
            string charset = RecogCharset(bytes, language);
            if (!string.IsNullOrEmpty(charset))
                return Encoding.GetEncoding(charset); 

            return Encoding.Default;
        }

        /// <summary>
        /// Recog the charset from byte array.
        /// </summary>
        /// <param name="bytes">the byte array.</param>
        /// <param name="language">the language.</param>
        /// <param name="maxLength">max length per time. the default is 1024, -1 to without limit.</param>
        /// <returns>charset string, will be empty when can't recog.</returns>
        public static string RecogCharset(byte[] bytes, NChardetLanguage language = NChardetLanguage.ALL, int maxLength = 1024)
        {
            if (bytes == null || bytes.Length == 0)
                return null;

            PSMDetector detector = new PSMDetector(language);
            string charset = String.Empty;

            if (maxLength > 0)
            {
                int count = 0;

                do
                {
                    var tempBytes = bytes.Skip(maxLength * count).Take(maxLength);
                    if (tempBytes == null || tempBytes.Count() == 0)
                        break;

                    detector.HandleData(tempBytes.ToArray(), tempBytes.Count(), ref charset);
                    if (!string.IsNullOrEmpty(charset))
                        break;

                    count++;
                }
                while (true);
            }
            else
            {
                detector.HandleData(bytes, bytes.Length, ref charset);
            }

            if (string.IsNullOrEmpty(charset))
                detector.DataEnd(ref charset);

            return charset;
        }
    }
}
{% endcodeblock %}

为了平衡效率与准确度，RecogCharset 方法提供了一个 maxLength 的参数，用于指定每次识别的最大byte数，如果指定长度的byte数组无法识别出编码，则会循环直至识别出编码或者所有byte都已参与识别。生成环境下测试该值使用 1024 效果较好。

你以为这样就完了，不，没有，竟然还有一个网站头部与正文部分使用不同编码的，为了避嫌，暂时不点名了。还好目前也就发现了这一个网站，好吧，拿回之前删除的指定编码的代码，把编码指定为我们要获取的部分的编码，最终改写后的方法如下：

{% codeblock lang:csharp %}
string CharsetReg = @"(meta.*?charset=""?(?<Charset>[^\s""'>;]+)""?)|(xml.*?encoding=""?(?<Charset>[^\s"">;]+)""?)";

string Content = null;
var bytes = new List<byte>();
Encoding encode = null;
String cache = string.Empty;

while (true)
{
    var b = (byte)ResponseStream.ReadByte();
    if (b < 0 || b == 255) //end of stream
        break;
    bytes.Add(b);

    if (!cache.EndsWith("</head>", StringComparison.OrdinalIgnoreCase))
        cache += (char)b;
}

// Charset check: input > NChardet > Parser
if (encode == null)
{
    string charset = NChardetHelper.RecogCharset(bytes.ToArray());
    if (!string.IsNullOrEmpty(charset))
        encode = Encoding.GetEncoding(charset);

    if (encode == null)
    {
        if (httpWebResponse.CharacterSet == "ISO-8859-1" || httpWebResponse.CharacterSet == "zh-cn")
        {
            Match match = Regex.Match(cache, CharsetReg, RegexOptions.IgnoreCase | RegexOptions.Multiline);
            if (match.Success)
            {
                try
                {
                    charset = match.Groups["Charset"].Value;
                    encode = Encoding.GetEncoding(charset);
                }
                catch { }
            }
        }

        if (httpWebResponse.CharacterSet != null && encode == null)
            encode = Encoding.GetEncoding(httpWebResponse.CharacterSet);
    }
}

if (encode == null)
    encode = Encoding.Default;

Content = encode.GetString(bytes.ToArray());
ResponseStream.Close();
{% endcodeblock %}

所有代码以及里面的一些依赖都已经在 [thrinax](https://github.com/ziyunhx/thrinax) 库中了，[thrinax](https://github.com/ziyunhx/thrinax) 的定位是一个使用 .Net 提供网络抓取，信息抽取，自然语言处理的库，其中的大部分代码都会来源于社区或其它开源算法或者论文，该库只是筛选统一方便调用。如果大家有这三方面的好的推荐，可以通过 Issue 或者博客评论发给我！

本文涉及 [HttpHelper](https://github.com/ziyunhx/thrinax/blob/master/Helper/HttpHelper.cs) 与 [NChardetHelper](https://github.com/ziyunhx/thrinax/blob/master/Helper/NChardetHelper.cs)，欢迎大家 Star 与 Fork [thrinax](https://github.com/ziyunhx/thrinax) 库。