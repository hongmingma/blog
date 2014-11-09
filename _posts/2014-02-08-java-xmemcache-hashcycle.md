---
layout: post
title:  xmemcache Jedis 一致性hash的实现
category: [分布式]
---

redis,memcached在做大规模集群分片部署的情况时，客户端读写数据主要采用一致性hash的方式打散数据。
xmemcache，Jedis是用的比较多的客户端程序，二者关于一致性HASH部分的实现基本相同，因为同出一人之手，
最近有时间读了下源码把流程梳理下。

####1.	处理流程如下
	
* 根据服务器节点数和配置连接池信息初始化连接数（节点数*连接数）

* 根据每个节点 配置的权(weight)为每个节点虚拟节点数(160*weight)，之所以虚拟这么多节点是为了尽量避免请求时数据集中命中少数节点，同时权越高命中的几率越大
一致性HASH环如果节点过少很容易形成数据热点，不利于打散数据。至于为什么采用160而不是161应该是经验值，看作者的注释应该是从nginx学来的，我没有深究。

* 根据每个节点的name、weight、order计算出一致性hash值 顺时针0-2³²-1,构造一致性HASH环，用treemap存储映射关系。
	{% highlight java %}
	    private void initialize(List<S> shards) {
	        nodes = new TreeMap<Long, S>();
	        for (int i = 0; i != shards.size(); ++i) {
	            final S shardInfo = shards.get(i);
	            if (shardInfo.getName() == null)
	            	for (int n = 0; n < 160 * shardInfo.getWeight(); n++) {
	            		nodes.put(this.algo.hash("SHARD-" + i + "-NODE-" + n), shardInfo);
	            	}
	            else
	            	for (int n = 0; n < 160 * shardInfo.getWeight(); n++) {
	            		nodes.put(this.algo.hash(shardInfo.getName() + "*" + shardInfo.getWeight() + n), shardInfo);
	            	}
	            resources.put(shardInfo, shardInfo.createResource());
	        }
	    }
	{% endhighlight %}
	
* 客户端请求时同样计算key的一致性hash值，根据值去hash环上顺时针找离它最近的一个虚拟节点，
然后根据该节点的信息从该节点建立的连接池中随机拿出一个连接创建读写的会话，从而实现并发读写

####2.	一致性HASH的优点

 一致性hash的好处是当服务节点增加时数据访问命中率与原来相同，扩容方便，当服务节点减少时只有逆时针方向与该节点相关的一片区域的请求不能命中 ，
其他节点不失效，hash取莫对扩展情况时数据则完全失效，代价太大

####3.	构建一致性HASH环与数据节点映射的小技巧

存取一致性hash值与节点对应关系用treemap能简化工作，因为treemap本身是有序的。并且tailMap(fromKey)方法返回这个Map的一个有序子集，
其键均大于等于fromKey [凸，treemap的lib竟然没有有效注释的]，所以两部操作就可以找到一条数据应该关联的数据节点。

查找数据节点映射关系的片段代码：
{% highlight java %}
public S getShardInfo(byte[] key) {
        SortedMap<Long, S> tail = nodes.tailMap(algo.hash(key));
        if (tail.size() == 0) {
            return nodes.get(nodes.firstKey());
        }
        return tail.get(tail.firstKey());
    }
{% endhighlight %}






 