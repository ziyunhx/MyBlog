title: 使用Icinga2监控服务器带宽以及内存占用
date: 2016-1-11
categories: 
- 运维
tags: 
- Icinga2
- Icinga Web2
- 服务器监控
- CentOS
- Memory
- Network

---

监控是系统运维的基础，Icinga是一个开源的监控系统框架，通过其广泛的插件支持，你可以监控几乎任何你能想到的网络资源；也可以在超过临界值时给予通知，根据监控数据生产报表。本文将介绍如何通过SNMP协议来监控服务器的带宽以及内存占用。

<!--more-->

SNMP（简单网络管理协议），由一组网络管理的标准组成，包含一个应用层协议、数据库模型和一组资源对象。该协议能够支持网络管理系统，用以监测连接到网络上的设备是否有任何引起管理上关注的情况。SNMP的目标是管理互联网 Internet 上众多厂家生产的软硬件平台，因此SNMP受Internet标准网络管理框架的影响也很大。

下表是 Icinga2 通过 SNMP 在各类系统上支持的项，本文仅介绍 Centos7 和 Windows 系统下的网络和内存的监控设置。

| 主机类型  | 接口    | 存储   | CPU负载  |内存    | 进程   | 环境变量 | 备注   |
| ---------| ------ |-------| ---------| ----- | ----- |---------| ------|
| Linux   | YES     | YES   | YES      |YES 	| YES  | NO     |       |
| Windows	| YES     | YES   | YES      |YES 	| YES  | NO     | check_snmp_win.pl      |
| Cisco router/switch	| YES     | N/A   | YES      |YES 	| N/A  | YES     |       |
| HP router/switch		| YES     | N/A   | YES      |YES 	| N/A  | NO     |       |
| Bluecoat proxy		| YES     | SNMP   | YES      |SNMP 	| NO  | YES     |       |
| CheckPoint on SPLAT	| YES     | YES   | YES      |YES 	| YES  | NO     | check_snmp_cpfw.pl      |
| CheckPoint on Nokia IP| YES     | YES   | YES      |NO 	| --  | NO     | check_snmp_vrrp.pl      |
| Boostedge   		| YES     | YES   | YES      |YES 	| --  | NO     |  check_snmp_boostedge.pl     |
| AS400		| YES     | YES   | YES      |YES 	| NO  | NO     |       |
| NetsecureOne Netbox | YES     | YES   | YES      |-- 	| YES  | NO     |       |
| Radware Linkproof	| YES     | N/A   | SNMP      |SNMP 	| NO  | NO     |  check_snmp_linkproof_nhr, check_snmp_vrrp.pl     |
| IronPort 		| YES     | SNMP   | SNMP      |SNMP 	| NO  | YES     |       |
| Cisco CSS   	| YES     | --   | YES      |YES 	| NO  | --     | check_snmp_css.pl      |

 - N/A, NO: 不支持或不存在
 - SNMP: 仅支持简单的SNMP查询
 - --: 未测试

# 安装与配置SNMP #

#### 安装 NET-SNMP ####

CentOS可以直接通过yum安装 NET-SNMP 的包，其它系统也可以通过 [NET-SNMP项目](http://sourceforge.net/projects/net-snmp/) 下载安装。

{% codeblock %}
# yum install net-snmp net-snmp-devel net-snmp-utils
{% endcodeblock %}

#### 配置 NET-SNMP ####

CentOS需要关闭selinux才能正常使用snmp v3，使用以下命令查看SELinux状态：

{% codeblock %}
# /usr/sbin/sestatus -v
{% endcodeblock %}

如果SELinux status参数为enabled即为开启状态。

关闭SELinux：

1: 临时关闭（不用重启机器），但机器重启后会失效： `# setenforce 0`
 
2: 永久关闭，需要修改配置文件需要重启机器：
修改/etc/selinux/config 文件，将SELINUX=enforcing改为SELINUX=disabled，重启系统。

停用 NET-SNMP 服务，并配置用户：

{% codeblock %}
# service snmpd stop
# net-snmp-config --create-snmpv3-user -ro -A tnidea@carey -a MD5 carey
{% endcodeblock %}

以上命令，创建了一个用户名为carey，密码为tnidea@carey的snmpv3用户，使用MD5加密。

#### 运行 NET-SNMP ####

运行 NET-SNMP 服务，并加入到开机启动项。

{% codeblock %}
# service snmpd start
# chkconfig snmpd on
{% endcodeblock %}

使用 snmpwalk 命令检测snmp服务是否正常启动，正常将返回相关结果：

{% codeblock %}
# snmpwalk -v 3 -u carey -a MD5 -A "tnidea@carey" -l authNoPriv 127.0.0.1 sysDescr
{% endcodeblock %}

Windows系统包中已经自带了 SNMP 功能，直接在 控制面板->添加或删除程序 中找到 简单网络管理协议(SNMP) 并勾选，安装完成后到服务中配置相关安全选项。

另外你还需要在防火墙上允许 SNMP 端口进行通讯，其默认端口为 161 。

# 安装SNMP模块监控网络与内存 #

首先安装 perl 环境，下面是以 Centos 为例：

{% codeblock %}
# yum install perl*
# yum install cpan
{% endcodeblock %}

然后通过 CPAN 来安装 Net-SNMP：

{% codeblock %}
# perl -MCPAN -e shell
cpan shell -- CPAN exploration and modules installation (v1.76)
ReadLine support enabled
cpan> install Net::SNMP
{% endcodeblock %}

CPAN 的安装目录可能在ROOT目录下，这会导致 Icinga2 无法调用，你需要根据安装结果提示将安装的文件复制到支持的目录，比如：`/usr/local/lib64/perl5/`。

另外还需要将 utils.pm 链接到 perl 的运行目录：

{% codeblock %}
# ln -s /usr/lib64/nagios/plugins/utils.pm /usr/local/lib64/perl5/
{% endcodeblock %}

另外你还需要通过修改配置文件 icinga2.conf 来启用 SNMP 插件检查命令，在该文件中增加 `include <manubulon>`。

#### 通过 SNMP 检查内存 ####

该功能需要 [check_snmp_mem.pl](http://nagios.manubulon.com/snmp_mem.html) 命令，如果不存在需要下载后放到 manubulon 定义的目录。下面是支持的一些参数：

|名称	|描述|
| ---------| ------ |
|snmp_address	|可选，主机地址，默认为变量 "$address$" 所设置的值。|
|snmp_nocrypt	|可选，定义 SNMP 加密方式，仅在使用 snmp_v3 时需要设置，默认为 false。|
|snmp_community	|可选，SNMP 接受团体，默认为 "public"。|
|snmp_port	|可选，SNMP 连接端口。|
|snmp_v2	|可选，SNMP 版本为 2c？默认为 false。|
|snmp_v3	|可选，SNMP 版本为 3？默认为 false。|
|snmp_login	|可选，snmp_v3 时的用户名，默认为 "snmpuser"。|
|snmp_password	|必须，snmp_v3 时的密码，无默认值。|
|snmp_v3_use_privpass	|可选，定义 snmp_v3 使用的私钥，默认为 false。|
|snmp_authprotocol	|可选，snmp_v3的验证接口，默认为 "md5,des"。|
|snmp_privpass	|必须，snmp_v3 时的私钥，仅在使用私钥时需要，无默认值。|
|snmp_warn	|可选，提示警告时的取值范围。|
|snmp_crit	|可选，提示错误时的取值返回。|
|snmp_is_cisco	|可选，修改 Cisco switches 的 OIDs，默认为 false。|
|snmp_perf	|可选，允许生成 perfdata 的值（用于图表），默认为 true。|
|snmp_timeout	|可选，命令超时时间，默认为 5秒。|

下面是一个检查内存的例子，放在 services.conf：

{% codeblock %}
apply Service "memory" {
  import "generic-service"

  check_command = "snmp-memory"
  vars.snmp_nocrypt = "authNoPriv"
  vars.snmp_v3 = true
  vars.snmp_login = "carey"
  vars.snmp_password = "tnidea@carey"
  vars.snmp_authprotocol = "MD5"
  assign where host.name == NodeName
}
{% endcodeblock %}

#### 通过 SNMP 检查网络 ####

该功能需要 [check_snmp_int.pl](http://nagios.manubulon.com/snmp_int.html) 命令，如果不存在需要下载后放到 manubulon 定义的目录。下面是支持的一些参数：

|名称	|描述|
| ---------| ------ |
|snmp_address	|可选，主机地址，默认为变量 "$address$" 所设置的值。|
|snmp_nocrypt	|可选，定义 SNMP 加密方式，仅在使用 snmp_v3 时需要设置，默认为 false。|
|snmp_community	|可选，SNMP 接受团体，默认为 "public"。|
|snmp_port	|可选，SNMP 连接端口。|
|snmp_v2	|可选，SNMP 版本为 2c？默认为 false。|
|snmp_v3	|可选，SNMP 版本为 3？默认为 false。|
|snmp_login	|可选，snmp_v3 时的用户名，默认为 "snmpuser"。|
|snmp_password	|必须，snmp_v3 时的密码，无默认值。|
|snmp_v3_use_privpass	|可选，定义 snmp_v3 使用的私钥，默认为 false。|
|snmp_authprotocol	|可选，snmp_v3的验证接口，默认为 "md5,des"。|
|snmp_privpass	|必须，snmp_v3 时的私钥，仅在使用私钥时需要，无默认值。|
|snmp_warn	|可选，提示警告时的取值范围。|
|snmp_crit	|可选，提示错误时的取值返回。|
|snmp_interface	|可选，网卡名称，默认为 "eth0"。注意该项是模糊匹配。|
|snmp_interface_perf	|可选，检查输入输出端口的带宽，默认为 true。|
|snmp_interface_label	|可选，在输出前自定义字段，例如 in=, out=, errors-out=, 等等...|
|snmp_interface_bits_bytes	|可选，输出数据格式使用 bits/s 或者 Bytes/s。需要 snmp_interface_kbits 设置为 true。默认为 true。|
|snmp_interface_percent	|可选， 输出数据使用相对于最大值的百分比表述，默认为 false。|
|snmp_interface_kbits	|可选，使用 KBits/s 来定义警告与错误的等级，默认为 true。|
|snmp_interface_megabytes	|可选，使用 Mbps 或 MBps 来定义警告与错误的等级，需要 snmp_interface_kbits 设置为 true。默认为 true。|
|snmp_interface_64bit	|可选，使用64位的计数器取代默认的计数器，当带宽大于等于 1Gb 时使用，默认为 false。|
|snmp_interface_errors	|可选，在输出时输出错误信息，默认为 true。|
|snmp_interface_noregexp	|可选，不使用正则匹配网卡名称，默认为 false。|
|snmp_interface_delta	|可选，检查间隔时间，默认为 "300" (5 分钟)。|
|snmp_warncrit_percent	|可选，将警告与错误的等级通过报告速度的百分比来定义，如果设置了 snmp_interface_megabytes 则需要设置为 false。 默认为 false。|
|snmp_perf	|可选，允许生成 perfdata 的值（用于图表），默认为 true。|
|snmp_timeout	|可选，命令超时时间，默认为 5秒。|

下面是一个检查网络的例子，同样放在 services.conf：

{% codeblock %}
apply Service "network" {
  import "generic-service"

  check_command = "snmp-interface"
  vars.snmp_nocrypt = "authNoPriv"
  vars.snmp_v3 = true
  vars.snmp_login = "carey"
  vars.snmp_password = "tnidea@carey"
  vars.snmp_interface = "eth0"
  vars.snmp_authprotocol = "MD5"
  vars.snmp_interface_megabytes = false
  assign where host.name == NodeName
}
{% endcodeblock %}