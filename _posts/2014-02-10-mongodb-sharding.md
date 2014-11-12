---
layout: post
title:  搭建mongodb sharding 集群
category: [分布式,nosql]
---
[参考：MongoDB-sharding-guide](http://docs.mongodb.org/master/MongoDB-sharding-guide.pdf)
[参考：集群认证方式](http://docs.mongodb.org/manual/tutorial/enable-authentication-in-sharded-cluster/)


### 集群架构图

![Alt text](/images/mongo.png)

***Shards*** store the data. To provide high availability and data consistency, in a production sharded cluster, each shard is
a replica set . For more information on replica sets, see Replica Sets.

***Query Routers***, or mongos instances, interface with client applications and direct operations to the appropriate shard
or shards. The query router processes and targets operations to shards and then returns results to the clients. A sharded
cluster can contain more than one query router to divide the client request load. A client sends requests to one query
router. Most sharded clusters have many query routers.

***Config servers*** store the cluster’s metadata. This data contains a mapping of the cluster’s data set to the shards. The
query router uses this metadata to target operations to specific shards. Production sharded clusters have exactly 3
config servers.

### 集群搭建示例

#### 资源分配

四台服务器搭建4个shard,如下列表列出每个服务器节点需要安装的服务，每个节点都是数据节点,1、2、3上面安装配置节点存储metadata 避免单点，
1、4安装路由节点，负责负载均衡请求。

>服务器IP

1,10.20.0.14 | 2,10.20.0.15 | 3,10.20.0.16 | 4,10.20.0.17

>shard的IP分配

shard_a(1,2,3) | shard_b(2,3,4) | shard_c(3,4,1) | shard_d(4,1,2)

>Config servers的IP分配

config_server(1) | config_server(2) | config_server(3)

>mongos 的IP分配

mongos(1)	|	mongos(3)

>端口分配

shard_a:27018 | shard_b:27019 | shard_c:27020 | shard_d:27021| config_server:27022 | mongos :27017

#### 配置示例

***1、每个节点安装服务配置文件目录结构***

安装过程略，解压即可，如过配置文件中指定的文件夹不存在，mkdir创建即可 如 mkdir shard_a ，比较简单，安装服务的配置文件目录结构。

1-10.20.0.14节点是shard_a/c/d的一个节点，同时还要安装congfig server 和 mongos
{% highlight java %}
/home/work/local/mongodb-2.6.4/etc  
├── config_server.conf  
├── mongos.conf  
├── shard_a.conf  
├── shard_c_arbiter.conf  
└── shard_d.conf 
{% endhighlight %} 

2-10.20.0.15节点是shard_a/b/d的一个节点，同时还要安装congfig server
{% highlight java %}
/home/work/local/mongodb-2.6.4/etc
├── config_server.conf
├── shard_a.conf
├── shard_b.conf
└── shard_d_arbiter.conf
{% endhighlight %} 

3-10.20.0.16节点是shard_a/b/c的一个节点，同时还要安装congfig server 和 mongos
{% highlight java %}
/home/work/local/mongodb-2.6.4/etc  
├── config_server.conf
├── mongos.conf
├── shard_a_arbiter.conf
├── shard_b.conf
└── shard_c.conf
{% endhighlight %} 

4-10.20.0.17节点是shard_b/c/d的一个节点
{% highlight java %}
/home/work/local/mongodb-2.6.4/etc  
├── shard_b_arbiter.conf
├── shard_c.conf
└── shard_d.conf
{% endhighlight %} 



#### 2、所有文件的配置示例
如果某个节点是某个shard的投票节点（非数据节点）方便日后维护管理，可以给把投票节点通过命名区分出来，如shard_a_arbiter.conf

shard_a.conf内容示例
{% highlight java %}
#numactl --interleave=all /home/work/local/mongodb-2.6.4/bin/mongod -f etc/shard_a.conf 指定CPU内存分配方式，针对NUMA是多核心CPU架构中的一种优化。
port=27018
fork=true
dbpath=/home/work/local/mongodb-2.6.4/data/shard_a/
logpath=/home/work/local/mongodb-2.6.4/logs/shard_a.log
logappend=true
#制定集群间简单认证方式 shard_key中的内容所有节点需要一致，这一步不是不需的
keyFile=/home/work/local/mongodb-2.6.4/data/key/shard_key
shardsvr=true
replSet=shard_a
{% endhighlight %} 

shard_b.conf内容示例
{% highlight java %}
#numactl --interleave=all /home/work/local/mongodb-2.6.4/bin/mongod -f etc/shard_b.conf
port=27019
fork=true
dbpath=/home/work/local/mongodb-2.6.4/data/shard_b/
logpath=/home/work/local/mongodb-2.6.4/logs/shard_b.log
logappend=true
keyFile=/home/work/local/mongodb-2.6.4/data/key/shard_key
shardsvr=true
replSet=shard_b
{% endhighlight %} 

##### shard_c.conf内容示例
{% highlight java %}
#numactl --interleave=all /home/work/local/mongodb-2.6.4/bin/mongod -f etc/shard_c_arbiter.conf
port=27020
fork=true
dbpath=/home/work/local/mongodb-2.6.4/data/shard_c/
logpath=/home/work/local/mongodb-2.6.4/logs/shard_c.log
logappend=true
keyFile=/home/work/local/mongodb-2.6.4/data/key/shard_key
shardsvr=true
replSet=shard_c
{% endhighlight %} 

##### shard_d.conf内容示例
{% highlight java %}
#numactl --interleave=all /home/work/local/mongodb-2.6.4/bin/mongod -f etc/shard_d.conf
port=27021
fork=true
dbpath=/home/work/local/mongodb-2.6.4/data/shard_d/
logpath=/home/work/local/mongodb-2.6.4/logs/shard_d.log
logappend=true
keyFile=/home/work/local/mongodb-2.6.4/data/key/shard_key
shardsvr=true
replSet=shard_d
{% endhighlight %} 

##### config_server.conf内容示例
{% highlight java %}
#numactl --interleave=all /home/work/local/mongodb-2.6.4/bin/mongod -f etc/config_server.conf
port=27022
fork=true
dbpath=/home/work/local/mongodb-2.6.4/data/config/
logpath=/home/work/local/mongodb-2.6.4/logs/config.log
logappend=true
keyFile=/home/work/local/mongodb-2.6.4/data/key/shard_key
{% endhighlight %} 

##### mongos.conf内容示例
{% highlight java %}
#numactl --interleave=all /home/work/local/mongodb-2.6.4/bin/mongos -f etc/mongos.conf
port=27017
fork=true
logpath=/home/work/local/mongodb-2.6.4/logs/mongos.log
logappend=true
keyFile=/home/work/local/mongodb-2.6.4/data/key/shard_key
configdb=10.20.0.14:27022,10.20.0.15:27022,10.20.0.16:27022
{% endhighlight %} 


#### 3、启动服务
每个节点上的服务都配置好后，可以参考下面命令把每个节点上的服务启动
{% highlight java %}
mongod -f etc/shard_a.conf
mongod -f etc/shard_b.conf
mongod -f etc/shard_c.conf
mongod -f etc/shard_d.conf
mongod -f etc/config_server.conf
mongos -f etc/mongos.conf
{% endhighlight %}

#### 4、配置shard的副本集，指定投票节点，***可以选择在副本集中的任何一个节点执行***

{% highlight java %}
#shard_a
config = {_id: 'shard_a', 
config = {_id: 'shard_a', members: [
                          {_id: 0, host: '10.20.0.14:27018'},
                          {_id: 1, host: '10.20.0.15:27018'},
                          {_id: 2, host: '10.23.0.16:27018',arbiterOnly:true}]
           }
rs.initiate(config)

#shard_b
config = {_id: 'shard_b', members: [
                          {_id: 0, host: '10.20.0.15:27019'},
                          {_id: 1, host: '10.20.0.16:27019'},
                          {_id: 2, host: '10.23.0.17:27019',arbiterOnly:true}]
           }
#shard_c
config = {_id: 'shard_c', members: [
                          {_id: 0, host: '10.20.0.16:27020'},
                          {_id: 1, host: '10.20.0.17:27020'},
                          {_id: 2, host: '10.23.0.14:27020',arbiterOnly:true}]
           }
rs.initiate(config)

#shard_d
config = {_id: 'rs_shard_d', members: [
                          {_id: 0, host: '10.20.0.17:27021'},
                          {_id: 1, host: '10.20.0.14:27021'},
                          {_id: 2, host: '10.23.0.15:27021',arbiterOnly:true}]
           }
rs.initiate(config)           
{% endhighlight %}

####5、初始化数据库权限等,***在mongos节点执行***

* 初始admin密码及权限
{% highlight java %}
./mongo
use admin
db.createUser({user: "admin",pwd: "admin","roles" : [ "userAdminAnyDatabase", "dbAdminAnyDatabase", "readWriteAnyDatabase", "clusterAdmin" ]})
{% endhighlight %}

* 绑定shard信息
{% highlight java %}
db.runCommand( { addshard : "shard_a/10.20.0.14:27018,10.20.0.15:27018" } );
db.runCommand( { addshard : "shard_b/10.20.0.15:27019,10.20.0.16:27019" } );
db.runCommand( { addshard : "shard_c/10.20.0.16:27020,10.20.0.17:27020" } );
db.runCommand( { addshard : "shard_d/10.20.0.17:27021,10.20.0.14:27021" } );
{% endhighlight %}

* 创建示例数据库及指定管理账号信息

{% highlight java %}
use test;
db.createUser({user: "work",pwd: "work","roles" : [ "dbOwner"]})

#配置示例数据库user表启用分片信息及分片key
db.runCommand( { enablesharding : "test" } );
db.runCommand( { shardcollection : "test.user", key : {_id : 1} } );
{% endhighlight %}


