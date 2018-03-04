ReplSet 初始化
===========================
### 功能介绍
目前在mongodb中，config和shardserver会使用ReplSet存储数据。在mongod进程启动过程中，会进行ReplSet的初始化。整个初始化过程可以概括为初始化以下内容：

- 所涉及到对象的实例化
- 状态的初始化变迁
- 后台任务资源的初始化

ReplSet所涉及到的对象主要包括ReplicationCoordinator、TopologyCoordinator、ReplicationCoordinatorExternalState、StorageInterface以及ReplicationExecutor。其中ReplicationCoordinator作为ReplSet功能实体的抽象，统一协调负责ReplSet所有功能的运转，TopologyCoordinator主要用于副本集之间拓扑结构的维护，具体主要包括一致性保证、leader election以及副本集配置管理等。ReplicationCoordinatorExternalState主要为ReplSet提供访问已经持久化到存储的资源接口，ReplicationExecutor为副本集的运转提供调度功能。

ReplSet中两个部分涉及状态变迁，一个是ReplicaSetConfig，一个是MemberState。前者主要是ReplSet的配置信息经历从持久化存储到内存加载过程所对应的状态变化，后者是服务本身作为ReplSet副本集的成员身份的状态变迁。

后台任务资源主要是涉及oplog同步使用到的后台任务线程的初始化，主要包括bgsync、applier以及reporter。


### 主要类图
与mongodb副本集实现相对应的核心抽象类为ReplicationCoordinator，实现类为ReplicationCoordinatorImpl。副本集功能所涉及的类主要包括：
![副本集主要类图](../pics/10replset/replset_init)


### 初始化流程

###### 1）对象的创建
mongodb有一个初始化框架ReplicationCoordinatorImpl，主要实现服务资源等一些全局信息的初始化。Replset的初始化入口在db.cpp这个文件中的MONGO_INITIALIZER_WITH_PREREQUISITES这个函数，它主要创建了StorageInterfaceImpl、ReplicationCoordinatorImpl和TopologyCoordinatorImpl。并通过repl::XXX::set()函数保存这些对象。

```
MONGO_INITIALIZER_WITH_PREREQUISITES(CreateReplicationManager,
                                     ("SetGlobalEnvironment", "SSLManager"))
(InitializerContext* context) {
    auto serviceContext = getGlobalServiceContext();
    repl::StorageInterface::set(serviceContext, stdx::make_unique<repl::StorageInterfaceImpl>());
    auto storageInterface = repl::StorageInterface::get(serviceContext);

    repl::TopologyCoordinatorImpl::Options topoCoordOptions;
    topoCoordOptions.maxSyncSourceLagSecs = Seconds(repl::maxSyncSourceLagSecs);
    topoCoordOptions.clusterRole = serverGlobalParams.clusterRole;

    auto replCoord = stdx::make_unique<repl::ReplicationCoordinatorImpl>(
        getGlobalReplSettings(),
        new repl::ReplicationCoordinatorExternalStateImpl(storageInterface),
        executor::makeNetworkInterface("NetworkInterfaceASIO-Replication").release(),
        new repl::TopologyCoordinatorImpl(topoCoordOptions),
        storageInterface,
        static_cast<int64_t>(curTimeMillis64()));
    repl::ReplicationCoordinator::set(serviceContext, std::move(replCoord));
    repl::setOplogCollectionName();
    return Status::OK();
}
```
###### 2）副本集的启动
主要是调用ReplicationCoordinatorImpl的startup接口，代码在db.cpp中_initAndListen()函数的以下代码

```
repl::getGlobalReplicationCoordinator()->startup(startupOpCtx.get());
```
该startup会调用到ReplicationCoordinatorImpl的startup函数。

```
void ReplicationCoordinatorImpl::startup(OperationContext* txn) {
    if (!isReplEnabled()) {
        stdx::lock_guard<stdx::mutex> lk(_mutex);        
        <font color=gray>_setConfigState_inlock(kConfigReplicationDisabled);</font>
        
        return;
    }

    {
        OID rid = _externalState->ensureMe(txn);

        stdx::lock_guard<stdx::mutex> lk(_mutex);
        fassert(18822, !_inShutdown);
        _setConfigState_inlock(kConfigStartingUp); //设置ConfigState的初始状态
        _myRID = rid;
        _slaveInfo[_getMyIndexInSlaveInfo_inlock()].rid = rid;
    }
`
    if (!_settings.usingReplSets()) {
		...
    }

    _replExecutor.startup(); //启动副本集的执行器

    _topCoord->setStorageEngineSupportsReadCommitted(
        _externalState->isReadCommittedSupportedByStorageEngine(txn)); //根据底层存储引擎是否支持snapshot设置topology的readcommit标志

    bool doneLoadingConfig = _startLoadLocalConfig(txn); //将本地副本集数据load到内存中
    if (doneLoadingConfig) {
        // If we're not done loading the config, then the config state will be set by
        // _finishLoadLocalConfig.
        stdx::lock_guard<stdx::mutex> lk(_mutex);
        invariant(!_rsConfig.isInitialized());
        _setConfigState_inlock(kConfigUninitialized);
    }
}
```
在现有实现中，加载本地副本集数据的过程被分为了两个部分分别放在_startLoadLocalConfig和_finishLoadLocalConfig两个函数中，前半部分主要是加载ReplicaSetConfig和最新的oplog纪录，并且判断是否有oplog需要被apply。而后半部分主要是启动副本集使用到的用于同步数据的后台任务，以及后续副本集拓扑结构使用到的心跳以及选举任务。

_startLoadLocalConfig的实现如下。

```
bool ReplicationCoordinatorImpl::_startLoadLocalConfig(OperationContext* txn) {
	//从local.replset.election表中读出上次的投票信息，主要的field包括["id", "term", "candidateIndex"]
    StatusWith<LastVote> lastVote = _externalState->loadLocalLastVoteDocument(txn);
    if (!lastVote.isOK()) {
		...
    } else {
        LockGuard topoLock(_topoMutex);
        _topCoord->loadLastVote(lastVote.getValue());
    }

	//从local.system.replset表中读出副本集的配置信息并加载到内存中
    StatusWith<BSONObj> cfg = _externalState->loadLocalConfigDocument(txn);
    if (!cfg.isOK()) {
		...
    }
    ReplicaSetConfig localConfig;
    Status status = localConfig.initialize(cfg.getValue());
    if (!status.isOK()) {
		...
    }

    //从local.replset.minvalid表中读取oplogDeleteFromPoint字段的值以及begin的值，其中begin的值纪录了当前apply到哪个oplog，通过begin的oplog和local.oplog.rs表中最新的oplog相比较确认是否有未applied得oplog，如果有则一条一条apply这些oplog。
    _externalState->cleanUpLastApplyBatch(txn);
    
    //从local.oplog.rs中读取最新的一条oplog记录
    auto lastOpTimeStatus = _externalState->loadLastOpTime(txn);

    //异步执行后续的初始化任务
    auto handle = _scheduleDBWork(stdx::bind(&ReplicationCoordinatorImpl::_finishLoadLocalConfig,
                                             this,
                                             stdx::placeholders::_1,
                                             localConfig,
                                             lastOpTimeStatus,
                                             lastVote));
    {
        stdx::lock_guard<stdx::mutex> lk(_mutex);
        _finishLoadLocalConfigCbh = handle;
    }

    return false;
}
```
