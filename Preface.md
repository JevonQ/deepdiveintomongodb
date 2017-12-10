MongoDB源码分析
===
##引言
&emsp;&emsp;由于工作原因，最近半年开始接触 *mongodb* 并对其进行深入优化改造，出于后期对于维护的把控以及能够做出更多合理的重构和修改，所以对其整体源码的分析就显得很有必要。但整个 *mongodb* 接近50w行的代码量非常庞大，如何高效的进行阅读，是得有一定的方法论的。当然因人而异，不同人有不同的方法，这里我按照自己的方法逐步肢解 *mongodb* 的源码，将其中的设计思想和实现考量展示出来，一是作为自己学习的总结，二也希望作为其他想要深入了解mongodb的人员的一个参考。

&emsp;&emsp;接下来，会按照以下方面展开介绍:

1. mongodb 简介
	* mongodb的发展历程（sql、nosql、newsql）
	* mongodb的优势和不足
	* mongodb的应用场景
	* mongodb的生态
2. mongodb 架构设计和源码树肢解
	* mongodb的设计哲学
	* mongodb的架构设计
	* mongodb的部署安装
	* mongodb的源码树肢解
3. mongodb 基础组件介绍
	* stdx模块解析
	* BSON解析
	* Logger解析
	* Concurrency模块解析
	* TaskExecutor解析
	* Failpoint框架解析
4. mongodb 服务启动流程
	* mongodb initializer框架解析
	* mongod 启动流程
	* mongos 启动流程
	* mongo 启动流程
5. mongodb 数据管理模型
	* 逻辑组织Database/Collection/Document/Index
		* catalog管理机制解析
		* 创建database流程
		* 创建collection流程
	* 数据存储BSON/_id/polymorphic schema
6. mongodb sharding 机制
	* sharding 原理及概念介绍
	* sharding初始化流程（ShardingState和OperationShardingState）
	* shard和chunk
	* sharding catalog
	* data distribution过程(shard key)
7. mongodb 数据存储引擎
	* 整体设计(类图关系抽象设计，如何支持pluggable的存储引擎)
	* RocksEngine(mongorocks和rocksdb)
	* WiredTiger
	* MmapV1
8. mongodb 网络通信
	* 通信模块整体设计
	* Message
	* TransportLayer
	* Session
	* Connection
	* MessagingPort
	* Network interface
	* ASIO 框架介绍
	* Message在网络层的接收发送流程
9. mongodb I/O
	* I/O 框架设计
		* Command 框架解析
		* I/O 框架解析
		* 事务一致性(事务和snapshot)
		* 锁机制解析
	* Indexing机制
		* Index类型介绍
		* Index的实现
	* Query过程
		* Cursor
		* Query流程分析
	* 插入更新删除流程
		* 插入流程
		* 更新流程
		* 删除流程
10. mongodb 副本集
	* 副本集初始化
	* 副本集管理
	* 心跳机制分析
	* 选举过程分析
	* oplog同步过程
11. mongodb load-balancing
	* balancer启动过程
	* 负载均衡策略
	* 负载信息收集
	* chunk分裂过程
	* chunk迁移过程
	* chunk合并过程
