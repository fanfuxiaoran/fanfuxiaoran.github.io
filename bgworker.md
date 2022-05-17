# 有趣的bgwoker

## bgworker 简介

PostgreSQL提供了一套frame使postgres可以启动单独后台进程跑用户提供的代码，被称为backgroud worker，简称bgworker。
那bgworker和普通进程有什么区别吗？

1. attach to PostgreSQL's shared memory area

2. connect to databases internally

根据bgwoker上面的两个特性，用户可以在bgwoker中通过SPI直接连数据库，或通过函数直接访问数据库中的数据，无需通过网络连接连接到数据库，灵活性和效率都极大提高。

## 怎么实现一个bgwoker?
参考 :https://www.mytecdb.com/blogDetail.php?id=220 

补充：
有两个注册bgworker的入口

1) RegisterBackgroundWorker

2) RegisterDynamicBackgroudWorker

- RegisterBackgroundWorker 需要在_PG_init() 中调用(_PG_init()在加载外部库时会默认调用), 其所在的外部库需要注册到shared_preload_libraries.
  整个流程: postmaster在启动的时候会加载shared_preload_libraries里面指定的外部库，调用_PG_init, 调用RegisterBackgroundWorker注册启动bgworker.
  这类bgworker和postmaster一起启动

- RegisterDynamicBackgroudWorker，当postgers 启动之后，用户也可以在其它进程中，例如通过RegisterBackgroundWorker中启动的bgworker中调用
  RegisterDynamicBackgroudWorker 动态注册启动新的bgworker.

## 源代码解读

### 重要的数据结构

- BackgroundWorkerList 一个 RegisteredBgWorker的list,记录bgworker的列表

  BackgroundWorkerList 是postmaster进程私有数据，其他进程看不到。

  BackgroundWorker 主要记录bgworker函数入口，flags, bgworker 重启策略等
  RegisteredBgWorker, 在BackgroundWorker外面包一层，记录其对应的pid,状态等
 
```
typedef struct RegisteredBgWorker
{

	BackgroundWorker rw_worker; /* its registry entry */
	
	struct bkend *rw_backend;       /* its BackendList entry, or NULL */
	
	pid_t           rw_pid;                 /* 0 if not running */
	
	int                     rw_child_slot;
	
	TimestampTz rw_crashed_at;      /* if not 0, time it last crashed */
	
	int                     rw_shmem_slot;
	
	bool            rw_terminate;
	
	slist_node      rw_lnode;               /* list link */

} RegisteredBgWorker;
```
  
- BackgroundWorkerArray
  和BackgroundWorkerList类似，是一个列表，但在Shared memory中,它的node是BackgroundWorkerSlot

  BackgroundWorkerSlot 同样是对BackgroundWorker的包装，记录pid以及是否terminate等信息

```
typedef struct BackgroundWorkerArray 
{ 

        int                     total_slots; 

        BackgroundWorkerSlot slot[FLEXIBLE_ARRAY_MEMBER]; 

} BackgroundWorkerArray; 

typedef struct BackgroundWorkerSlot
{
        bool            in_use;
        bool            terminate;
        pid_t           pid;                    /* InvalidPid = not started yet; 0 = dead */
        uint64          generation;             /* incremented when slot is recycled */
        BackgroundWorker worker;
} BackgroundWorkerSlot;
```
 
BackgroundWorkerList 和 BackgroundWorkerArray 看着很类似，为什么要使用两个类似的数据结构呢？

两者最主要的区别是前者是postmaster private的数据， 后者是在shared memory 中共享的数据。
postmaster进程通过读取BackgroundWorkerList, 来启动bgworker进程。而外部则是通过函数操作
shared memory中的BackgroundWorkerArray 新增、删除、改变bgworker。
postmaster进程会观察BackgroundWorkerArray中的变化，并相应的改变其private的BackgroundWorkerList.

这样做的好处
- 可以防止外部进程对shared memory中的东西乱改，导致postmaster崩溃。
- 隔离bgwoker提供的外部接口功能和postmaster启动并管理bgworker的核心实现（此处比较抽象，可通过阅读下面部分了解）

### 重要流程

- 创建bgworker
  1. 调用 RegisterDynamicBackgroupWorker

  把worker放到BackgroundWorkerArray, 给postmaster进程发PMSIGNAL_BACKGROUND_WORKER_CHANGE 信号

  2. postmaster收到信号，调用BackgroundWorkerStateChange 函数,把worker拷贝到BackgroundWorkerList
  
  3. postmaster进程每次loop遍历BackgroundWorkerList，对于新注册bgwoker创建新进程并启动bgworker
 
- 终止bgworker
  1. 调用TerminateBackgroundWorker, 把对应的slot的terminate设置为false,给postmaster进程发PMSIGNAL_BACKGROUND_WORKER_CHANGE 信号

  2. postmaster收到信号，调用BackgroundWorkerStateChange 函数,kill bgworker进程, 把worker terminate 设置为true
  
  3. postmaster进程每次loop遍历BackgroundWorkerList，对于terminate为true的bgworker, 调用ForgetBackgroundWorker,
  从BackgroundWorkerList中删除，并回收slot.

BackgroundWorkerStateChange是一个重要函数，postmaster收到PMSIGNAL_BACKGROUND_WORKER_CHANGE后，调用该函数, 主要功能如上所述。

由上面的流程可以看出，外部进程主要和BackgroundWorkerArray交互，实现创建、终止bgworker等功能, postmaster主要操作BackgroundWorkerList.
以后想扩展bgworker的功能，只需在bgworker.c中，基于BackgroundWorkerArray实现相应函数即可，postmaster的主要逻辑不变。

### 其它的一些功能
1. BackgroundWorker 中的 bgw_notify_pid

   如果bgw_notify_pid 设置了pid,则在bgworker启动或终止的时候通知该pid进程
2. BackgroundWorker中可以设置重启策略，如果bgworker crash之后，postmaster进程会在每个loop检查是否需要重启bgworker.



