title: 安装与配置Icinga Web2
date: 2015-10-30
categories: 
- 运维
tags: 
- Icinga2
- Icinga Web2
- 服务器监控
- CentOS

---

Icinga Web2是Icinga2的Web前端展示界面之一，它也是支持Icinga2的Web界面中我个人最喜欢的。通过Icinga Web2，你可以方便的展示与管理你的监控，也可以自定义一些显示模块。本文将介绍Icinga Web2的安装与配置，其中部分经验可能在Icinga2的其它Web界面也适用。

<!--more-->

### 设置包管理库 ###

你需要使用以下命令将Icinga2的所在的包管理库加入到你的系统中，如果你按照我之前的文章操作的话，其实这步已经做过了。

Debian (debmon):

{% codeblock %}
wget -O - http://debmon.org/debmon/repo.key 2>/dev/null | apt-key add -
echo 'deb http://debmon.org/debmon debmon-wheezy main' >/etc/apt/sources.list.d/debmon.list
apt-get update
{% endcodeblock %}

Ubuntu Trusty:

{% codeblock %}
wget -O - http://packages.icinga.org/icinga.key | apt-key add -
add-apt-repository 'deb http://packages.icinga.org/ubuntu icinga-trusty main'
apt-get update
{% endcodeblock %}

如果你使用的是其它的Ubuntu版本，只需要将trusty替换成你实际环境的名字。

RHEL 和 CentOS:

{% codeblock %}
rpm --import http://packages.icinga.org/icinga.key
curl -o /etc/yum.repos.d/ICINGA-release.repo http://packages.icinga.org/epel/ICINGA-release.repo
yum makecache
{% endcodeblock %}

Fedora:

{% codeblock %}
rpm --import http://packages.icinga.org/icinga.key
curl -o /etc/yum.repos.d/ICINGA-release.repo http://packages.icinga.org/fedora/ICINGA-release.repo
yum makecache
{% endcodeblock %}

SLES 11:

{% codeblock %}
zypper ar http://packages.icinga.org/SUSE/ICINGA-release-11.repo
zypper ref
{% endcodeblock %}

SLES 12:

{% codeblock %}
zypper ar http://packages.icinga.org/SUSE/ICINGA-release.repo
zypper ref
{% endcodeblock %}

openSUSE:

{% codeblock %}
zypper ar http://packages.icinga.org/openSUSE/ICINGA-release.repo
zypper ref
{% endcodeblock %}

RHEL/CentOS的软件包依赖于EPEL，确保你已经配置了它！这步应该在安装Icinga2时已经完成。
Debian wheezy的软件包依赖于wheezy-packports库，请在安装前启用它。

### 安装 Icinga Web2 ###

你可以通过系统的包管理器来安装Icinga Web2，下面是一些系统的具体命令：

Debian 和 Ubuntu:

{% codeblock %}
apt-get install icingaweb2
{% endcodeblock %}

RHEL, CentOS 和 Fedora:

{% codeblock %}
yum install icingaweb2 icingacli
{% endcodeblock %}

Debian wheezy/RHEL/CentOS用户请仔细阅读包管理库的提示。

SLES 和 openSUSE:

{% codeblock %}
zypper install icingaweb2 icingacli
{% endcodeblock %}

### 配置Icinga Web2 ###

你可以通过Icinga Web2配置向导，或者直接通过执行命令来完成配置，本文仅介绍通过配置向导来完成配置，如果需要使用命令行来完成，请查看 Icinga Web 2官方文档。

STEP 1: 通过icingacli创建一个token，用来安装Icinga Web2：

{% codeblock %}
icingacli setup token create
icingacli setup token show
{% endcodeblock %}

STEP 2: 使用浏览器访问 http://127.0.0.1/icingaweb2/setup （将IP替换为实际IP或域名），将上一步命令行得到的token输入到Steup Token的输入框中，继续下一步。

 ![Image](https://www.tnidea.com/media/image/icinga-websetup-1.png)

STEP 3: 根据需要选择需要安装的模块，我这边选择了除翻译外的所有模块。

 ![Image](https://www.tnidea.com/media/image/icinga-websetup-2.png)

STEP 4: 根据系统检查结果，解决需要修改的项，全部完成后刷新确认。

 ![Image](https://www.tnidea.com/media/image/icinga-websetup-3.png)

本例中，需要解决的有PHP时区，LDAP，PDO-MySQL，PDO-PostgreSQL问题。PDO-MySQL，PDO-PostgreSQL只需要重启Web服务器即可解决。

修改PHP时区：
````使用 vi 打开 /etc/php.ini；查找 date.timezone，删除最前面的分号，在结尾增加时区标签，这里使用： Asia/Shanghai 。````

启用LDAP：
{% codeblock %}
yum install php-ldap
{% endcodeblock %}

重启Web服务器，CentOS 7直接使用 systemctl restart httpd 重启。
现在界面上所有的状态应该都已经变成了绿色，继续下一步吧，后面没有特别讲到的步骤都是直接点击下一步的！

STEP 5: 配置Icinga Web2数据库

 ![Image](https://www.tnidea.com/media/image/icinga-websetup-4.png)

此处直接填写我们创建的icingaweb2数据库的信息，默认情况下用户名和密码都是icingaweb2。由于我们并没有给予icingaweb2用户创建表的权限，因此你还需要给一个有创建数据库和表权限的用户。

![Image](https://www.tnidea.com/media/image/icinga-websetup-5.png)

STEP 6: 创建一个Icinga Web2的管理员账号

![Image](https://www.tnidea.com/media/image/icinga-websetup-6.png)

STEP 7: 配置Icinga IDO数据库信息

![Image](https://www.tnidea.com/media/image/icinga-websetup-7.png)

此处直接填写我们创建的icinga数据库的信息，默认情况下用户名和密码都是icinga。

ok! 一切就绪，登陆进去瞅瞅吧！

![Image](https://www.tnidea.com/media/image/icinga-websetup-8.png)