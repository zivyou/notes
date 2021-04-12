
#### 主要数据结构
```mermaid
classDiagram
  eio_req <-- eio_req
  etp_worker --> eio_req
  etp_reqq --> eio_req

  class eio_req {
    ptr eio_cb
    data
    next
    grp_next
    grp_prev
    destroy()
    feed()
  }

  class etp_worker {
    prev
    next
    eio_req* req
    dbuf
    dirp
    thread_t tid
  }

  class etp_reqq {
    eio_req[] qs
    eio_req[] qe
  }

```

#### eio_reqq环形队列(req_queue)
```mermaid
graph LR
  start[[eio_req]] --> node2[[eio_req]] --> more["..."]-->start

```

#### 核心运行流程(eio_init)
```mermaid
graph TD
eio_init --启动线程--> pthread_once 
--start routine--> etp_once_init;
pthread_once --set callback-->  want_poll_cb & done_poll_cb;
etp_once_init --"set callback around fork"--> etp_atfork_prepare & etp_atfork_paren & etp_atfork_child

etp_atfork_prepare --加锁--> wrklock,reqlock,reslock,prwlock

etp_atfork_paren --释放锁-->  prwlock,reslock,reqlock,wrklock

etp_atfork_child --清理req_queue队列--> clearReqQueue --清理etp_worker队列--> clearEtpWorker --> etp_atfork_paren
```

#### 核心接口(eio_open为例)
```mermaid
graph TD
eio_open --申请一个eio_req--> newEioReq[new eio_req] --初始化eio_req--> 设置类型 & 设置callback & 设置data & ... --> eio_submit --> etp_submit --> isGroup{是不是EIO_GROUP类型} --no--> count++["nreqs++;npending++"] --> pushIntoReqq[加入reqq队列] --> pollCbExsit{want_poll_cb是否设置了?} --yes--> doCb["want_poll_cb()"]
pollCbExsit --no--> return
isGroup --yes--> count2++["nreqs++;nready++"]-->pushIntoReqq2[加入reqq队列]-->startThread["etp_maybe_start_thread()"]

startThread-->threadAmountCtrl{线程池满了?} --yes-->return
threadAmountCtrl --no--> etpStartThread["etp_start_thread()"]
etpStartThread --> newWrk["new etp_worker()"] --> createThread["thread_create(thread_create (&wrk->tid, etp_proc, (void *)wrk))"] --> createOK["线程创建成功?"] --no--> return
createOK--yes-->initWrk["init wrk; started++"]-->return
```

#### 核心运行流程(etp_proc: 所有线程的routine)

```mermaid
graph TD
etp_proc --锁住reqlock--> reqLock["mutex_lock(reqlock)"] --获取队头req--> reqShift["req=reqq_shifft()"] --> ... --> eio_execute

```

#### 核心运行流程(eio_execute)
```mermaid
graph TD

eio_execute --传入etp_worker,eio_req--> switch{判断req类型} --> idRead[READ] & WRITE & OPEN & ...;
idRead --> id3["POSIX read()"]
````