title: 安装Icinga Web2所需服务
date: 2015-10-25
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

Icinga2通过DB IDO模块将所有配置与状态信息都保存在一个数据库中，这些数据会被Icinga Web2，Icinga Reporting或Icinga Web 1.x使用。

目前Icinga2支持使用MySQL和PostgreSQL作为后端数据库，本文仅介绍MySQL作为后端数据库的情况，如果你使用PostgreSQL，请依据官方文档配置好PostgreSQL。

### 安装MySQL数据库服务 ###

Debian/Ubuntu:

{% codeblock %}
$ apt-get install mysql-server mysql-client
{% endcodeblock %}

RHEL/CentOS 5/6:

{% codeblock %}
$ yum install mysql-server mysql
$ chkconfig mysqld on
$ service mysqld start
$ mysql_secure_installation
{% endcodeblock %}

RHEL/CentOS 7 和 Fedora:

{% codeblock %}
$ yum install mariadb-server mariadb
$ systemctl enable mariadb
$ systemctl start mariadb
$ mysql_secure_installation
{% endcodeblock %}

SUSE:

{% codeblock %}
$ zypper install mysql mysql-client
$ chkconfig mysqld on
$ service mysqld start
{% endcodeblock %}

FreeBSD:

{% codeblock %}
$ pkg install mysql56-server
$ sysrc mysql_enable=yes
$ service mysql-server restart
$ mysql_secure_installation
{% endcodeblock %}

### 安装MySQL的IDO模块 ###

通过默认的包管理器安装icinga2-ido-mysql包：

Debian/Ubuntu:

{% codeblock %}
$ apt-get install icinga2-ido-mysql
{% endcodeblock %}

RHEL/CentOS:

{% codeblock %}
$ yum install icinga2-ido-mysql
{% endcodeblock %}

SUSE:

{% codeblock %}
$ zypper install icinga2-ido-mysql
{% endcodeblock %}

FreeBSD:

FreeBSD的MySQL IDO模块已经包含在icinga2包中，位于 /usr/local/share/icinga2-ido-mysql/schema/mysql.sql。

注意：Debian/Ubuntu的包中提供了一个数据库配置向导，你可以根据个人喜好来选择是否使用它。

### 设置MySQL数据库 ###

为Icinga 2设置一个MySQL数据库:

{% codeblock %}
$ mysql -u root -p

mysql>  CREATE DATABASE icinga;
        GRANT SELECT, INSERT, UPDATE, DELETE, DROP, CREATE VIEW, INDEX, EXECUTE ON icinga.* TO 'icinga'@'localhost' IDENTIFIED BY 'icinga';
{% endcodeblock %}

在创建数据库完成后，使用以下命令导入Icinga2 IDO数据库结构：

{% codeblock %}
$ mysql -u root -p icinga < /usr/share/icinga2-ido-mysql/schema/mysql.sql
{% endcodeblock %}

为了使用Icinga Web2，我们为Icinga Web2也创建一个空的数据库：

{% codeblock %}
$ mysql -u root -p

mysql>  CREATE DATABASE icingaweb2;
        GRANT SELECT, INSERT, UPDATE, DELETE, DROP, CREATE VIEW, INDEX, EXECUTE ON icingaweb2.* TO 'icinga'@'localhost' IDENTIFIED BY 'icinga';
{% endcodeblock %}

启用MySQL IDO模块

{% codeblock %}
$ icinga2 feature enable ido-mysql
{% endcodeblock %}

启用后别忘了重启Icinga2服务：

Debian/Ubuntu, RHEL/CentOS 6 和 SUSE:

{% codeblock %}
$ service icinga2 restart
{% endcodeblock %}

RHEL/CentOS 7 和 Fedora:

{% codeblock %}
$ systemctl restart icinga2
{% endcodeblock %}

FreeBSD:

{% codeblock %}
$ service icinga2 restart
{% endcodeblock %}

### 安装Web服务 ###

Debian/Ubuntu:

{% codeblock %}
$ apt-get install apache2
{% endcodeblock %}

RHEL/CentOS 6:

{% codeblock %}
$ yum install httpd
$ chkconfig httpd on
$ service httpd start
{% endcodeblock %}

RHEL/CentOS 7/Fedora:

{% codeblock %}
$ yum install httpd
$ systemctl enable httpd
$ systemctl start httpd
{% endcodeblock %}

SUSE:

{% codeblock %}
$ zypper install apache2
$ chkconfig on
$ service apache2 start
{% endcodeblock %}

FreeBSD (nginx，你也可以使用 apache24 来作为Web服务)：

{% codeblock %}
$ pkg install nginx php56-gettext php56-ldap php56-openssl php56-mysql php56-pdo_mysql php56-pgsql php56-pdo_pgsql php56-sockets php56-gd pecl-imagick pecl-intl
$ sysrc php_fpm_enable=yes
$ sysrc nginx_enable=yes
$ sed -i '' "s/listen\ =\ 127.0.0.1:9000/listen\ =\ \/var\/run\/php5-fpm.sock/" /usr/local/etc/php-fpm.conf
$ sed -i '' "s/;listen.owner/listen.owner/" /usr/local/etc/php-fpm.conf
$ sed -i '' "s/;listen.group/listen.group/" /usr/local/etc/php-fpm.conf
$ sed -i '' "s/;listen.mode/listen.mode/" /usr/local/etc/php-fpm.conf
$ service php-fpm start
$ service nginx start
{% endcodeblock %}

### 配置防火墙规则 ###

例如：

{% codeblock %}
$ iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
$ service iptables save
{% endcodeblock %}

RHEL/CentOS 7 使用下面命令：

{% codeblock %}
$ firewall-cmd --add-service=http
$ firewall-cmd --permanent --add-service=http
{% endcodeblock %}

### 配置额外的命令通道 ###

Web接口和一些其它模块通过额外的命令通道来向Icinga2发送命令，你可以使用下面的命令启用它： 

{% codeblock %}
$ icinga2 feature enable command
{% endcodeblock %}

同样，你需要重启Icinga2服务来使它生效。

默认情况下，icingacmd用户组拥有读写命令通道文件的权限，因此你需要使用下面的命令将Web服务的用户加入到该组：

{% codeblock %}
$ usermod -a -G icingacmd www-data
{% endcodeblock %}

FreeBSD: www用户组拥有文件的读写权限，你不需要额外操作什么。

Debian包使用nagios作为默认用户名和用户组，因此你需要将上述命令中的icingacmd修改为nagios。

Web服务的用户名在不同的环境下是不同的，因此你可以尝试修改www-data为wwwrun、www、或者 apache。

你可以使用以下命令来检测是否成功将用户加入到icingacmd用户组：

{% codeblock %}
$ id <your-webserver-user>
{% endcodeblock %}

