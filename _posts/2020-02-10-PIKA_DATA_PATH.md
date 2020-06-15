---
layout: post
title: Pika传火计划之读写流程
---

<br>
<div style="text-align: center">
<img width = '800' height ='800' src ="https://s1.ax1x.com/2020/05/08/YnbjQf.png">
</div>


### Introduction

通过上次[Pika线程模型](https://whoiami.github.io/PIKA_THREAD_MODEL)的分享，得知主要的命令处理是由线程池的线程负责的。而命令的通用处理流程主要是由PikaClientConn决定的，在其处理过程当中对于不同的命令，通过多态的方式调class Cmd处理接口，动态选择不同命令的处理函数。这里主要梳理pika的主要读写流程。


<br>

### PikaClientConn和Cmd通用处理流程

#### WorkerThread 处理流程

```c++
class PikaClientConn: public pink::RedisConn {
  void AsynProcessRedisCmds(const std::vector<pink::RedisCmdArgsType>& argvs,
      std::string* response) override;
  std::atomic<int> resp_num;
  std::vector<std::shared_ptr<std::string>> resp_array;
  std::shared_ptr<Cmd> DoCmd(const PikaCmdArgsType& argv,
      const std::string& opt,
      std::shared_ptr<std::string> resp_ptr);
}
```

Pink层通过AsynProcessRedisCmds的调用，Pika上层可以自己定义对于接受命令后的后续处理流程。

```c++
void PikaClientConn::AsynProcessRedisCmds(
    const std::vector<pink::RedisCmdArgsType>& argvs, std::string* response){
  BgTaskArg* arg = new BgTaskArg();
  arg->redis_cmds = argvs;
  arg->conn_ptr =
    std::dynamic_pointer_cast<PikaClientConn>(shared_from_this());
  g_pika_server->ScheduleClientPool(&DoBackgroundTask, arg);
}
```

1，worker thread 调用AsynProcessRedisCmds，将待处理Cmd封装成BgTaskArg。

2，BgTaskArg任务放入线程池中，后续由线程池中的一个线程继续处理这个请求。

3，worker thread 的调用返回，worker thread 继续运行自己流程。


<br>

#### ThreadPoolThread 处理流程

```c++
void PikaClientConn::DoBackgroundTask(void* arg) {
  // sanity check
  // ...
  BgTaskArg* bg_arg = reinterpret_cast<BgTaskArg*>(arg);
  std::shared_ptr<PikaClientConn> conn_ptr = bg_arg->conn_ptr;
  conn_ptr->BatchExecRedisCmd(bg_arg->redis_cmds);
  delete bg_arg;
}

void PikaClientConn::BatchExecRedisCmd(
    const std::vector<pink::RedisCmdArgsType>& argvs) {
  resp_num.store(argvs.size());
  for (size_t i = 0; i < argvs.size(); ++i) {
    std::shared_ptr<std::string> resp_ptr = std::make_shared<std::string>();
    resp_array.push_back(resp_ptr);
    ExecRedisCmd(argvs[i], resp_ptr);
  }
  TryWriteResp();
}

void PikaClientConn::ExecRedisCmd(
    const PikaCmdArgsType& argv, std::shared_ptr<std::string> resp_ptr) {
  std::string opt = argv[0];
  slash::StringToLower(opt);
  std::shared_ptr<Cmd> cmd_ptr = DoCmd(argv, opt, resp_ptr);
}
```

1，ThreadPoolThread调用DoBackgroundTask，检查BgTaskArg 的合法性。

2，调用BatchExecRedisCmd，在此线程中对所有命令进行逐一处理。

3，调用DoCmd 进行命令的具体处理。

4，调用TryWriteResp 对于返回的所有结果整合，之后通知WorkerThread 该PikaClientConn内的结果可以写回客户端。

<br>

DoCmd的处理流程如下。

```c++
std::shared_ptr<Cmd> PikaClientConn::DoCmd(
    const PikaCmdArgsType& argv,
    const std::string& opt,
    std::shared_ptr<std::string> resp_ptr) {
  std::shared_ptr<Cmd> c_ptr = g_pika_cmd_table_manager->GetCmd(opt);
  
  if (!auth_stat_.IsAuthed(c_ptr)) {
    c_ptr->res().SetRes(CmdRes::kErrOther,"NOAUTH Authentication required.");
    return c_ptr;
  }
  // lock free
  // slowlog_slower_thann is atomic int
  if (g_pika_conf->slowlog_slower_than() >= 0) {
    start_us = slash::NowMicros();
  }
  // lock free
  // HasMonitorClients return atomic bool
  bool is_monitoring = g_pika_server->HasMonitorClients();
  if (is_monitoring) {
    ProcessMonitor(argv);
  }
  
  // Initial
  c_ptr->Initial(argv, current_table_);
  if (!c_ptr->res().ok()) {
    return c_ptr;
  }
  // partial lock free
  // update server statistic lock free
  // pdateTableQps NOT lock free
  g_pika_server->UpdateQueryNumAndExecCountTable(
    current_table_, opt, c_ptr->is_write());
  // sanity check
  ... 
  // Process Command
  c_ptr->Execute();

  if (g_pika_conf->slowlog_slower_than() >= 0) {
    ProcessSlowlog(argv, start_us);
  }
}
```

1，根据具体命令生成其基类的std::shared_ptr\<Cmd\> 方便多态实现。

2，对于连接进行权限认证，对应命令可以查看[Redis Auth](https://redis.io/commands/auth)命令，和Pika配置文件[Pika配置文件说明](https://github.com/Qihoo360/pika/wiki/pika-配置文件说明) 中对于密码的相关配置。

3，将命令放入monitor线程，对应命令可以查看[Redis Monitor](https://redis.io/commands/monitor) 命令。

4，调用Cmd::Initial。

5，调用Cmd::Execute。

6，如果开启Slowlog，则记录Slowlog，对应命令可以查看[Slowlog](https://redis.io/commands/slowlog)命令。


<br>

### Cmd 通用处理流程

在PikaClientConn的通用处理流程中，对于不同Cmd的操作都是调用其基类处理函数Initial和Execute，Initial和Execute函数内部会调用纯虚函数DoInitial和Do，通过多态查找派生类的真正实现。

```c++
class Cmd: public std::enable_shared_from_this<Cmd> {
  virtual void DoInitial() = 0;
  virtual void Do(std::shared_ptr<Partition> partition = nullptr) = 0;
  void Cmd::Initial(const PikaCmdArgsType& argv,
                  const std::string& table_name) {
    argv_ = argv;
    table_name_ = table_name;
    res_.clear(); // Clear res content
    Clear();      // Clear cmd, Derived class can has own implement
    DoInitial();
  };
  void Cmd::Execute() {
    ...
    if (g_pika_conf->classic_mode()...) {
      // invoke InternalProcessCommand and Cmd::Do
      ProcessSinglePartitionCmd();
    } else {
      ...
    }
  };
  void Cmd::InternalProcessCommand(std::shared_ptr<Partition> partition,
    std::shared_ptr<SyncMasterPartition> sync_partition) {
    slash::lock::MultiRecordLock record_lock(partition->LockMgr());
    if (is_write()) {
      record_lock.Lock(current_key());
    }
    // invoke Cmd::Do
    DoCommand(partition, hint_keys);
    DoBinlog(sync_partition);
    if (is_write()) {
      record_lock.Unlock(current_key());
    }
  }
}
```

任何具体的命令继承Cmd之后，需要实现DoInitial和Do 两个纯虚函数。在之后的通用处理流程中Cmd会做相应的调用。Cmd对外主要暴露Initial 和Execute 两个接口。

1，Initial清除前一次调用的残留数据，同时调用DoInitial虚函数。

2，Execute判断pika运行模式，主要调用InternalProcessCommand。

2.1，对于操作DB 和Binlog 这两个动作加锁，确保DB 和Binlog 是一致的。

2.2，调用DoCommand，其内部主要调用Do 虚函数。

2.3，调用DoBinlog，将命令处理后写入Binlog。

<br>


#### DoCommand

DoCommand的作用主要是将命令写入DB。

```c++
void Cmd::DoCommand(
    std::shared_ptr<Partition> partition, const HintKeys& hint_keys) {
  if (!is_suspend()) {
    partition->DbRWLockReader();
  }
  Do(partition);
  if (!is_suspend()) {
    partition->DbRWUnLock();
  }
}
```

BGSAVE，FLUSHALL，FLUSHDB除了之外，其余所有命令在执行Do函数之前都需要加读锁。对于这几个特殊的命令而言，它们的共同点是都需要清除数据，为保证清除过程没有其它操作同时进行，需要对相应的分片或者db加上写锁阻塞其他操作。具体来说，它们Do的函数实现内部会直接调DbRWLockWriter，阻塞其它操作。



<br>

#### DoBinlog

DoBinlog的作用主要是将命令写入Binlog。

```c++
void Cmd::DoBinlog(std::shared_ptr<SyncMasterPartition> partition) {
  Status s = partition->ConsensusProposeLog(shared_from_this(),
        std::dynamic_pointer_cast<PikaClientConn>(conn_ptr), resp_ptr);
}
```


通过 ConsensusProposeLog => InternalAppendBinlog => (std::shared_ptr\<Binlog\>)Logger()->Put(binlog) 一系列的函数调用，最终调用class Binlog的Put接口将，binlog 字符串写入Binlog 文件当中。

![](/public/images/2020-02-10/binlog.png)

Binlog文件是由一个一个Blocks组成的，这样组织主要防止binlog文件的某一个点损坏造成整个文件不可读。每一个binlog 字符串先序列化成BinlogItem 结构，如黄色板块所示，组成BinlogItem之后，再加上8个bytes（Length，Time，Type）组成完整的可以落盘的数据。


<br>

### 命令执行过程的差异化处理

以上讨论了Pika的通用处理流程，所有命令的处理都要经过以上的处理流程，对于每一条命令的具体处理细节，由具体的命令实现决定。下面以SetCmd为例。

```c++
class SetCmd : public Cmd {
  virtual void DoInitial() override;
  virtual void Do(std::shared_ptr<Partition> partition = nullptr);
  void GetCmd::DoInitial() {
    if (!CheckArg(argv_.size())) {
      res_.SetRes(CmdRes::kWrongNum, kCmdNameGet);
      return;
    }
    key_ = argv_[1];
    return;
  }
  void SetCmd::Do(std::shared_ptr<Partition> partition) {
    switch (condition_) {
      ...
      case SetCmd::kNX:
        s = partition->db()->Setnx(key_, value_, &res, sec_);
        break;
      default:
        s = partition->db()->Set(key_, value_);
        break;
    }
    ...
  }
}
```

SetCmd的DoInitial实现主要初始化未继承自Cmd的数据。

SetCmd的Do实现主要是根据Set命令的几种变形进行不同的Blackwidow接口调用。

<br>

以上我们介绍了Pika主要的读写流程，但是在一致性场景下我们不能够完全按照以上的读写路径进行处理，下面我们来看一下一致性场景下数据的读写流程。


<br>

### 一致性实现中的数据写入

一致性场景下，并不是像上面所说DoCommand 和DoBinlog 在一起执行的。一致性场景下，需要在Leader的Execute中做DoBinlog，然后对于这条Binlog在得到一定数目的Follower确认之后，利用存下来的MemLog::LogItem中的PikaClientConn 和Cmd指针，调用DoExecTask，其中再次调用Execute，进行DoCommand 的操作。

由于性能考虑，DoCommand的操作需要多线程并发执行，这样一条Conn的命令就有可能被几个线程同时执行，那么如何保证运行结果的正确性呢。

在PikaClientConn 中记录了当前Conn需要执行的子命令的个数和所有子命令的response指针数组。一致性场景中会将该子命令对应的resp_ptr指针与PikaClientConn 和Cmd 存成MemLog::LogItem存下来，当这条子命令在得到一定数目的Follower确认之后，将当前LogItem执行的结果写入LogItem 中的resp_ptr 中并且resp_num 自减，执行最后一个命令的线程负责将其他所有线程的执行结果组合，返回客户端。

```c++
void PikaClientConn::BatchExecRedisCmd(
    const std::vector<pink::RedisCmdArgsType>& argvs) {
  resp_num.store(argvs.size());
  for (size_t i = 0; i < argvs.size(); ++i) {
    std::shared_ptr<std::string> resp_ptr = std::make_shared<std::string>();
    resp_array.push_back(resp_ptr);
    ExecRedisCmd(argvs[i], resp_ptr);
  }
  TryWriteResp();
}

void PikaClientConn::ExecRedisCmd(
    const PikaCmdArgsType& argv, std::shared_ptr<std::string> resp_ptr) {
  std::shared_ptr<Cmd> cmd_ptr = DoCmd(argv, opt, resp_ptr);
  // level == 0 or (cmd error) or (is_read)
  if (g_pika_conf->consensus_level() == 0
      || !cmd_ptr->res().ok() || !cmd_ptr->is_write()) {
    *resp_ptr = std::move(cmd_ptr->res().message());
    resp_num--;
  }
}
```

1，调用BatchExecRedisCmd，初始化resp_num，为此次请求子命令的个数。

2，初始化resp_ptr，存入PikaClientConn指针数组，传入ExecRedisCmd。

3，在ExecRedisCmd中，调用DoCmd，如果返回正常，则等待Follower确认，再执行DoCommand。如果返回异常，直接标记当前子命令异常，并且resp_num 自减，如果是PikaClientConn 是单条命令的情况，这时候调用BatchExecRedisCmd中的TryWriteResp 就可以直接返回客户端了，没有必要同步到从。

```c++
void PikaClientConn::DoExecTask(void* arg) {
  cmd_ptr->Execute();
  *resp_ptr = std::move(cmd_ptr->res().message());
  conn_ptr->resp_num--;
  conn_ptr->TryWriteResp();
}

void PikaClientConn::TryWriteResp() {
  int expected = 0;
  if (resp_num.compare_exchange_strong(expected, -1)) {
    for (auto& resp : resp_array) {
      WriteResp(std::move(*resp));
    }
    resp_array.clear();
    NotifyEpoll(true);
  }
}
```

1，如果DoCmd返回正常，一致性模块会最终对于每一条子命令调用一次DoExecTask。

2，每次处理子命令都尝试TryWriteResp，只有当前resp_num 是0 才可以整合PikaClientConn中所有子命令执行结果，通知WorkerThread写回客户端。



<br>

### Reference

[https://github.com/Qihoo360/pika/tree/v3.3.4](https://github.com/Qihoo360/pika/tree/v3.3.4)
