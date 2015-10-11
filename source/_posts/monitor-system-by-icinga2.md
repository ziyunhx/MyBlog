title: 使用Icinga2和Icinga Web2搭建监控服务
date: 2015-10-10
categories: 
- 运维
tags: 
- Icinga2
- Icinga Web2
- 服务器监控
- CentOS

---

监控是系统运维的基础，Icinga是一个开源的监控系统框架，通过其广泛的插件支持，你可以监控几乎任何你能想到的网络资源；也可以在超过临界值时给予通知，根据监控数据生产报表。本文主要介绍其搭建过程，大部分内容翻译自官方文档。

<!--more-->
在使用Icinga2前，我们曾经自己编写过监控框架，也用过其它的类似监控系统和在线服务。功能都大同小异，现在之所以选择Icinga2 + Icinga Web2来构建整个系统的监控平台主要是由于其高度可自定义性以及广泛的插件。但是由于该版本发布时间比较短，国内少有将最新版本用于生产环境，导致文档较少，很多问题需要自行摸索。写这个系列文章也是想将自己遇到的坑列出来让其它人少走弯路。

下面将介绍Icinga2与Icinga Web2的安装，所有的操作步骤均仅在CentOS 7上验证（特殊说明的除外），不保证其它系统下的完备性。

##安装Icinga2##

###设置Icinga2安装源###

首先你要根据你的系统选择执行下面的命令将Icinga2的项目库地址加入到包管理中。

Debian (debmon):

{% codeblock %}
$ wget -O - http://debmon.org/debmon/repo.key 2>/dev/null | apt-key add -
$ echo 'deb http://debmon.org/debmon debmon-jessie main' >/etc/apt/sources.list.d/debmon.list
$ apt-get update
{% endcodeblock %}

Ubuntu (PPA):

{% codeblock %}
$ add-apt-repository ppa:formorer/icinga
$ apt-get update
{% endcodeblock %}

RHEL/CentOS:

{% codeblock %}
$ rpm --import http://packages.icinga.org/icinga.key
$ curl -o /etc/yum.repos.d/ICINGA-release.repo http://packages.icinga.org/epel/ICINGA-release.repo
$ yum makecache
{% endcodeblock %}
* 这个软件包依赖于[EPEL库](http://fedoraproject.org/wiki/EPEL "EPEL")中的其它软件包，请确保以及允许了[EPEL库](http://fedoraproject.org/wiki/EPEL "EPEL")。CentOS 7可以直接通过以下命令来安装：$ yum install epel-release

Fedora:

{% codeblock %}
$ rpm --import http://packages.icinga.org/icinga.key
$ curl -o /etc/yum.repos.d/ICINGA-release.repo http://packages.icinga.org/fedora/ICINGA-release.repo
$ yum makecache
{% endcodeblock %}

SLES 11:

{% codeblock %}
$ zypper ar http://packages.icinga.org/SUSE/ICINGA-release-11.repo
$ zypper ref
{% endcodeblock %}
* 这个软件包依赖于openssl1，它是SLES 11安全模块的一部分。

SLES 12:

{% codeblock %}
$ zypper ar http://packages.icinga.org/SUSE/ICINGA-release.repo
$ zypper ref
{% endcodeblock %}

openSUSE:

{% codeblock %}
$ zypper ar http://packages.icinga.org/openSUSE/ICINGA-release.repo
$ zypper ref
{% endcodeblock %}

###安装Icinga2###

Debian/Ubuntu:

{% codeblock %}
$ apt-get install icinga2
{% endcodeblock %}

RHEL/CentOS 5/6:

{% codeblock %}
$ yum install icinga2
$ chkconfig icinga2 on
$ service icinga2 start
{% endcodeblock %}

RHEL/CentOS 7 和 Fedora:

{% codeblock %}
$ yum install icinga2
$ systemctl enable icinga2
$ systemctl start icinga2
{% endcodeblock %}

SLES/openSUSE:

{% codeblock %}
$ zypper install icinga2
{% endcodeblock %}

###在安装过程中启用功能###

在默认安装中，启用了checker mainlog notification这三个功能，你可以通过执行以下命令来获取当前开启和关闭的所有功能：
{% codeblock %}
$ icinga2 feature list
{% endcodeblock %}

如果你希望开启或关闭某个功能，可以enable/disable命令，例如我们现在开启command功能：
{% codeblock %}
$ icinga2 feature enable command
{% endcodeblock %}
* 如果你启用了某个功能导致在接下来的启动过程中出现错误，记得回来关闭掉！

###文件默认位置###

|路径	|	描述|
|-------|--------------|
|/etc/icinga2	|包含Icinga2所有的配置文件|
|/etc/init.d/icinga2	|Icinga2的启动脚本|
|/usr/sbin/icinga2	|Icinga2程序目录|
|/usr/share/doc/icinga2	|Icinga2的文档目录|
|/usr/share/icinga2/include	|模板库和插件命令配置|
|/var/run/icinga2	|程序运行PID文件|
|/var/run/icinga2/cmd	|命令通道和状态套接字|
|/var/cache/icinga2	|缓存、调试、状态文件|
|/var/spool/icinga2	|用于性能数据脱机文件|
|/var/lib/icinga2	|状态文件，集群日志，本地证书和配置文件|
|/var/log/icinga2	|CompatLogger功能产生的日志文件以及相关工具目录|

##安装Check插件

Icinga2只是一个监控框架，如果没有插件，它将不知道如何检查外部服务。Icinga2是从nagios发展而来，因此它兼容nagios的插件。你可以直接使用包管理工具来安装插件包，下表将列出插件包得名称和默认安装路径。

|操作系统 |	包名称 |	安装路径|
|-------|---------|--------|
|RHEL/CentOS (EPEL)	|nagios-plugins-all	|/usr/lib/nagios/plugins 或 /usr/lib64/nagios/plugins|
|Debian	|nagios-plugins	|/usr/lib/nagios/plugins|
|FreeBSD	|nagios-plugins	|/usr/local/libexec/nagios|
|OS X (MacPorts)	|nagios-plugins	|/opt/local/libexec|

根据安装目录的差异，你可能需要修改Icinga2的配置来指定插件包得安装位置。配置文件为 /etc/icinga2/constants.conf，修改PluginDir的值为插件目录即可。

##运行Icinga2

###启动脚本

根据上面的文件位置表格，我们知道Icinga2的默认启动脚本位于： /etc/init.d/icinga2。

具体命令使用格式为 /etc/init.d/icinga2 {start|stop|restart|reload|checkconfig|status} ；命令具体含义与一般服务相同。

对于部分操作系统，例如：Fedora，openSUSE 和 RHEL/CentOS 7 可以使用服务来操作，Icinga2在安装时已经安装了必要的服务。

例如你可以使用以下命令来检查Icinga2的运行状态：

{% codeblock %}
$ systemctl status icinga2
{% endcodeblock %}

将Icinga2设置为自启动：

{% codeblock %}
$ systemctl enable icinga2
{% endcodeblock %}

以及重启Icinga2:

{% codeblock %}
$ systemctl restart icinga2
{% endcodeblock %}

如果启动出错，别忘了检查插件以及配置文件，你也可以使用 journalctl -xn 来查看错误信息。由于篇幅过长，Icinga Web2的安装与配置将放到下篇博客中！