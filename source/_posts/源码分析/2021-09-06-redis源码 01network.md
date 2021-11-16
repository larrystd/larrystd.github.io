---
title: redis源码 01network
date: 2021-09-06
updated: 2021-09-06
tags: 
  - redis
  - cpp
categories:
  - source code
aplayer: true
---

Redis采用的是基于内存的单进程单线程模型的KV数据库，由C语言编写,被用作缓存和消息中间件。
* redis性能很高, 但在安全性上做了折衷。常用作缓存说明数据库中存有备份数据, redis的少量丢失数据也完全可以从数据库中恢复。
* redis优势是读, leveldb优势是写。redis是内存数据库, leveldb包括内存部分和磁盘部分, LSM tree(Log Structured Merge Tree)是提高写性能的方式, 对标于B+ tree, 读比前者低效率, 写比前者高效率。

* redis的单线程也没那么神,如果条命令执行如果占用大量时间，会造成其他线程阻塞，所以Redis是面向高速执行的数据库。

Redis采用的是服务器-客户端的模式访问数据, 核心源码是服务器, 监听6379端口。任何语言编写的客户端均可通过该端口与server连接。redis server监听事件的方式也是react + eventloop的模型, 然而因为是单线程, 没有实现线程池, 并发控制等功能
。

### 事件循环

#### ae.h

redis的事件循环位于`ae.h`中。

首先用宏定义设置一些常量
```c
#ifndef __AE_H__
#define __AE_H__

#include <time.h>

#define AE_OK 0
#define AE_ERR -1

#define AE_NONE 0           //未设置
#define AE_READABLE 1       //事件可读
#define AE_WRITABLE 2       //事件可写

// 事件类型
#define AE_FILE_EVENTS 1                                //文件事件
#define AE_TIME_EVENTS 2                                //时间事件
#define AE_ALL_EVENTS (AE_FILE_EVENTS|AE_TIME_EVENTS)   //文件和时间事件
#define AE_DONT_WAIT 4
#define AE_NOMORE -1            //定时事件
#define AE_DELETED_EVENT_ID -1  //时间事件被删除的标志
```

文件事件也就是一般监听的fd, 时间事件是也就是定时事件。

用`typedef`设置函数类型
```c
// 文件事件处理 函数类型
typedef void aeFileProc(struct aeEventLoop *eventLoop, int fd, void *clientData, int mask);
// 时间事件处理函数类型
typedef int aeTimeProc(struct aeEventLoop *eventLoop, long long id, void *clientData);
typedef void aeEventFinalizerProc(struct aeEventLoop *eventLoop, void *clientData);
typedef void aeBeforeSleepProc(struct aeEventLoop *eventLoop);
```

<!-- more -->


事件结构体
```c
typedef struct aeFileEvent {
    // 文件时间类型：AE_NONE，AE_READABLE，AE_WRITABLE
    int mask; /* one of AE_(READABLE|WRITABLE) */
    // 可读处理函数
    aeFileProc *rfileProc;
    // 可写处理函数
    aeFileProc *wfileProc;
    // 数据
    void *clientData;
} aeFileEvent;
// 时间事件, 也就是定时器
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    // 时间事件到达的时间的秒数
    long when_sec; /* seconds */
    // 时间事件到达的时间的毫秒数
    long when_ms; /* milliseconds */
    // 时间事件处理函数
    aeTimeProc *timeProc;
    // 时间事件终结函数
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    // 指向下一个时间事件, 时间事件列表用链表存储
    struct aeTimeEvent *next;
} aeTimeEvent;  //时间事件

/// 就绪事件, 由epoll的event_wait产生
typedef struct aeFiredEvent {
    // 就绪事件的文件描述符
    int fd;
    // 就绪事件类型：AE_NONE，AE_READABLE，AE_WRITABLE标注可读事件, 可写事件
    int mask;
} aeFiredEvent; //就绪事件
```

eventloop
```c
/// eventloop主要保存, 注册的events, 就绪的events, 时间序列的头节点
typedef struct aeEventLoop {
    // 当前已注册的最大的文件描述符
    int maxfd;   
    // 设置的fd监听集合的大小
    int setsize; /* max number of file descriptors tracked */
    // 下一个时间事件的ID
    long long timeEventNextId;
    time_t lastTime;     /* Used to detect system clock skew */
    // 注册的文件事件表
    aeFileEvent *events; /* Registered events */
    // 已就绪的文件事件表
    aeFiredEvent *fired; /* Fired events */
    // 时间事件链表的头节点指针
    aeTimeEvent *timeEventHead;
    // 是否进行事件处理
    int stop;
    // polling的事件状态数据
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
} aeEventLoop;  //事件轮询EventLoop
```

具体的执行函数

```c
/* Prototypes */
/// 创建删除eventloop
aeEventLoop *aeCreateEventLoop(int setsize);
void aeDeleteEventLoop(aeEventLoop *eventLoop);
void aeStop(aeEventLoop *eventLoop);
/// 处理file event
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData);
    
void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask);
int aeGetFileEvents(aeEventLoop *eventLoop, int fd);
// timeevent
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc);

int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id);
/// events
int aeProcessEvents(aeEventLoop *eventLoop, int flags);
int aeWait(int fd, int mask, long long milliseconds);
void aeMain(aeEventLoop *eventLoop);
char *aeGetApiName(void);
void aeSetBeforeSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *beforesleep);
/// 设置监听fd的max
int aeGetSetSize(aeEventLoop *eventLoop);
int aeResizeSetSize(aeEventLoop *eventLoop, int setsize);
```

#### ae.c

EventLoop
```c
// 创建并初始化一个EventLoop的状态结构
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    int i;
    // 分配eventloop空间
    if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) goto err;
    // 分配eventloop文件事件表events和就绪事件表fired空间
    // 开始已经分配好FileEvent的最大空间, 直接用
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
    eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);
    
    if (eventLoop->events == NULL || eventLoop->fired == NULL) goto err;
    // 设置一些参数
    eventLoop->setsize = setsize;
    eventLoop->lastTime = time(NULL);
    // 初始化时间事件(链表)
    eventLoop->timeEventHead = NULL;
    eventLoop->timeEventNextId = 0;
    // 事件处理打开
    eventLoop->stop = 0;
    eventLoop->maxfd = -1;
    eventLoop->beforesleep = NULL;

    // 创建一个epoll实例保存到eventLoop的events中
    if (aeApiCreate(eventLoop) == -1) goto err;
    /* Events with mask == AE_NONE are not set. So let's initialize the
     * vector with it. */
    // 初始化监听的事件为空
    for (i = 0; i < setsize; i++)
        eventLoop->events[i].mask = AE_NONE;    // AE_NONE 未设置事件状态
    return eventLoop;

// 删除事件轮询的状态结构
void aeDeleteEventLoop(aeEventLoop *eventLoop) {
    aeApiFree(eventLoop);
    zfree(eventLoop->events);
    zfree(eventLoop->fired);
    zfree(eventLoop);
}
```

FileEvent

```c
// 创建一个文件事件aeFileEvent *fe
// 设置监听fd的事件类型为mask，让事件发生时，则调用proc函数
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    // fe指向 fd对应的文件事件表地址
    aeFileEvent *fe = &eventLoop->events[fd];

    // 监听fd的mask类型事件
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    // 事件类型
    fe->mask |= mask;
    // 设置事件类型对应处理的函数
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    // 设置客户端传入数据
    fe->clientData = clientData;
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}

// 删除一个fd的mask事件，从eventLoop的事件表中
void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask)
{
    if (fd >= eventLoop->setsize) return;
    // fe指向 fd对应的文件事件表地址
    aeFileEvent *fe = &eventLoop->events[fd];
    // fd当前事件为空，则直接返回
    if (fe->mask == AE_NONE) return;

    // 删除fd的mask事件
    aeApiDelEvent(eventLoop, fd, mask);
    // 更新mask事件类型
    fe->mask = fe->mask & (~mask);
    if (fd == eventLoop->maxfd && fe->mask == AE_NONE) {
        /* Update the max fd 更新当前的最大事件描述符 */
        int j;

        for (j = eventLoop->maxfd-1; j >= 0; j--)
        /// 遍历找到目前最大的fd
            if (eventLoop->events[j].mask != AE_NONE) break;
        eventLoop->maxfd = j;
    }
}
```

TimeEvent
```cpp
// 创建一个时间事件节点, aeTimeEvent *te
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc)
{
    // 设置timeEvent的ID
    long long id = eventLoop->timeEventNextId++;

    /// timeEvent结构体
    aeTimeEvent *te;

    // 分配timeEvent的空间
    te = zmalloc(sizeof(*te));
    if (te == NULL) return AE_ERR;
    te->id = id;

    // 设置时间事件处理的时间
    aeAddMillisecondsToNow(milliseconds,&te->when_sec,&te->when_ms);
    // 设置时间事件处理的方法
    te->timeProc = proc;
    // 设置时间事件终结的方法
    te->finalizerProc = finalizerProc;
    // 传入的数据
    te->clientData = clientData;
    // 将当前事件设置为链表的头节点
    te->next = eventLoop->timeEventHead;
    // 指向当前事件
    eventLoop->timeEventHead = te;
    return id;
}

// 删除指定ID的时间事件，惰性删除
int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id)
{
    // 时间事件头节点地址
    aeTimeEvent *te = eventLoop->timeEventHead;
    // 遍历所有节点
    while(te) {
        // 找到应该删除时间事件
        if (te->id == id) {
            /// 设置id为AE_DELETED_EVENT_ID
            te->id = AE_DELETED_EVENT_ID;
            return AE_OK;
        }
        // 指向下一个时间事件地址
        te = te->next;
    }
    return AE_ERR; /* NO event with the specified ID found */
}

// 寻找第一个快到时的时间事件
static aeTimeEvent *aeSearchNearestTimer(aeEventLoop *eventLoop)
{
    // 时间事件头节点地址
    aeTimeEvent *te = eventLoop->timeEventHead;
    aeTimeEvent *nearest = NULL;

    // 遍历所有的时间事件
    while(te) {
        // 顺序遍历,寻找第一个快到时的aeTimeEvent，保存到nearest中
        // 事件复杂度O(N)
        if (!nearest || te->when_sec < nearest->when_sec ||
                (te->when_sec == nearest->when_sec &&
                 te->when_ms < nearest->when_ms))
            nearest = te;
        te = te->next;
    }
    return nearest;
}

// 处理时间事件
static int processTimeEvents(aeEventLoop *eventLoop) {
    int processed = 0;
    aeTimeEvent *te, *prev;
    long long maxId;
    // return current time 当前事件
    time_t now = time(NULL);

    // 尝试发现时间混乱的情况，如果上一次处理事件的时间比当前时间还要大, 重置最近一次处理事件的时间
    if (now < eventLoop->lastTime) {
        te = eventLoop->timeEventHead;
        while(te) {
            /// timeevent到达的秒数为0
            te->when_sec = 0;
            te = te->next;
        }
    }
    // 设置上一次时间事件处理的时间为当前时间
    eventLoop->lastTime = now;

    // 保存上一个节点
    prev = NULL;
    /// 时间事件序列的链表
    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;   //当前时间事件表中的最大ID
    
    // 遍历时间事件链表
    while(te) {
        long now_sec, now_ms;
        long long id;

        /* Remove events scheduled for deletion. */
        // 处理删除的事件, 这里才进行删除, id标识
        if (te->id == AE_DELETED_EVENT_ID) {
            aeTimeEvent *next = te->next;
            // 从事件链表中删除事件的节点
            if (prev == NULL)
                eventLoop->timeEventHead = te->next;
            else
                prev->next = te->next;
            // 终止处理finalizerProc
            if (te->finalizerProc)
                te->finalizerProc(eventLoop, te->clientData);
            zfree(te);
            te = next;
            continue;
        }

        if (te->id > maxId) {
            te = te->next;
            continue;
        }
        // 获取当前时间
        aeGetTime(&now_sec, &now_ms);
        /// 当前时间下，处理已经到达时间的timeevent
        // 当前事件比te->when_sec大
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;

            id = te->id;
            // 调用时间事件处理方法
            retval = te->timeProc(eventLoop, id, te->clientData);
            // 时间事件次数加1
            processed++;
            // 是等间隔执行的事件，继续设置它的到时时间
            if (retval != AE_NOMORE) {
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            // 普通定时事件，则retval为-1，则将其时间事件删除，ID惰性删除
            } else {
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        // 下一个时间事件节点
        prev = te;
        te = te->next;
    }
    return processed;   //返回执行事件的次数
}
```

ProcessEvents
```cpp
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;
    /// flag标注要执行什么类型的事件
    // 即AE_TIME_EVENTS和AE_FILE_EVENTS两种类型事件
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    // 当前还没有要处理的文件事件
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;

        /// 时间间隔
        struct timeval tv, *tvp;

        // 如果设置了时间事件且没设置不阻塞标识
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            // 获取最近到时的时间事件
            shortest = aeSearchNearestTimer(eventLoop);

        if (shortest) {
            long now_sec, now_ms;
            // 获取当前时间
            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;
            // 等待该时间事件到时所需要的时长
            long long ms =
                (shortest->when_sec - now_sec)*1000 +
                shortest->when_ms - now_ms;

            // 如果没到时
            if (ms > 0) {
                // 保存时长到tvp中
                tvp->tv_sec = ms/1000;
                tvp->tv_usec = (ms % 1000)*1000;

            // 如果已经到时，则将tvp的时间设置为0
            } else {
                tvp->tv_sec = 0;
                tvp->tv_usec = 0;
            }

        // 没有获取到了最早到时的时间事件，说明时间事件链表为空
        } else {
            // 如果设置了不阻塞标识
            if (flags & AE_DONT_WAIT) {
                // 将tvp的时间设置为0
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                // 阻塞到第一个时间事件的到来
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }

        // 设置了参数tvp, tvp表示最近的时间事件。如果tvp为NULL，则阻塞直到到来，否则等待tvp设置阻塞的时间，这样执行完文件事件，可以立刻执行时间事件
        // 通过epoll获取就绪的时间事件个数

        numevents = aeApiPoll(eventLoop, tvp);
        // 遍历就绪文件事件表
        for (j = 0; j < numevents; j++) {
            // 获取就绪文件事件的地址
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            // 获取就绪文件事件的类型，文件描述符
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;
            // 如果是文件可读事件发生, 执行读回调fe->rfileProc
            if (fe->mask & mask & AE_READABLE) {
                // 设置读事件标识 且 调用读事件方法处理读事件
                rfired = 1;
                // 可读回调函数
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            // 如果是文件可写事件发生, 执行写回调fe->wfileProc
            if (fe->mask & mask & AE_WRITABLE) {
                // 读写事件的执行发法不同，则执行写事件，避免重复执行相同的方法
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            processed++;    //执行的事件次数加1
        }
    }
    /* Check time events */
    // 执行时间事件, 由于经过epoll的等待，这时最少有一个时间事件可供执行
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}


// 事件轮询的主函数
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    // 一直处理事件
    while (!eventLoop->stop) {
        // 执行beforesleep()
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        //处理到时的时间事件和就绪的文件事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```

#### ae_epoll.c

epoll结构体, 包括监听的epfd和注册的事件表
```cpp
#include <sys/epoll.h>

typedef struct aeApiState {
    // epoll事件的文件描述符
    int epfd;
    // 事件表
    struct epoll_event *events;
} aeApiState;   //事件的状态
```


创建epoll实例
```cpp
// 创建一个epoll实例，保存到eventLoop中
static int aeApiCreate(aeEventLoop *eventLoop) {
    aeApiState *state = zmalloc(sizeof(aeApiState));

    if (!state) return -1;
    // 初始化事件表空间
    state->events = zmalloc(sizeof(struct epoll_event)*eventLoop->setsize);
    if (!state->events) {
        zfree(state);
        return -1;
    }
    // 创建一张事件表，返回一个epfd来标识这张事件表
    // 得到epfd以及epoll内部事件表
    state->epfd = epoll_create(1024); /* 1024 is just a hint for the kernel */
    if (state->epfd == -1) {
        zfree(state->events);
        zfree(state);
        return -1;
    }
    // 保存到事件轮询状态结构中
    eventLoop->apidata = state;
    return 0;
}

// 释放epoll实例和事件表空间
static void aeApiFree(aeEventLoop *eventLoop) {
    aeApiState *state = eventLoop->apidata;

    close(state->epfd);
    zfree(state->events);
    zfree(state);
}
```

epoll注册事件

```cpp
// 在epfd标识的事件表上注册fd的事件
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;
    /// epoll监听的事件
    struct epoll_event ee = {0}; /* avoid valgrind warning */

    // EPOLL_CTL_ADD，向epfd注册fd的上的event
    // EPOLL_CTL_MOD，修改fd已注册的event
    // #define AE_NONE 0           //未设置
    // #define AE_READABLE 1       //事件可读
    // #define AE_WRITABLE 2       //事件可写
    // 判断epoll fd事件的操作，如果没有设置事件，则进行注册事件，否则进行修改
    int op = eventLoop->events[fd].mask == AE_NONE ?
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    // struct epoll_event {
    //      uint32_t     events;      /* Epoll events */
    //      epoll_data_t data;        /* User data variable */
    // };
    ee.events = 0;
    // 合并之前的事件类型
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    // 根据mask映射设置事件类型ee.events
    if (mask & AE_READABLE) ee.events |= EPOLLIN;   //读事件
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;  //写事件
    // fd放在epoll_event.data中
    ee.data.fd = fd;
    
    // 将ee事件注册到epoll中
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    return 0;
}

// 在epfd标识的事件表上注删除fd的事件
static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int delmask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee = {0}; /* avoid valgrind warning */
    // 删除后的事件类型
    int mask = eventLoop->events[fd].mask & (~delmask);

    ee.events = 0;
    // 更新ee.events
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd; 
    // 如果fd的事件不为空事件，则进行修改
    if (mask != AE_NONE) {
        epoll_ctl(state->epfd,EPOLL_CTL_MOD,fd,&ee);
    // 如果fd的事件为空了，就直接进行删除
    } else {
        /* Note, Kernel < 2.6.9 requires a non null event pointer even for
         * EPOLL_CTL_DEL. */
        epoll_ctl(state->epfd,EPOLL_CTL_DEL,fd,&ee);
    }
}
```

polling

```cpp
// 等待所监听文件描述符上有事件发生
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    // 监听事件表上是否有事件发生
    // 事件返回到state->events中
    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    // 至少有一个就绪的事件
    if (retval > 0) {
        int j;

        numevents = retval;
        // 遍历就绪的事件表，将其加入到eventLoop的就绪事件表中
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            // 根据就绪的事件类型，设置事件状态mask
            // EPOLLIN表示可读 -> READ
            if (e->events & EPOLLIN) mask |= AE_READABLE;
            // EPOLLOUT表示可写 -> WRITE
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            // 添加到就绪事件表中
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    // 返回就绪的事件个数
    return numevents;
}
```



### server

* server是主要服务器代码，包含了建立接收连接等过程。
* 对于server来说, 与muduo类似, 对accept后的fd包装成一个client对象, 同时用eventloop注册到epoll中, 并设置可读回调事件。一旦连接可读, 自动执行回调事件将可读数据读取到client对象的buffer缓冲区中。并自动调用处理函数, 处理完毕之后send到fd中。
* client 发来的是命令server自动执行输出结果到缓冲区, 最后刷给client


redis object
```cpp
typedef struct redisObject {
    //对象的数据类型，占4bits，共5种类型
    // REDIS_STRING	字符串对象
    // REDIS_LIST 列表对象
    // REDIS_HASH 哈希对象
    // REDIS_SET 集合对象
    // REDIS_ZSET 有序集合对象, 有序集合关联一个排序的分数, 依据其进行排序
    unsigned type:4;

    //对象的编码，占4bits，共10种类型
    // 编码亦即对象底层使用的数据结果

    // REDIS_ENCODING_INT	long 类型的整数, 字符串对象的编码可以是 int 、 raw 或者 embstr 。
    // REDIS_ENCODING_EMBSTR	embstr 编码的简单动态字符串, 保存短字符串
    // REDIS_ENCODING_RAW	简单动态字符串SDS Simple Dynamic String
    // REDIS_ENCODING_HT	字典
    // REDIS_ENCODING_LINKEDLIST	双端链表
    // REDIS_ENCODING_ZIPLIST	压缩列表(一种连续内存块组成的顺序型（sequential）数据结构)
    // REDIS_ENCODING_INTSET	整数集合 有序、无重复地保存多个整数值, 实现方式其实是array, 查找插入删除复杂度均为O(n)
    // REDIS_ENCODING_SKIPLIST	跳跃表
    unsigned encoding:4;

    unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */

    //对象引用计数
    int refcount;

    //指向底层数据实现的指针
    void *ptr;
} robj;


// 初始化一个在栈中分配的Redis对象
#define initStaticStringObject(_var,_ptr) do { \
    _var.refcount = 1; \
    _var.type = OBJ_STRING; \
    _var.encoding = OBJ_ENCODING_RAW; \
    _var.ptr = _ptr; \
} while(0)


/// 数据库对象
typedef struct redisDb {
    // 键值对dict，保存数据库中所有的键值对
    dict *dict;                 /* The keyspace for this DB */
    // 过期字典，保存着设置过期的键和键的过期时间
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /*Keys with clients waiting for data (BLPOP) */
    // 保存着 处于阻塞状态的键，value为NULL
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    // 事务模块，用于保存被WATCH命令所监控的键
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    struct evictionPoolEntry *eviction_pool;    /* Eviction pool of keys */
    // 数据库ID
    int id;                     /* Database ID */
    // 键的平均过期时间
    long long avg_ttl;          /* Average TTL, just for stats */
} redisDb;
```

跳跃表
```cpp
// 跳跃表插入
// 1、通过查找子函数获取最底层链表中<=num值的最近的节点，赋给prevs
// 2、随机产生该节点的层数
// 跳跃表自然有序
/// 跳跃表节点
typedef struct zskiplistNode {
    robj *obj;                          //保存成员对象的地址
    double score;                       //分值
    struct zskiplistNode *backward;     //后退指针
    
    /// 每层的前进指针
    struct zskiplistLevel {
        struct zskiplistNode *forward;  //前进指针
        unsigned int span;              //跨度
    } level[];                          //层级，柔型数组
} zskiplistNode;

typedef struct zskiplist {
    /// 维护头节点和尾节点
    struct zskiplistNode *header, *tail;//header指向跳跃表的表头节点，tail指向跳跃表的表尾节点
    unsigned long length;       //跳跃表的长度或跳跃表节点数量计数器，除去第一个节点
    int level;                  //跳跃表中节点的最大层数，除了第一个节点
} zskiplist;

typedef struct zset {
    dict *dict;         //字典
    zskiplist *zsl;     //跳跃表
} zset; //有序集合类型
```

```cpp
// 排序对象
typedef struct _redisSortObject {
    robj *obj;
    /// 联合体, 排序标准为score后者 cmp
    union {
        double score;
        robj *cmpobj;
    } u;
} redisSortObject;


// redis命令对象
struct redisCommand {
    char *name;
    // proc函数指针，指向返回值为void，参数为client *c的函数
    redisCommandProc *proc;
    int arity;
    char *sflags; /* Flags as string representation, one char per flag. */
    int flags;    /* The actual flags, obtained from the 'sflags' field. */
    /* Use a function to determine keys arguments in a command line.
     * Used for Redis Cluster redirect. */
    // getkeys_proc是函数指针，返回值是有个整型的数组
    // 从命令行判断该命令的参数
    redisGetKeysProc *getkeys_proc;
    /* What keys should be loaded in background when calling this command? */
    // 指定哪些参数是key
    int firstkey; /* 第一个参数是 key The first argument that's a key (0 = no keys) */
    int lastkey;  /* 最后一个参数是 key The last argument that's a key */
    int keystep;  /* 第一个参数和最后一个参数的步长  The step between first and last key */
    // microseconds记录执行命令的耗费总时长
    // calls记录命令被执行的总次数
    long long microseconds, calls;
};

// 命令处理程序
/// typedef redisCommandProc函数类型, redisCommandProc是一个(void)(client* c)的函数类型
typedef void redisCommandProc(client *c);
typedef int *redisGetKeysProc(struct redisCommand *cmd, robj **argv, int argc, int *numkeys);

/// redis server对象
struct redisServer {
    /* General ××××××××××××××××××××××××××××××××××××××××××××××××××××××××××××××××××××××××××××*/
    pid_t pid;                  /* Main process pid. */
    // 配置文件的绝对路径
    char *configfile;           /* Absolute config file path, or NULL */
    // 可执行文件的绝对路径
    char *executable;           /* Absolute executable file path. */
    // 执行executable文件的参数
    char **exec_argv;           /* Executable argv vector (copy). */
    // serverCron()调用的频率
    int hz;                     /* serverCron() calls frequency in hertz */

    // 数据库数组，长度为16
    redisDb *db;
    // 命令表
    dict *commands;             /* Command table */
    
    // 事件循环
    aeEventLoop *el;
    ...
```


#### server.c
server对象, 是redis服务器的核心。
```cpp
// redisCommand数组
/// name, 实现函数, 参数个数...
/// redisCommand是一个类型, 内部redisCommandProc是执行函数类型
/// typedef void redisCommandProc(client *c);
/// getCommand这些都是该函数类型的函数

struct redisCommand redisCommandTable[] = {
    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
    {"setnx",setnxCommand,3,"wmF",0,NULL,1,1,1,0,0},
    ...

// 初始化服务器
void initServer(void) {
    int j;

    // 忽略SIGHUP和SIGPIPE信号
    signal(SIGHUP, SIG_IGN);
    signal(SIGPIPE, SIG_IGN);
    // 设置信号处理函数
    setupSignalHandlers();

    // 开启系统日志
    if (server.syslog_enabled) {
        openlog(server.syslog_ident, LOG_PID | LOG_NDELAY | LOG_NOWAIT,
            server.syslog_facility);
    }

    // 初始化并创建数据结构
    server.pid = getpid();
    server.current_client = NULL;
    server.clients = listCreate();
    server.clients_to_close = listCreate();
    server.slaves = listCreate();
    server.monitors = listCreate();
    server.clients_pending_write = listCreate();
    server.slaveseldb = -1; /* Force to emit the first SELECT command. */
    server.unblocked_clients = listCreate();
    server.ready_keys = listCreate();
    server.clients_waiting_acks = listCreate();
    server.get_ack_from_slaves = 0;
    server.clients_paused = 0;
    server.system_memory_size = zmalloc_get_memory_size();

    // 创建共享对象
    createSharedObjects();
    // 设置最大客户端连接数量
    adjustOpenFilesLimit();

    
    // 创建事件循环的结构
    server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);
    server.db = zmalloc(sizeof(redisDb)*server.dbnum);

    /* Open the TCP listening socket for the user commands. */
    // 监听端口
    if (server.port != 0 &&
        listenToPort(server.port,server.ipfd,&server.ipfd_count) == C_ERR)
        exit(1);

    /* Open the listening Unix domain socket. */
    // 打开Unix本地端口
    if (server.unixsocket != NULL) {
        unlink(server.unixsocket); /* don't care if this fails */
        server.sofd = anetUnixServer(server.neterr,server.unixsocket,
            server.unixsocketperm, server.tcp_backlog);
        if (server.sofd == ANET_ERR) {
            serverLog(LL_WARNING, "Opening Unix socket: %s", server.neterr);
            exit(1);
        }
        anetNonBlock(NULL,server.sofd);
    }

    /* Abort if there are no listening sockets at all. */
    if (server.ipfd_count == 0 && server.sofd < 0) {
        serverLog(LL_WARNING, "Configured to not listen anywhere, exiting.");
        exit(1);
    }

    /* Create the Redis databases, and initialize other internal state. */
    // 创建并初始化数据库
    for (j = 0; j < server.dbnum; j++) {
        server.db[j].dict = dictCreate(&dbDictType,NULL);
        server.db[j].expires = dictCreate(&keyptrDictType,NULL);
        server.db[j].blocking_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].ready_keys = dictCreate(&setDictType,NULL);
        server.db[j].watched_keys = dictCreate(&keylistDictType,NULL);
        server.db[j].eviction_pool = evictionPoolAlloc();
        server.db[j].id = j;
        server.db[j].avg_ttl = 0;
    }

    ...
        /* Create an event handler for accepting new connections in TCP and Unix
     * domain sockets. */
    // 为每一个TCP连接的client创建文件事件，并安装acceptTcpHandler()函数来accept连接
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
```

监听函数`listenToPort`和接受连接函数`acceptTcpHandler`
```cpp
/// server监听端口, 来自client的连接
// 利用sockfd来监听连接
int listenToPort(int port, int *fds, int *count) {
    int j;

    /* Force binding of 0.0.0.0 if no bind address is specified, always
     * entering the loop if j == 0. */
    // 没有制定绑定的地址，设置为空
    if (server.bindaddr_count == 0) server.bindaddr[0] = NULL;
    
    // 遍历server绑定的地址
    for (j = 0; j < server.bindaddr_count || j == 0; j++) {
        // 没有绑定地址的情况
        if (server.bindaddr[j] == NULL) {
            int unsupported = 0;
            /* Bind * for both IPv6 and IPv4, we enter here only if
             * server.bindaddr_count == 0. */
            // 绑定 IPv6 的所有地址

            // 执行listen accept并返回fd
            // count表示监听到的文件描述符的序号
            fds[*count] = anetTcp6Server(server.neterr,port,NULL,
                server.tcp_backlog);

            if (fds[*count] != ANET_ERR) {
                anetNonBlock(NULL,fds[*count]);
                (*count)++;

            } else if (errno == EAFNOSUPPORT) {
                unsupported++;
                serverLog(LL_WARNING,"Not listening to IPv6: unsupproted");
            }

            // 绑定 IPv4 的所有地址
            if (*count == 1 || unsupported) {
                /* Bind the IPv4 address as well. */
                fds[*count] = anetTcpServer(server.neterr,port,NULL,
                    server.tcp_backlog);
                if (fds[*count] != ANET_ERR) {
                    anetNonBlock(NULL,fds[*count]);
                    (*count)++;
                } else if (errno == EAFNOSUPPORT) {
                    unsupported++;
                    serverLog(LL_WARNING,"Not listening to IPv4: unsupproted");
                }
            }
            /* Exit the loop if we were able to bind * on IPv4 and IPv6,
             * otherwise fds[*count] will be ANET_ERR and we'll print an
             * error and return to the caller with an error. */
            if (*count + unsupported == 2) break;
        // 绑定 IPv6 的指定的地址
        } else if (strchr(server.bindaddr[j],':')) {
            /* Bind IPv6 address. */
            fds[*count] = anetTcp6Server(server.neterr,port,server.bindaddr[j],
                server.tcp_backlog);
        // 绑定 IPv4 的指定的地
        } else {
            /* Bind IPv4 address. */
            fds[*count] = anetTcpServer(server.neterr,port,server.bindaddr[j],
                server.tcp_backlog);
        }
        if (fds[*count] == ANET_ERR) {
            serverLog(LL_WARNING,
                "Creating Server TCP listening socket %s:%d: %s",
                server.bindaddr[j] ? server.bindaddr[j] : "*",
                port, server.neterr);
            return C_ERR;
        }
        
        // 设置fd为非阻塞
        anetNonBlock(NULL,fds[*count]);
        (*count)++;
    }
    return C_OK;
}


// 创建一个TCP的连接处理程序
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd, max = MAX_ACCEPTS_PER_CALL; //最大一个处理1000次连接
    char cip[NET_IP_STR_LEN];
    UNUSED(el);
    UNUSED(mask);
    UNUSED(privdata);

    while(max--) {
        // accept接受client的连接
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
        if (cfd == ANET_ERR) {
            if (errno != EWOULDBLOCK)
                serverLog(LL_WARNING,
                    "Accepting client connection: %s", server.neterr);
            return;
        }
        // 打印连接的日志
        serverLog(LL_VERBOSE,"Accepted %s:%d", cip, cport);
        // 创建一个连接状态的client
        acceptCommonHandler(cfd,0,cip);
    }
}

// acceptCommonHandler(cfd,0,cip);函数包含以下内容
// 创建一个文件事件状态el，且监听读事件，开始接受命令的输入
/// 事件可读处理函数是readQueryFromClient
        if (aeCreateFileEvent(server.el,fd,AE_READABLE,
            readQueryFromClient, c) == AE_ERR)
        {
            close(fd);
            zfree(c);
            return NULL;
        }
```

#### 从可读得到命令，到执行命令和写入客户端

* 可读处理函数

client 有两个缓冲区, querybuf是输入缓冲区进行read fd, buf是输出缓冲区把命令执行的结果写入其中。

```cpp
// 读取client的输入缓冲区的内容
/// 作为fd可读的回调事件, 自动将可读的fd数据读入到缓冲区中, 并调用processInputBuffer(c);处理之
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    client *c = (client*) privdata;
    int nread, readlen;
    size_t qblen;
    UNUSED(el);
    UNUSED(mask);

    // 读入的长度，默认16MB
    readlen = PROTO_IOBUF_LEN;
    // 设置读入的长度readlen
    if (c->reqtype == PROTO_REQ_MULTIBULK && c->multibulklen && c->bulklen != -1
        && c->bulklen >= PROTO_MBULK_BIG_ARG)
    {
        int remaining = (unsigned)(c->bulklen+2)-sdslen(c->querybuf);

        if (remaining < readlen) readlen = remaining;
    }

    // 输入缓冲区的长度
    qblen = sdslen(c->querybuf);
    // 更新缓冲区的峰值
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
    // 扩展缓冲区的大小
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    // 将client发来的命令，读入到输入缓冲区中
    nread = read(fd, c->querybuf+qblen, readlen);
    // 读操作出错
    if (nread == -1) {
        if (errno == EAGAIN) {
            return;
        } else {
            serverLog(LL_VERBOSE, "Reading from client: %s",strerror(errno));
            freeClient(c);
            return;
        }
    // 读操作完成
    } else if (nread == 0) {
        serverLog(LL_VERBOSE, "Client closed connection");
        freeClient(c);
        return;
    }
    // 更新输入缓冲区的已用大小和未用大小。
    sdsIncrLen(c->querybuf,nread);
    // 设置最后一次服务器和client交互的时间
    c->lastinteraction = server.unixtime;
    // 如果是主节点，则更新复制操作的偏移量
    if (c->flags & CLIENT_MASTER) c->reploff += nread;
    // 更新从网络输入的字节数
    server.stat_net_input_bytes += nread;
    ...


    // 处理client输入的命令内容
    processInputBuffer(c);
}

// 处理client输入的命令内容
void processInputBuffer(client *c) {
    server.current_client = c;

    // 一直读输入缓冲区的内容
    while(sdslen(c->querybuf)) {
        /* Return if clients are paused. */
        // 如果处于暂停状态，直接返回
        if (!(c->flags & CLIENT_SLAVE) && clientsArePaused()) break;

        /* Immediately abort if the client is in the middle of something. */
        // 如果client处于被阻塞状态，直接返回
        if (c->flags & CLIENT_BLOCKED) break;
        // 如果client处于关闭状态，则直接返回
        if (c->flags & (CLIENT_CLOSE_AFTER_REPLY|CLIENT_CLOSE_ASAP)) break;

        ...

        /* Multibulk processing could see a <= 0 length. */
        // 如果参数为0，则重置client
        if (c->argc == 0) {
            resetClient(c);
            // 执行命令成功后重置client
            /// 执行发来的命令
            /// 核心 processCommand(c)
            if (processCommand(c) == C_OK)
                resetClient(c);
            if (server.current_client == NULL) break;
        }
    }
    // 执行成功，则将用于崩溃报告的client设置为NULL
    server.current_client = NULL;
}

int processCommand(client *c) {

    // 如果是 quit 命令，则单独处理
    if (!strcasecmp(c->argv[0]->ptr,"quit")) {
        addReply(c,shared.ok);
        c->flags |= CLIENT_CLOSE_AFTER_REPLY;   //设置client的状态为回复后立即关闭，返回C_ERR
        return C_ERR;
    }
    ...
    /* Exec the command */
    // 执行命令
    // client处于事务环境中，但是执行命令不是exec、discard、multi和watch
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        // 除了上述的四个命令，其他的命令添加到事务队列中
        queueMultiCommand(c);
        addReply(c,shared.queued);
    // 执行普通的命令
    } else {
        /// 核心 调用call函数
        call(c,CMD_CALL_FULL);
        // 保存写全局的复制偏移量
        c->woff = server.master_repl_offset;
        // 如果因为BLPOP而阻塞的命令已经准备好，则处理client的阻塞状态
        if (listLength(server.ready_keys))
            handleClientsBlockedOnLists();
    }
    return C_OK;
}

//call
void call(client *c, int flags) {
    long long dirty, start, duration;
    int client_old_flags = c->flags;    //备份client的flags

    /* Sent the command to clients in MONITOR mode, only if the commands are
     * not generated from reading an AOF. */
    // 将命令发送给 MONITOR
    if (listLength(server.monitors) &&
        !server.loading &&
        !(c->cmd->flags & (CMD_SKIP_MONITOR|CMD_ADMIN)))
    {
        replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
    }

    // 初始化Redis操作数组，用来追加命令的传播
    redisOpArrayInit(&server.also_propagate);

    /* Call the command. */
    // 备份脏键数
    dirty = server.dirty;
    // 获取执行命令的开始时间
    start = ustime();
    // 核心, 调用proc(c)执行命令
    c->cmd->proc(c);
    // 命令的执行时间
    duration = ustime()-start;
    // 命令修改的键的个数
    dirty = server.dirty-dirty;
    if (dirty < 0) dirty = 0;
    ... 
```

* 调用c->cmd->proc(c);执行命令之后都会把结果放到回复缓冲区中。而将输出缓冲区数据写回客户端在eventloop中

get命令执行过程

```cpp
//调用getGenericCommand实现GET命令
void getCommand(client *c) {
    getGenericCommand(c);
}

//GET 命令的底层实现
int getGenericCommand(client *c) {
    robj *o;

    //lookupKeyReadOrReply函数是为执行读操作而返回key的值对象，找到返回该对象，找不到会发送信息给client
    //如果key不存在直接，返回0表示GET命令执行成功
    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.nullbulk)) == NULL)
        return C_OK;

    //如果key的值的编码类型不是字符串对象
    if (o->type != OBJ_STRING) {
        addReply(c,shared.wrongtypeerr);    //返回类型错误的信息给client，返回-1表示GET命令执行失败
        return C_ERR;
    } else {
        addReplyBulk(c,o);  //返回之前找到的对象作为回复给client，返回0表示GET命令执行成功
        return C_OK;
    }
}


// 添加obj到client的回复缓冲区中
void addReply(client *c, robj *obj) {
    // 准备client为可写的
    if (prepareClientToWrite(c) != C_OK) return;
    /// 编码对象
    if (sdsEncodedObject(obj)) {
        // 如果固定的回复缓冲区空间不足够，则添加到回复链表中，可能引起内存分配
        if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != C_OK)
            _addReplyObjectToList(c,obj);
    // 如果是int编码的对象
    } else if (obj->encoding == OBJ_ENCODING_INT) {

        if (listLength(c->reply) == 0 && (sizeof(c->buf) - c->bufpos) >= 32) {
            char buf[32];
            int len;
            // 转换为字符串
            len = ll2string(buf,sizeof(buf),(long)obj->ptr);
            
            // 将字符串添加到client的buf中
            if (_addReplyToBuffer(c,buf,len) == C_OK)
                return;
        }
        // 当前对象是整数，但是长度大于32位，则解码成字符串对象
        obj = getDecodedObject(obj);
        // 添加字符串对象的值到固定的回复buf中
        if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != C_OK)
            // 如果添加失败，则保存到回复链表中
            _addReplyObjectToList(c,obj);
        decrRefCount(obj);
    } else {
        serverPanic("Wrong obj->encoding in addReply()");
    }
}


// 将字符串s添加到固定回复缓冲区c->buf中
int _addReplyToBuffer(client *c, const char *s, size_t len) {

    size_t available = sizeof(c->buf)-c->bufpos;

    // 如果client即将关闭，则直接成功返回
    if (c->flags & CLIENT_CLOSE_AFTER_REPLY) return C_OK;

    /* If there already are entries in the reply list, we cannot
     * add anything more to the static buffer. */
    if (listLength(c->reply) > 0) return C_ERR;

    // 检查空间大小是否满足
    if (len > available) return C_ERR;

    // 将s拷贝到client的buf中，并更新buf的偏移量
    memcpy(c->buf+c->bufpos, s,len);
    c->bufpos+=len;
    return C_OK;
}
```

* 写回到客户端, 是在eventloop前的beforesleep函数中,这样将同一个客户端上的多个命令的多次返回，对多个命令做一个缓存，最终一次性统一返回，减少了返回的次数，提高了性能。

```cpp
// 事件轮询的主函数
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    // 一直处理事件
    while (!eventLoop->stop) {
        // 处理事件之前的函数, 作用是写回到客户端
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        //处理到时的时间事件和就绪的文件事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}


// 在Redis进入事件循环之前被调用
void beforeSleep(struct aeEventLoop *eventLoop) {
    UNUSED(eventLoop);
    ...

    /* Try to process pending commands for clients that were just unblocked. */
    // 处理所有非阻塞的client的输入缓冲区的内容
    if (listLength(server.unblocked_clients))
        processUnblockedClients();

    /* Write the AOF buffer on disk */
    // 将AOF缓存冲洗到磁盘中
    flushAppendOnlyFile(0);

    /* Handle writes with pending output buffers. */
    // 处理放在clients_pending_write链表中的待写的client，将输出缓冲区的内容写到fd中
    handleClientsWithPendingWrites();
}

// 这个函数是在进入事件循环之前调用的，希望我们只需要将回复写入客户端输出缓冲区
int handleClientsWithPendingWrites(void) {
    listIter li;
    listNode *ln;
    // 要写或者安装写处理程序的client链表的长度
    int processed = listLength(server.clients_pending_write);
    // 设置遍历方向
    listRewind(server.clients_pending_write,&li);
    // 遍历链表
    while((ln = listNext(&li))) {
        // 取出当前client
        client *c = listNodeValue(ln);
        // 删除client的 要写或者安装写处理程序 的标志
        c->flags &= ~CLIENT_PENDING_WRITE;
        // 从要写或者安装写处理程序的client链表中删除
        listDelNode(server.clients_pending_write,ln);

        /* Try to write buffers to the client socket. */
        // 将client的回复数据发送给client，但是不会删除fd的可读事件
        if (writeToClient(c->fd,c,0) == C_ERR) continue;

        /* If there is nothing left, do nothing. Otherwise install
         * the write handler. */

        if (clientHasPendingReplies(c) &&
            aeCreateFileEvent(server.el, c->fd, AE_WRITABLE,
                sendReplyToClient, c) == AE_ERR)
        {
            freeClientAsync(c);
        }
    }
    // 返回处理的client的个数
    return processed;
}


// 将输出缓冲区的数据写给client，如果client被释放则返回C_ERR，没被释放则返回C_OK
int writeToClient(int fd, client *c, int handler_installed) {
    ssize_t nwritten = 0, totwritten = 0;
    size_t objlen;
    size_t objmem;
    robj *o;

    // 如果指定的client的回复缓冲区中还有数据，则返回真，表示可以写socket
    while(clientHasPendingReplies(c)) {
        // 固定缓冲区发送未完成
        if (c->bufpos > 0) {
            // 将输出缓冲区buf的数据写到fd中
            nwritten = write(fd,c->buf+c->sentlen,c->bufpos-c->sentlen);
            // 写失败跳出循环
            if (nwritten <= 0) break;
            // 更新发送的数据计数器
            c->sentlen += nwritten;
            totwritten += nwritten;

    ...
```