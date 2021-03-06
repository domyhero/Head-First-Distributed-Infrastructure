


[TOC]


# Introduction


## Reference


### Tutorials & Docs


# Quick Start
## Installation
### Linux

从[这里](http://zookeeper.apache.org/releases.html)下载，下载解压之后可以直接以独立模式运行。Server被包含在了一个的Jar包中。
 tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181

This file can be called anything, but for the sake of this discussion call it conf/zoo.cfg. Change the value of dataDir to specify an existing (empty to start with) directory. Here are the meanings for each of the fields:tickTime
the basic time unit in milliseconds used by ZooKeeper. It is used to do heartbeats and the minimum session timeout will be twice the tickTime.dataDir
the location to store the in-memory database snapshots and, unless specified otherwise, the transaction log of updates to the database.clientPort
the port to listen for client connections

Now that you created the configuration file, you can start ZooKeeper:bin/zkServer.sh start



### Docker


``` shell
FROM ubuntu:vivid


RUN apt-get update \
 && apt-get -y install git ant openjdk-8-jdk \
 && apt-get clean
RUN mkdir /tmp/zookeeper
WORKDIR /tmp/zookeeper
RUN git clone https://github.com/apache/zookeeper.git .
RUN git checkout release-3.5.1-rc2
RUN ant jar
# 将zookeeper配置文件复制到指定地方
RUN cp /tmp/zookeeper/conf/zoo_sample.cfg /tmp/zookeeper/conf/zoo.cfg
RUN echo "standaloneEnabled=false" >> /tmp/zookeeper/conf/zoo.cfg
RUN	echo "dynamicConfigFile=/tmp/zookeeper/conf/zoo.cfg.dynamic" >> /tmp/zookeeper/conf/zoo.cfg
ADD zk-init.sh /usr/local/bin/
ENTRYPOINT ["/usr/local/bin/zk-init.sh"]
```


使用以下命令构建单机版的Docker镜像


``` 
docker build -t `whoami`/zookeeper:latest .
```


zk-init.sh脚本内容如下：


``` shell
#!/bin/bash


ZK=$1
MYID=1
IPADDRESS=`ip -4 addr show scope global dev eth0 | grep inet | awk '{print \$2}' | cut -d / -f 1`


cd /tmp/zookeeper
if [ -n "$ZK" ] 
then
  output=`./bin/zkCli.sh -server $ZK:2181 get /zookeeper/config | grep ^server` 
  
  # extract all the zk-ids from the output
  declare -a id_list=()
  while read x;
  do
    id_list+=(`echo $x | cut -d"=" -f1 | cut -d"." -f2`)
  done <<<$(output)
  sorted_id_list=( $(
    for el in "${id_list[@]}"
    do  
      echo "$el"
    done | sort -n) )
  
  # get the next increasing number from the sequence
  MYID=$((${sorted_id_list[${#sorted_id_list[@]}-1]}+1))
  echo $output >> /tmp/zookeeper/conf/zoo.cfg.dynamic
  echo "server.$MYID=$IPADDRESS:2888:3888:observer;2181" >> /tmp/zookeeper/conf/zoo.cfg.dynamic
  cp /tmp/zookeeper/conf/zoo.cfg.dynamic /tmp/zookeeper/conf/zoo.cfg.dynamic.org
  /tmp/zookeeper/bin/zkServer-initialize.sh --force --myid=$MYID
  ZOO_LOG_DIR=/var/log ZOO_LOG4J_PROP='INFO,CONSOLE,ROLLINGFILE' /tmp/zookeeper/bin/zkServer.sh start
  /tmp/zookeeper/bin/zkCli.sh -server $ZK:2181 reconfig -add "server.$MYID=$IPADDRESS:2888:3888:participant;2181"
  /tmp/zookeeper/bin/zkServer.sh stop
  ZOO_LOG_DIR=/var/log ZOO_LOG4J_PROP='INFO,CONSOLE,ROLLINGFILE' /tmp/zookeeper/bin/zkServer.sh start-foreground
else
  echo "server.$MYID=$IPADDRESS:2888:3888;2181" >> /tmp/zookeeper/conf/zoo.cfg.dynamic
  /tmp/zookeeper/bin/zkServer-initialize.sh --force --myid=$MYID
  ZOO_LOG_DIR=/var/log ZOO_LOG4J_PROP='INFO,CONSOLE,ROLLINGFILE' /tmp/zookeeper/bin/zkServer.sh start-foreground
fi


```


``` 
docker run --net=host --name zk1 `whoami`/zookeeper
```


注意，这里使用了`—net`指令，即是直接将Zookeeper的端口映射到了宿主机上，如果要在一台宿主机上Run多个Zookeeper实例，请使用container网络，建议使用Docker-Compose进行构建。


We need the ip of our node. This will be the ip of your host, or in a single host setup you will need to inspect the container like this:


``` 
docker inspect zk1|grep IPAddress
```


We specify the IP address when starting the second node:


``` 
docker run --net=host --name zk2 containersol/zookeeper <ip of the first zookeeper>
```


This time we also see a few WARNINGS and even an ERROR, but in the end 


we have two nodes running. We throw in a third for good measure:


``` 
docker run --net=host --name zk3 containersol/zookeeper <ip of 1st zookeeper>
```


集群建立之后，可以通过如下命令验证：


``` shell
docker exec -it zk1 bin/zkCli.sh -server <ip of 1st zookeeper>:2181 config| grep ^server
server.1=<ip of 1st zookeeper>:2888:3888:participant;0.0.0.0:2181
server.2=<ip of 2nd zookeeper>:2888:3888:participant;0.0.0.0:2181
server.3=<ip of 3rd zookeeper>:2888:3888:participant;0.0.0.0:2181
```