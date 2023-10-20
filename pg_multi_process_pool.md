# 基于bgworker 实现多进程池

pg 中使用多进程而不是多线程实现并发。如果想启动新进程处理 DB 中的数据或者做一些其它的任务，可以自己写代码 + bgworker 实现。

Bgworker 的介绍参考[Postgres background worker](bgworker.md)

## 怎么启动个进程

### launcher 

动态启动一个 bgworker 的函数是 RegisterDynamicBackgroundWorker， 但是需要有一个守护进程可以接受用户的命令或者根据其它的逻辑，比如时间间隔等来调用该函数启动新的进程。我们把守护进程叫做 launcher。

那 launcher 是怎么启动的呢？

- 把 launcher 的 package 加到 shared_preload_libraries

- 在动态库的 "_PG_init()" 函数中调用 RegisterBackgroundWorker(BackgroundWorker *worker) 进行注册 launcher 进程

Pg在启动的时候会处理 shared_preload_libraries， 加载对应的库并执行 "_PG_init()" 函数中调用来启动launcher。

### worker

#### 启动一个 worker

如果进程需要连接数据库处理数据，每个进程只能连接一个数据库。数据库的 catalog 是相互隔离的, catalog 的 cache 是每个进程私有的。

我们看一下 diskquota 中 launcher 启动 一个 worker 的例子

```
    BackgroundWorker      worker;

    // 初始化 BackgroundWorker
    memset(&worker, 0, sizeof(BackgroundWorker));
    worker.bgw_flags      = BGWORKER_SHMEM_ACCESS | BGWORKER_BACKEND_DATABASE_CONNECTION;
    worker.bgw_start_time = BgWorkerStart_RecoveryFinished;
    worker.bgw_restart_time = BGW_NEVER_RESTART;
    sprintf(worker.bgw_library_name, DISKQUOTA_BINARY_NAME);
    sprintf(worker.bgw_function_name, "disk_quota_worker_main");
    snprintf(worker.bgw_name, sizeof(worker.bgw_name), "diskquota bgworker %d",dibd);

    // MyProcPid 是 launcher 进程的 pid
    worker.bgw_notify_pid = MyProcPid;
    worker.bgw_main_arg   = (Datum)PointerGetDatum(dq_worker);

    //调用RegisterDynamicBackgroundWorker
    old_ctx = MemoryContextSwitchTo(TopMemoryContext);
    ret     = RegisterDynamicBackgroundWorker(&worker, &(bgworker_handles[dq_worker->id]));
    MemoryContextSwitchTo(old_ctx);
    if (!ret)
    {
        elog(WARNING, "Create bgworker failed");
        result = UNKNOWN;
        goto Failed;
    }
    BgwHandleStatus status;
    pid_t           pid;

    // bgworker 进程是 postmaster 异步启动的，等待进程启动
    status = WaitForBackgroundWorkerStartup(bgworker_handles[dq_worker->id], &pid);
    if (status == BGWH_STOPPED)
    {
        ereport(WARNING, (errcode(ERRCODE_INSUFFICIENT_RESOURCES), errmsg("could not start background process"),
        errhint("More details may be available in the server log.")));
        result = UNKNOWN;
        goto Failed;
    }
    if (status == BGWH_POSTMASTER_DIED)
    {
        ereport(WARNING, (errcode(ERRCODE_INSUFFICIENT_RESOURCES),
        errmsg("cannot start background processes without postmaster"),
        errhint("Kill all remaining database processes and restart the database.")));
        result = UNKNOWN;
        goto Failed;
    }
    
    Assert(status == BGWH_STARTED);
```

#### 停掉一个 worker

- worker 进程自然结束.
- launcher 进程调用 TerminateBackgroundWorker(handle).

如果需要在worker 结束的时候做一些清理工作，可以调用 on_shmem_exit:
```
on_shmem_exit(FreeWorkerOnExit, 0);
```
当bgworker 结束的时候，会调用 FreeWorkerOnExit 做一些清理工作，或者设置一些 flags.

## 实现线程池

有时候我们不仅要启动多个进程完成并发任务，进程运行过程中需要维护一些信息，比如上面提到的 handle, pid 等
而且需要控制进程的数量，这时候往往需要一个进程池

- pool_worker : 存储对应的 workerId, dbid, start_time, handle 等，还有 “dlist_node node” 把node 串成双向链表。
- pool 数组，大小由最大进程数量决定，而且需要放到shared memory 里面，因为launcher 和 worker 都需要访问
  它。
  free_worker_list 和 running_worker_list 两个头指针
- 锁，用来维护 pool 的并发访问

launcher 读 pool 的信息，找到可用worker，启动 bgworker, 并记录相关信息到worker里面。

worker 结束的时候，通过 “FreeWorkerOnExit” 函数，设置 worker 的状态为free。 

launcher 只能收到 postmaster 发来的 signal 知道有 worker 进程结束了，但并不知道哪个 worker（database) 结束了，
需读 worker pool 得到这些信息。

如果应用程序特别简单，可以使用一个进程计数器来表示在运行的进程，来控制进程的数量，但是往往大多数我们需要知道一些运行状态
及维护运行的时候信息，方便调度及出错处理，也方便以后扩展增加新的功能。
