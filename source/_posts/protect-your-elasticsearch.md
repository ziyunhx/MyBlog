title: 保护好你的Elasticsearch全文检索库
date: 2016-07-27
categories: 
- 运维
tags:
- SearchGuard
- Elasticsearch
- Snapshot

---

 Elasticsearch是一款基于Lucence的全文检索数据库，在文本分析、搜索等领域被广泛使用，但是默认的配置通过限制仅允许局域网访问来保证数据的安全性，如果你需要对公网提供服务，便需要额外安装插件来实现访问控制了。裸奔不仅可能带来被脱库，甚至数据会被删除而且不可恢复。本文将介绍如何选择和配置插件来保障Elasticsearch的安全。

<!--more-->
 本文介绍的环境均在 Centos 7 下，其它系统同理，只需要修改部分命令。

 ## 1. 安装 Elasticsearch，开启外网访问权限 ##

 - 安装JDK 7+，并配置 JAVA_HOME [Oracle 官网](http://docs.oracle.com/javase/8/docs/technotes/guides/install/install_overview.html);

 - 根据官网提供的方法安装 Elasticsearch：[使用yum安装](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-repositories.html);

 - 配置 /etc/elasticsearch/elasticsearch.yml 文件，此处修改 network.host 来允许外网访问：

        network.host:0.0.0.0
 
 - 为了防止他人的端口扫描，修改掉默认端口吧，例如设置成：

        http.port: 9999

 - 安装管理插件，例如 head ，重启 Elasticsearch。

 ## 2. 安装与配置 SearchGuard 2 ##

 - 安装 Elasticsearch 对应版本的 search-guard-ssl 和 search-guard-2 插件：

        cd /usr/share/elasticsearch
        sudo bin/plugin install -b com.floragunn/search-guard-ssl/2.3.4.14
        sudo bin/plugin install -b com.floragunn/search-guard-2/2.3.4.3

 - 配置 search-guard-ssl，生成证书：

        cd ~
        yum install wget unzip
        wget https://github.com/floragunncom/search-guard-ssl/archive/2.3.4.zip
        unzip 2.3.4.zip
        cd search-guard-ssl-2.3.4/example-pki-scripts/
        ./example.sh
 
 - 复制相应证书到配置文件目录，在集群中时修改 node-0-keystore.jks 中的数字保证与配置文件中相同：

        cp node-0-keystore.jks /etc/elasticsearch/
        cp truststore.jks /etc/elasticsearch/

 - 将目录中的 elasticsearch.yml.example 的内容复制到 /etc/elasticsearch/elasticsearch.yml

 - 在 /etc/elasticsearch/elasticsearch.yml 中增加以下内容：

        searchguard.ssl.transport.keystore_filepath: node-0-keystore.jks
        #注意 node-0-keystore.jks 与拷入目录的证书名称相同
        searchguard.ssl.transport.keystore_password: changeit
        searchguard.ssl.transport.truststore_filepath: truststore.jks
        searchguard.ssl.transport.truststore_password: changeit
        searchguard.ssl.transport.enforce_hostname_verification: false

 ## 3. 修改账户密码以及角色权限 ##

 - 使用自带的 hash 脚本生成密码（mycleartextpassword 替换为你需要设置的密码）：

        sh /usr/share/elasticsearch/plugins/search-guard-2/tools/hash.sh -p mycleartextpassword
 
 - 将生成的密码复制到 /usr/share/elasticsearch/plugins/search-guard-2/sgconfig/sg_internal_users.yml 中你需要的用户名的 password 后，并注释其它用户；

 - 在相同目录的 sg_roles.yml 中自定义你的用户角色权限；

 - 在 sg_roles_mapping.yml 中指定用户与角色间的对应关系。

 ## 4. 更新应用配置 SearchGuard 2 文件 ##

 - 拷贝证书到 SearchGuard 2 插件的配置文件目录：

        cd ~/search-guard-ssl-2.3.4/example-pki-scripts/
        cp truststore.jks /usr/share/elasticsearch/plugins/search-guard-2/sgconfig/
        cp kirk-keystore.jks /usr/share/elasticsearch/plugins/search-guard-2/sgconfig/

 - 重启 Elasticsearch 服务

 - 使用命令应用配置（即时不修改配置文件也需要执行）：

        sh plugins/search-guard-2/tools/sgadmin.sh -cd plugins/search-guard-2/sgconfig/ -ks plugins/search-guard-2/sgconfig/kirk-keystore.jks -ts plugins/search-guard-2/sgconfig/truststore.jks -nhnv -cn elasticsearch 
        # -cn 后为集群名称（默认为elasticsearch），如果你修改过 cluster.name，则将它替换成你修改的名称

 - 再次连接 Elasticsearch 服务，输入你设置的密码即可正常访问。

 ## 5. 给你的 Elasticsearch 服务创建快照备份 ##

 Elasticsearch 的数据安全不仅来自外部，内部成员的误操作同样可能带来不可挽回的损失；尽量将账户设置为最小所需权限是一个好方法，但是仍然不可避免存在超级用户，加上 head 插件缺失的确认机制，一不小心就回到解放前了！因此定时备份数据十分重要。

 - 设置共享目录，并挂载到集群上的所有机器，并赋予 Elasticsearch 用户写权限；

 - 在 elasticsearch.yml 中增加共享目录的配置，并重启 ES 服务：

        path.repo: ["/mount/backups"]

 - 使用 curl 创建备份仓库：

        $ curl -XPUT 'http://localhost:9200/_snapshot/my_backup' -d '{
            "type": "fs",
            "settings": {
                "location": "my_backup",
                "compress": true
            }
        }'

 - 对指定的 index 创建快照备份（注意索引间不要有空格）：

        $ curl -XPUT 'http://localhost:9200/_snapshot/my_backup/snapshot_1' -d '{
            "indices": "index_1,index_2"
        }'

 - 查询备份状态
 
        GET http://localhost:9200/_snapshot/my_backup/snapshot_1/_status

 - 把最后两部设置成定时任务吧，这样你就可以定期备份了；而且由于快照机制是增量备份，所以也不用担心空间了！

 好了，有关 Elasticsearch 的设置就到这里了。安全无小事，本文也仅是简单介绍了下方法和思路，许多细节都需要自己慢慢琢磨哦！