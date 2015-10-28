title: 安装与配置Icinga Web2
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

## 配置 DB IDO MySQL ##

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

Enabling the IDO MySQL module

The package provides a new configuration file that is installed in /etc/icinga2/features-available/ido-mysql.conf. You will need to update the database credentials in this file.

All available attributes are explained in the IdoMysqlConnection object chapter.

You can enable the ido-mysql feature configuration file using icinga2 feature enable:

$ icinga2 feature enable ido-mysql
{% endcodeblock %}

Module 'ido-mysql' was enabled.
Make sure to restart Icinga 2 for these changes to take effect.
After enabling the ido-mysql feature you have to restart Icinga 2:

Debian/Ubuntu, RHEL/CentOS 6 and SUSE:

$ service icinga2 restart
{% endcodeblock %}

RHEL/CentOS 7 and Fedora:

$ systemctl restart icinga2
{% endcodeblock %}

FreeBSD:

$ service icinga2 restart
{% endcodeblock %}

Webserver

Debian/Ubuntu:

$ apt-get install apache2
{% endcodeblock %}

RHEL/CentOS 6:

$ yum install httpd
$ chkconfig httpd on
$ service httpd start
{% endcodeblock %}

RHEL/CentOS 7/Fedora:

$ yum install httpd
$ systemctl enable httpd
$ systemctl start httpd
{% endcodeblock %}

SUSE:

$ zypper install apache2
$ chkconfig on
$ service apache2 start
{% endcodeblock %}

FreeBSD (nginx, but you could also use the apache24 package):

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

Firewall Rules

Example:

$ iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
$ service iptables save
{% endcodeblock %}

RHEL/CentOS 7 specific:

$ firewall-cmd --add-service=http
$ firewall-cmd --permanent --add-service=http
{% endcodeblock %}

FreeBSD: Please consult the FreeBSD Handbook how to configure one of FreeBSD's firewalls.

Setting Up External Command Pipe

Web interfaces and other Icinga addons are able to send commands to Icinga 2 through the external command pipe.

You can enable the External Command Pipe using the CLI:

$ icinga2 feature enable command
{% endcodeblock %}

After that you will have to restart Icinga 2:

Debian/Ubuntu, RHEL/CentOS 6 and SUSE:

$ service icinga2 restart
{% endcodeblock %}

RHEL/CentOS 7 and Fedora:

$ systemctl restart icinga2
{% endcodeblock %}

FreeBSD:

$ service icinga2 restart
{% endcodeblock %}

By default the command pipe file is owned by the group icingacmd with read/write permissions. Add your webserver's user to the group icingacmd to enable sending commands to Icinga 2 through your web interface:

$ usermod -a -G icingacmd www-data
{% endcodeblock %}

FreeBSD: On FreeBSD the rw directory is owned by the group www. You do not need to add the user icinga to the group www.

Debian packages use nagios as the default user and group name. Therefore change icingacmd to nagios.

The webserver's user is different between distributions so you might have to change www-data to wwwrun, www, or apache.

Change "www-data" to the user you're using to run queries.

You can verify that the user has been successfully added to the icingacmd group using the id command:

$ id <your-webserver-user>
{% endcodeblock %}

Installing up Icinga Web 2

Please consult the installation documentation for further instructions on how to install Icinga Web 2.

Addons

A number of additional features are available in the form of addons. A list of popular addons is available in the Addons and Plugins chapter.