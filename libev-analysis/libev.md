# node-0.0.1源码中的libev

#### 目录结构
.
├── ev++.h                // libev simple C++ wrapper classes
├── ev.c                  // libev event processing core, watcher management
├── ev.h                  // libev native API header
├── ev_epoll.c            // libev epoll fd activity backend
├── ev_kqueue.c           // libev kqueue backend
├── ev_poll.c             // libev poll fd activity backend
├── ev_port.c             // libev solaris event port backend
├── ev_select.c           // libev select fd activity backend
├── ev_vars.h             // loop member variable declarations
├── ev_win32.c            // libev win32 compatibility cruft (_not_ a backend)
├── ev_wrap.h
├── event.c               // libevent compatibility layer
├── event.h               // libevent compatibility header, only core events supported
└── event_compat.h


#### ev_loop的启动入口
ev.c:  void ev_loop (EV_P_ int flags);

调用路径:
```mermaid
graph TD
ev_loop["ev_loop()"] --> call_pending["call_pending()"] --"(1)"--> ev_loop_verify["ev_loop_verify()"] --"(1)"--> postfork{postfork};
postfork --yes--> queue_events["queue_events()"] --"(1)"--> call_pending["call_pending()"];

ev_loop_verify --"(1)"--> preparecnt{preparecnt} --yes--> queue_events --"(1)"--> call_pending;

postfork --no--> loop_fork["loop_fork()"];
call_pending --"(2)"--> fd_reify["fd_reify()"]

fd_reify --> backend_poll["backend_poll()"] --> timers_reify["timers_reify()"] --> idle_reify["idle_reify()"] --"(2)"--> call_pending
```

从上面看，call_pending()是一个核心的函数。
下面看下call_pending()的逻辑

```C
void static inline
call_pending (struct ev_loop *loop)
{
  int pri;
  // 看起来是5个优先级队列
  for (pri = 5; pri--; )
    // 每个优先级队列遍历一遍
    while (((loop)->pendingcnt) [pri])
      {
        // 从优先级队列中取出了一个loop实例。取的方法没看明白，和优先级队列的组织结构有关
        ANPENDING *p = ((loop)->pendings) [pri] + --((loop)->pendingcnt) [pri];
        if (__builtin_expect (((p->w) != 0),(1)))
          {
            p->w->pending = 0;
            (p->w)->cb (loop, (p->w), (p->events));
          }
      }
}

```

从上面看，ev_loop的每个tick会经历几个重要的流程/阶段:
- 调用queue_events()检查loop->pending队列
- 调用backend_poll()检查epoll各个fd的结果
- 调用timers_reify()检查各个timer
- 调用idle_reify()执行同步代码


接下来先看下queue_events(),了解下loop->pending队列的管理:
1. queue_events()往loop->pending添加event: 调用ev_feed_event(struct ev_loop*, void* w, int revents)
添加的是void* w, 一个ev_watcher* 类型
2. 


#### 主要数据结构

```mermaid
classDiagram

  ev_watcher <|-- ev_io
  ev_watcher <|-- ev_timer
  ev_watcher <|-- ev_watcher_time
  
  ev_io *-- ev_once
  ev_timer *-- ev_once

  ev_io --> ev_watcher_list
  ev_watcher_list --> ev_watcher_list

  class ev_io {
    object ev_watcher
    ev_watcher_list* next
    fd
    events
  }

  class ev_timer {
    object ev_watcher

    timestamp at
    timestamp repeat
  }

  class ev_watcher_list {
    object ev_watcher
    ev_watcher_list* next
  }

  class ev_once {
    function cb
    void* arg
  }

  class ev_watcher {
    active
    pending
    priority
    void* data
    function cb()
  }


  class ev_watcher_time {
    object ev_watcher
    timestamp at
  }
 

  class ev_loop {
    timestamp ev_rt_now
    timestamp now_floor
    timestamp mn_now;
    timestamp rtmn_diff;
    timestamp io_blocktime;
    timestamp timeout_blocktime;

    int backend
    int activecnt

    int loop_count
    int backend_fd
    timestamp backend_fudge

    backend_modify()
    bakcend_poll()
    int[2] evpipe

    pid curpid
    void* vec_ri
    void* vec_ro
    void* vec_wi
    void* vec_wo
    int vec_max

    int pollmax
    int pollcnt
    int* pollidxs
    int pollidxmax
    int kqueue_changemax;
    int kqueue_changecnt;
    int kqueue_eventmax;

    int[] pendingmax;
    int[] pendingcnt;

    int * fdchanges;
    int fdchangemax;
    int fdchangecnt; 
    ...
  }


  ev_loop --* ev_io
  ev_loop --* pollfd
  ev_loop --* kevent
  ev_loop --* ANFD
  ev_loop --* ANPENDING
  ev_loop --* ANHE
  ev_loop --* ev_idle
  ev_loop --* ev_prepare
  ev_loop --* ev_check
  ev_loop --* ev_fork

  ev_watcher <--> ev_idle
  ev_watcher <--> ev_prepare
  ev_watcher <--> ev_fork

```

#### 关键过程分析

```mermaid
graph TD
  evOnce["ev_once()"] --> newEvOnce["new ev_once()"] --> initEvOnce["ev_once.init()"]

  initEvOnce --> fdJudge["fd>=0?"] --yes--> evIOStart["ev_io_start()"]
  initEvOnce --> timeoutJudge["timeout>=0?"] --yes--> evTimerStart["ev_timer_start(loop, w)"]


  evTimerStart --> evStart["ev_start()"] --> unheap["unheap(loop->timers)"]

evIOStart --> evStart2["ev_start()"] --> wListAdd["wlist_add()"] --> fdChange["fd_change()"]

```

```mermaid
graph TD
evStart["ev_start()"] --pri_adjust--> priAdjust["set w.pri in [-1, 1]"] --> activeW["w->active=active"] --> evRef["ev_ref(loop)"]
```