---
layout:     post
title:      "详解开源大数据引擎Greenplum的架构和技术特点"
date:       2017-04-21 13:42:00
author:     "zhouleihao"
header-img: "http://gpdb.docs.pivotal.io/4390/graphics/highlevel_arch.jpg"
---
>《程序员》3月技术板投稿。

### 什么是Greenplum
大数据引擎Greenplum（以下简称GPDB）是一款开源的数据仓库。基于开源的PostgreSQL改造，主要用来处理大规模的数据分析任务，相比现在的Hadoop，Greenplum更适合做大数据的存储、计算和分析引擎。Greenplum项目基于Apache 2许可发布。

### Greenplum的MPP架构
GPDB是典型的Master/Slave架构，在Greenplum集群中，存在一个Master节点和多个Segment节点，其中每个节点上可以运行多个数据库。与Oracle RA的Shared Everynothing架构不同，Greenplum采用shared nothing架构（MPP）。简单来说，Shared Nothing是一个分布式的架构，每个节点相对独立。典型的Shared Nothing系统会集数据库、内存Cache、等存储状态的信息；而不在节点上保存状态的信息。节点点之间的信息交互都是通过节点互联网络实现。通过将数据分布到多个节点上来实现规模数据的存储，通过并行查询处理来提高查询性能。每个节点仅查询自己的数据。所得到的结果再经过主节点处理得到最终结果。通过增加节点数目达到系统线性扩展。

![Greenplum架构图](http://gpdb.docs.pivotal.io/4390/graphics/highlevel_arch.jpg "Greenplum架构图")

如上图为GPDB的基本架构，客户端通过网络连接到gpdb，其中Master Host是GP的主节点（客户端的接入点），Segment Host是子节点（连接并提交SQL语句的接口），主节点是不存储用户数据的，子节点存储数据并负责SQL查询，主节点负责相应客户端请求并将请求的SQL语句进行转换，转换之后调度后台的子节点进行查询，并将查询结果返回客户端。

#### Greenplum Master
Master只存储系统元数据，业务数据全部分布在Segments上。其作为整个数据库系统的入口，负责建立与客户端的连接，SQL的解析并形成执行计划，分发任务给Segment实例，并且收集Segment的执行结果。正因为Master不负责计算，所以Master不会成为系统的瓶颈。

Master节点的高可用，类似于Hadoop的NameNode HA，如下图，Standby Master通过synchronization process，保持与Primary Master的catalog和事务日志一致，当Primary Master出现故障时，Standby Master承担Master的全部工作。
![Greenplum架构图](http://gpdb.docs.pivotal.io/4390/graphics/standby_master.jpg "Greenplum架构图")


#### Segments
Greenplum中可以存在多个Segment，Segment主要负责业务数据的存储和存取，用户查询SQL的执行，每个Segment存放一部分用户数据，但是用户不能直接访问Segment，所有对Segment的访问都必须经过Master。进行数据访问时，所有的Segment先并行处理与自己有关的数据，如果需要关联处理其他Segment上的数据，Segment可以通过Interconnect进行数据的传输。Segment节点越多，数据就会打的越散，处理速度就越快。因此与Share All数据库集群不同，通过增加Segment节点服务器的数量，Greenplum的性能会成线性增长。


![Greenplum架构图](http://gpdb.docs.pivotal.io/4390/graphics/group-mirroring.png "Greenplum架构图")

每个Segment的数据冗余存放在另一个Segment上，数据实时同步，当Primary Segment失效时，Mirror Segment将自动提供服务，当Primary Segment恢复正常后，可以很方便的使用gprecoverseg -F工具来同步数据。

#### Interconnect
Interconnect是Greenplum架构中的网络层，是GPDB系统的总要组件，默认情况下，使用UDP协议，但是Greenplum会对数据包进行校验，因此可靠性等同于TCP，但是性能上会更好。在使用TCP协议的情况下，Segment的实例不能超过1000，但是使用UDP则没有这个限制。

![Greenplum架构图](http://gpdb.docs.pivotal.io/4390/graphics/multi_nic_arch.jpg "Greenplum架构图")

### Greenplum，新的解决方案
前面介绍了GPDB的基本架构，让读者对GPDB有了初步的了解，下面对GPDB的部分特性描述可以很好的理解为什么选择GPDB作为新的解决方案。

#### 丰富的工具包，运维从此不是事儿
对比开源社区的其他项目在运维上的困难，GPDB提供了丰富的管理工具，图形化的web监控页面，帮助管理员更好的管理集群，监控集群本身以及所在服务器的运行状况。

最近的公有云集群迁移过程中，impala总查询段达到100的时候，系统开始变得极不稳定，后来在外援的帮助下发现是系统内核本身的问题，在恶补系统内核参数的同时，发现GPDB的工具也变相的填充了我们的短板，比如提供了gpcheck和gpcheckperf等命令，用以检测GPDB运行所需要的系统配置是否合理以及对相关硬件做性能测试，如下，执行gpcheck命令后，检测sysctl.conf中参数的设置是否符合要求，如果对参数的含义感兴趣，可以自行百度学习。

	[gpadmin@gzns-waimai-do-hadoop280 greenplum]$ gpcheck --host mdw
	variable not detected in /etc/sysctl.conf: 'net.ipv4.tcp_max_syn_backlog'
	variable not detected in /etc/sysctl.conf: 'kernel.sem'
	variable not detected in /etc/sysctl.conf: 'net.ipv4.conf.all.arp_filter'
	/etc/sysctl.conf value for key 'kernel.shmall' has value '4294967296' and expects '4000000000'
	variable not detected in /etc/sysctl.conf: 'net.core.netdev_max_backlog'
	/etc/sysctl.conf value for key 'kernel.sysrq' has value '0' and expects '1'
	variable not detected in /etc/sysctl.conf: 'kernel.shmmni'
	variable not detected in /etc/sysctl.conf: 'kernel.msgmni'
	/etc/sysctl.conf value for key 'net.ipv4.ip_local_port_range' has value '10000  65535' and expects '1025 65535'
	variable not detected in /etc/sysctl.conf: 'net.ipv4.tcp_tw_recycle'
	hard nproc not found in /etc/security/limits.conf
	soft nproc not found in /etc/security/limits.conf

另外，在安装过程中，用其提供的gpssh-exkeys命令打通所有机器免密登录后，可以很方便的使用gpassh命令对所有的机器批量操作，如下图演示了在master主机上执行gpssh命令后，在集群的五台机器上批量执行pwd命令。

	[gpadmin@gzns-waimai-do-hadoop280 greenplum]$ gpssh -f hostlist 
	=> pwd
	[sdw3] /home/gpadmin
	[sdw4] /home/gpadmin
	[sdw5] /home/gpadmin
	[ mdw] /home/gpadmin
	[sdw2] /home/gpadmin
	[sdw1] /home/gpadmin
	=> 

诸如上述的工具GPDB还提供了很多，比如恢复segment节点的gprecoverseg命令，比如切换主备节点的gpactivatestandby命令，等等。这类工具的提供让集群的维护变得很简单，当然我们也可以基于强大的工具包开发自己的管理后台，让集群的维护更加的傻瓜化。

#### 查询计划和并行执行，SQL优化利器
查询计划包括了一些传统的操作，比如：扫表，关联，聚合，排序等。另外，GPDB有一个特定的操作：移动（motion）。移动操作涉及到查询处理期间在Segment之间移动数据。

下面的SQL是TPCH中Query 1的简化版，用来简单描述查询计划。

		explain select
		   	    o_orderdate,
		   	    o_shippriority
		from
		        customer,
		        orders
		where
		        c_mktsegment = 'MACHINERY'
		        and c_custkey = o_custkey
		        and o_orderdate < date '1995-03-20'
		LIMIT 10;
	
	                                                       QUERY PLAN
		----------------------------------------------------------------------------------------------------------
		Limit  (cost=98132.28..98134.63 rows=10 width=8)
		   ->  Gather Motion 10:1  (slice2; segments: 10)  (cost=98132.28..98134.63 rows=10 width=8)
	         ->  Limit  (cost=98132.28..98134.43 rows=1 width=8)
	               ->  Hash Join  (cost=98132.28..408214.09 rows=144469 width=8)
	                     Hash Cond: orders.o_custkey = customer.c_custkey
	                     ->  Append-only Columnar Scan on orders  (cost=0.00..241730.00 rows=711519 width=16)
	                           Filter: o_orderdate < '1995-03-20'::date
	                     ->  Hash  (cost=60061.92..60061.92 rows=304563 width=8)
	                           ->  Broadcast Motion 10:10  (slice1; segments: 10)  (cost=0.00..60061.92 	rows=304563 width=8)
	                                 ->  Append-only Columnar Scan on customer  (cost=0.00..26560.00 	rows=30457 width=8)
	                                       Filter: c_mktsegment = 'MACHINERY'::bpchar
		 Settings:  enable_nestloop=off
		 Optimizer status: legacy query optimizer

执行计划执行从下至上，可以看到每个计划节点操作的额外信息。

1. 在各个节点上扫描各自节点所存储的customer表数据，按照条件过滤生成数据rs
2. 各节点将自己生成的rs依次发送到其他节点。
3. 各个节点上的orders表的数据，和各自节点上收到的rs进行join。这样就保证本机上的数据只和本机的数据join。
4. 各个节点将join后的结果发送给master
 
上面的执行过程可以看出，GPDB是将rs给每个含有orders表数据的节点都发了一份的。为了有效传输，Greenplum是选择将小表广播数据，而不是将大表广播。为了最大限度的实现并行化处理，GPDB会将查询计划分成多个处理步骤。在查询执行期间，分发到Segment上的各部分会并行的执行一系列的处理工作，并且只处理属于自己部分的工作。重要的是，可以在同一个主机上启动多个postgresql数据库进行更多表的关联以及更复杂的查询操作，单台机器的性能得到更加充分的发挥。

> **如何查看执行计划？**
> 
> 如果一个查询表现出很差的性能，可以通过查看执行计划找到可能的问题点。
> 
> * 计划中是否有一个操作花费时间超长？
> * 规划期的评估是否接近实际情况？
> * 选择性强的条件是否较早出现？
> * 规划期是否选择了最佳的关联顺序？
> * 规划其是否选择性的扫描分区表？
> * 规划其是否合适的选择了Hash聚合与Hash关联操作？


#### 高效的数据导入，批量不再是瓶颈
前面提到，Greenplum的Master节点只负责客户端交互和其他一些必要的控制，而不承担任何的计算任务。在加载数据的时候，会先进行数据分布的处理工作，为每个表指定一个分发列，接下来，所有的节点同时读取数据，根据选定的Hash算法，将当前节点数据留下，其他数据通过interconnect传输到其他节点上去，保证了高性能的数据导入。通过结合外部表和gpfdist服务，GPDB可以做到每小时导入2TB数据，在不改变ETL流程的情况下，可以从impala快速的导入计算好的数据为消费提供服务。

> 使用gpfdist的优势在于其可以确保再度去外部表的文件时，GPDB系统的所有Segment可以完全被利用起来，但是需要确保所有Segment主机可以具有访问gpfdist的网络。


#### 其他
* GPDB支持LDAP认证，这一特性的支持，让我们可以把目前Impala的角色权限控制无缝的迁移到GPDB。
* GPDB基于Postgresql 8.2开发，通过psql命令行工具可以访问GPDB数据库的所有功能，另外支持JDBC、ODBC等访问方式，产品接口层只需要进行少量的适配即可使用GPDB提供服务。
* GPDB支持基于资源队列的管理，可以为不同类型工作负载创建资源独立的队列，并且有效的控制用户的查询以避免系统超负荷运行。比如，可以为VIP用户，ETL生产，任性和adhoc等创建不同的资源队列。同时，支持优先级的设置，在并发争用资源时，高优先级队列的语句将可以获得比低优先级资源队列语句更多的资源。


最近在对GPDB做调研和测试，过程中用TPCH做性能的测试，通过和网络上其他服务的对比发现在5个节点的情况下已经有了很高的查询速度，但是由于测试环境服务器问题，具体的性能数据还要在接下来的新环境中得出，不过GPDB基于postgresql开发，天生支持丰富的统计函数，支持横向的线性扩展，内部容错机制，有很多功能强大的运维管理命令和代码，相比impala而言，显然在SQL的支持，实时性和稳定性上更胜一筹。

本文只是对Greenplum的初窥，接下来更深入的剖析以及在工作中的实践经验分享也请关注DA的wiki。更多的关于Greenplum基本的语法和特性，也可以参考PostgreSQL的官方文档。

### 参考
[Pivotal Greenplum® Database 4.3.9.1 Documentation](http://gpdb.docs.pivotal.io/4390/common/welcome.html)