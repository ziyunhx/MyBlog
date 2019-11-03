title: 使用Docker快速部署Storm环境
date: 2015-11-18
categories: 
- Storm
tags:
- Docker
- csharp
- C#
- .NET
- storm
- mono

---

Storm的部署虽然不是特别麻烦，但是在生产环境中，为了提高部署效率，方便管理维护，使用Docker来统一管理部署是一个不错的选择。下面是我开源的一个新的项目，一个配置好了storm与mono环境的Docker镜像编排：[storm-mono-docker](https://github.com/ziyunhx/storm-mono-docker)。

<!--more-->

这个项目得益于[https://github.com/ptgoetz/storm-vagrant](https://github.com/ptgoetz/storm-vagrant)和[https://github.com/wurstmeister/storm-docker](https://github.com/wurstmeister/storm-docker)；在此感谢他们的付出！
项目使用的Docker镜像托管在 [https://index.docker.io](https://index.docker.io)。

## 准备工作 ##

- 安装 Docker Engine，[https://docs.docker.com/](https://docs.docker.com/)

- 安装 docker-compose [http://docs.docker.com/compose/install/](http://docs.docker.com/compose/install/)

- 克隆git项目： 

		git clone https://github.com/ziyunhx/storm-mono-docker

## 使用 ##

首先将命令行目录切换到刚刚克隆下来的git项目目录；

通过以下命令启动集群：

	docker-compose up -d

- 你也可以使用 docker-compose up 命令来将结果输出到当前命令行界面，但是在你结束它之前无法进行任何其它操作，而一旦命令行退出，所有的容器都将停止。而 docker-compose up -d 将在后台启动所有容器。

停止这个集群的所有容器：

	docker-compose stop

容器一旦停止，下次直接启动将无法正常链接容器，导致storm运行异常，你可以在结束后使用以下命令结束和移除所有的Docker缓存：

	docker kill $(docker ps -q) ; docker rm $(docker ps -a -q)

增加更多的supervisors：

	docker-compose scale supervisor=4


使用以下命令删除所有的镜像文件（小心，这会让你下一次启动时花费更多时间下载容器，仅在不想继续使用时执行）：

	docker rmi $(docker images -q -a)

## 重新构建和更新 ##

你可以在修改Dockerfile后使用以下命令来重新构建镜像：rebuild.sh ；

使用以下命令来更新镜像到最新版本：refresh.sh 。

## 问与答 ##

### 如何访问Storm UI来查看运行状况？ ###
在docker-compose.yml中有下面这段配置：

    ui:
      image: ziyunhx/storm-ui
	      ports:
	        - "49080:8080"

它告诉我们将Docker镜像的8080端口映射到了主机的49080，因此你可以通过访问 http://localhost:49080 来访问。如果你使用 boot2docker ，你可以通过以下命令得到虚拟机的IP：

    $ boot2docker ip
    The VM's Host only interface IP address is: 192.168.59.103

返回的结果就是你的IP，本例中可以通过 http://192.168.59.103:49080 来访问。

### 如何部署提交一个topology？ ###

如果 nimbus 的IP与端口不是默认的，你需要指定它们后来提交，本例中可以使用以下命令：

    storm jar target/your-topology-fat-jar.jar com.your.package.AndTopology topology-name -c nimbus.host=192.168.59.103 -c nimbus.thrift.port=49627

如果上述命令没有起作用，你可以在本地的Storm配置文件（storm.yaml）配置以下项：

	nimbus.host: "192.168.59.103"	
	nimbus.thrift.port: 49627

然后执行以下命令提交：

	storm jar target/your-topology-fat-jar.jar com.your.package.AndTopology topology-name

### 如何连接我的容器? ###
通过使用 docker-compose ps 找到你希望连接的容器的ssh端口，然后通过ssh连接：

    $ ssh root@`boot2docker ip` -p $CONTAINER_PORT

密码是 'ziyunhxpass' （位于：https://registry.hub.docker.com/u/ziyunhx/base/dockerfile/）。
