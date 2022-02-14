---
title: CAP理论
date: 2017-06-09 23:47:19
tags: [distributed systems]
---
### 介绍
CAP理论指对于一个分布式计算系统来说，不可能同时满足以下三点：
1. 一致性（Consistence) 

    一致性指的是所有节点总是访问到最新的数据，这里的一致性指的是强一致性，即[!Linearizability](http://cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf).
2. 可用性（Availability）

   可用性指对于每一个没有宕机的节点，总是可以返回正确的结果。这里的可用性不同于“高可用”（也就是低宕机时间）。
    
3. 分区容错性（Network Partitioning）

   分区容错性指即使发生了网络分区，节点仍然可以响应客户端的请求。  

CAP理论认为分布式系统最多只能CAP中的两点。

![](http://howtodoinjava.com/wp-content/uploads/2015/07/CAP-Theorem-Example-e1436534705620.png)

常见的单机数据库如MYSQL不提供分区容忍，属于AP的系统。

而对于近年来比较火热的NOSQL，设计时更多考虑水平扩展（Scale out），大多提供分区容忍。因而要在一致性和可用性之间做出选择。

以Cassandra和HBase为例，Cassandra的P2P模型提供了良好的水平扩展能力，
默认提供最终一致性，选择了CAP中的AP。Cassandra也可以配置成强一致性，
这时就要失去了CAP中的可用性A。

而HBase则是常见的主从架构，牺牲了可用性，选择了CP。

### CAP理论的不足

CAP理论提出以来，得到了计算机理论界的广泛认可。不过也有不少人指出CAP理论的不足之处。

DANIEL ABADI 在他的博文 [Problems with CAP, and Yahoo’s little known NoSQL system](http://dbmsmusings.blogspot.co.il/2010/04/problems-with-cap-and-yahoos-little.html)指出了三点CAP的不足。

1. CP系统表面的定义是支持一致性和分区容错性，没有可用性。但是一直不可用的系统是无用的。应该的定义是只有在发生网络分区时，才牺牲可用性。

2. CA 和 CP是相同的。发生网络分区时，才会有可用性的问题，这个系统是CP的。没有网络分区时，则是可用的（CA）。本质上是同一个系统的两种情况。

3. CAP理论无法描述分布式系统中的另外一个重要属性：延迟（latency)。


因此他建议使用 PACELC 来替换 CAP： 如果发生了网络分区（P），可用性（A）和一致性（C）如何选择，或者（E）在正常情况，延迟（L）和一致性（C）如何取舍。

Dynamo在发生网络分区时，牺牲了一致性（C），选择了可用性（A）。正常情况下，为了降低延迟（L），放弃了强一致性（C）。所以Dynamo是PA/EL。

MYSQL则始终提供强一致性（C），在某些情况下选择性的放弃了可用性（A）和低延迟（L）。所以MYSQL是PC/EC.

### 参考
* [CAP_theorem Wiki](https://en.wikipedia.org/wiki/CAP_theorem)
* [Problems with CAP, and Yahoo’s little known NoSQL system](http://dbmsmusings.blogspot.co.il/2010/04/problems-with-cap-and-yahoos-little.html)
* [Please stop calling databases CP or AP](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html)
<!-- more -->
