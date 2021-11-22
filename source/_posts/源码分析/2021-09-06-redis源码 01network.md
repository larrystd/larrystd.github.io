---
title: redis源码 network
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

Redis采用的是服务器-客户端的模式访问数据, 核心源码是服务器, 监听6379端口。任何语言编写的客户端均可通过该端口与server连接。redis server监听事件的方式也是react + eventloop的模型, 然而因为是单线程, 没有实现线程池, 并发控制等功能。


### ae.h

ae.h中定义了一些server用到处理事件的结构体, 例如事件循环, 定时事件, 被poll触发的事件等。同时提供了相关的操作函数


```cpp
// 宏定义一些事件类型等的标志
#define AE_FILE_EVENTS 1
#define AE_TIME_EVENTS 2

struct aeEventLoop;

/* Types and data structures */
typedef void aeFileProc(struct aeEventLoop *eventLoop, int fd, void *clientData, int mask); // 文件操作函数
typedef int aeTimeProc(struct aeEventLoop *eventLoop, long long id, void *clientData); // 定时器回调函数
typedef void aeEventFinalizerProc(struct aeEventLoop *eventLoop, void *clientData);
typedef void aeBeforeSleepProc(struct aeEventLoop *eventLoop);

/* File event structure */
typedef struct aeFileEvent {    // 文件事件
    int mask; /* one of AE_(READABLE|WRITABLE|BARRIER) 表示事件的状态*/
    aeFileProc *rfileProc;  // 读文件处理函数
    aeFileProc *wfileProc;  // 写文件处理函数
    void *clientData;
} aeFileEvent;

/* Time event structure 相当于定时器 用链表来维护*/
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    aeTimeProc *timeProc;   // 定时任务
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *prev;
    struct aeTimeEvent *next;
} aeTimeEvent;

/* A fired event */
typedef struct aeFiredEvent { // 被触发的事件
    int fd;
    int mask;
} aeFiredEvent;

/* State of an event based program */
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    time_t lastTime;     /* Used to detect system clock skew */
    // 维护文件处理函数
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead; // 定时器header
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
} aeEventLoop;

/* Prototypes */
aeEventLoop *aeCreateEventLoop(int setsize);    // 创建eventloop
void aeDeleteEventLoop(aeEventLoop *eventLoop);
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData);    // 创建File事件
void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask);
int aeGetFileEvents(aeEventLoop *eventLoop, int fd);
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc);   // 定时事件
int aeWait(int fd, int mask, long long milliseconds);
void aeMain(aeEventLoop *eventLoop);    // 事件循环主函数
```

ae.c, 说明了事件循环整个逻辑的实现。注意epoll_wait的超时时间是定时器最早时间, 这令epoll_wait的阻塞不会影响定时器的执行。

定时器任务有一个id表示对当前定时器的操作, 执行过的定时任务id设置为delete, 这样下次就会把该执行完的定时任务删除了

```cpp
void aeMain(aeEventLoop *eventLoop) {   // 事件执行主函数, 调用aeProcessEvents
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);  // 调用aeProcessEvents
    }
}

int aeProcessEvents(aeEventLoop *eventLoop, int flags)  // 处理事件
{
    int processed = 0, numevents;

    /* Nothing to do? return ASAP */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop); // 找定时器中最近的事件(即定时器最小的事件)
        if (shortest) { // 如果找到了, 设置timeval tv, *tvp的值
            long now_sec, now_ms;

            aeGetTime(&now_sec, &now_ms);   // 得到当前的时间
            tvp = &tv;
        } else {
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }

        /* Call the multiplexing API, will return only on timeout or when
         * some event fires. */
        numevents = aeApiPoll(eventLoop, tvp);  // 调用poll等待触发的事件, tvp是作为超时时间输进去, 保证

        /* After sleep callback. */
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);

        for (j = 0; j < numevents; j++) {   // 对触发的事件进行操作, 执行对应的回调函数
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0; /* Number of events fired for current fd. */

            int invert = fe->mask & AE_BARRIER;

            /* Fire the writable event. */
            if (fe->mask & mask & AE_WRITABLE) {    // 可写事件
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);    // 执行写回调函数
                    fired++;
                }
            }
            /* If we have to invert the call, fire the readable event now
             * after the writable one. */
            if (invert && fe->mask & mask & AE_READABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);    // 执行写回调函数
                    fired++;
                }
            }
            processed++;
        }
    }
    /* Check time events */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);  // 处理定时事件

    return processed; /* return the number of processed file/time events */
}

static aeTimeEvent *aeSearchNearestTimer(aeEventLoop *eventLoop)    // 离当前时间最近的定时器，也就是找时间戳最小的aeTimeEvent节点
{
    aeTimeEvent *te = eventLoop->timeEventHead;
    aeTimeEvent *nearest = NULL;

    while(te) {
        if (!nearest || te->when_sec < nearest->when_sec ||
                (te->when_sec == nearest->when_sec &&
                 te->when_ms < nearest->when_ms))   // 其实就是找定时链表的最小值
            nearest = te;
        te = te->next;
    }
    return nearest;
}

static int processTimeEvents(aeEventLoop *eventLoop) {  // 处理定时任务, 定时链表应该是按照时间从小到大排序的
    int processed = 0;
    aeTimeEvent *te;
    long long maxId;
    time_t now = time(NULL);    // 返回从1970.1.1到现在的秒数

    if (now < eventLoop->lastTime) {    // 一般来说当前时间应该大于eventLoop->lastTime, 如果小于说明一个需要执行了定时任务也没有
        te = eventLoop->timeEventHead;
        while(te) {
            te->when_sec = 0;
            te = te->next;
        }
    }
    eventLoop->lastTime = now;  // 已经执行的最晚时间

    te = eventLoop->timeEventHead;  // 定时链表的头检点
    maxId = eventLoop->timeEventNextId-1;
    while(te) {
        long now_sec, now_ms;
        long long id;

        /* Remove events scheduled for deletion. */
        if (te->id == AE_DELETED_EVENT_ID) {    // 如果这个定时任务需要删除, 显然te->id是对定时任务的操作。如果一个定时器已经执行过, 那么下次它就要被删除
            aeTimeEvent *next = te->next;
            if (te->prev)
                te->prev->next = te->next;
            else
                eventLoop->timeEventHead = te->next;
            if (te->next)
                te->next->prev = te->prev;
            if (te->finalizerProc)
                te->finalizerProc(eventLoop, te->clientData);
            zfree(te);
            te = next;
            continue;
        }

        aeGetTime(&now_sec, &now_ms);   // 得到当前的时间
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms)) // 如果te的时间小于当前时间(说明te需要执行了)
        {
            int retval;

            id = te->id;
            retval = te->timeProc(eventLoop, id, te->clientData);   // 执行定时任务
            processed++;    // 已经执行的个数
            if (retval != AE_NOMORE) {
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
                te->id = AE_DELETED_EVENT_ID;   // 该定时任务需要删除
            }
        }
        te = te->next;
    }
    return processed;
}

int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask, 
        aeFileProc *proc, void *clientData) // 增加一个FileEvent
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    aeFileEvent *fe = &eventLoop->events[fd];

    if (aeApiAddEvent(eventLoop, fd, mask) == -1)   // 调用epoll添加event
        return AE_ERR;
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```
#### ae_epoll.c

epoll结构体, 包括监听的epfd和注册的事件表

```cpp
typedef struct aeApiState {
    int epfd;
    struct epoll_event *events;
} aeApiState;

static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) { // 向epoll注册列表添加事件, fd为文件描述符,mask为事件之类型
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee = {0}; /* avoid valgrind warning */
    /* If the fd was already monitored for some event, we need a MOD
     * operation. Otherwise we need an ADD operation. */
    int op = eventLoop->events[fd].mask == AE_NONE ?
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    ee.events = 0;
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd;
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    return 0;
}

static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) { // 使用epoll来进行IO多路复用
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1); // reactor模式, epoll_wait阻塞到, 超时时间是输入的tvp
    if (retval > 0) {   // 有激活的事件, 设置好eventLoop->fired[]
        int j;

        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    return numevents;   // 返回激活事件的数量
}
```

### server中的网络处理

* server是主要服务器代码，包含了建立接收连接等过程。
* 对于server来说, 与muduo类似, 对accept后的fd包装成一个client对象, 同时用eventloop注册到epoll中, 并设置可读回调事件。一旦连接可读, 自动执行回调事件将可读数据读取到client对象的buffer缓冲区中。并自动调用处理函数, 处理完毕之后send到fd中。
* client 发来的是命令server自动执行输出结果到缓冲区, 最后刷给client


server.c中有`int main()`函数, 是整个redis服务器的入口, 首先初始化server, 设置好监听连接。

```cpp
int main(int argc, char **argv) {
    struct timeval tv;
    int j;
    ..
    initServer();   // 初始化server, 包括监听连接, 相当于server.start()
    ...

void initServer(void) { // 初始化server设置, 监听连接, 接受连接
    int j;

    signal(SIGHUP, SIG_IGN);
    signal(SIGPIPE, SIG_IGN);

    /// 创建定时器事件到eventloop中, 设置好回调函数
    if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        serverPanic("Can't create event loop timers.");
        exit(1);
    }

    /// 创建sockfd(非阻塞socket), 设置sockfd可监听连接
    if (server.port != 0 &&
        listenToPort(server.port,server.ipfd,&server.ipfd_count) == C_ERR)  // 执行监听端口的操作
        exit(1);

    /* Create an event handler for accepting new connections in TCP and Unix
     * domain sockets. */
    /// 将sockfd注册到epoll中, 每次被触发则调用回调函数acceptTcpHandler
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)   // 创建对sockfd创建要监听的文件事件, 用来监听连接, 连接到来调用acceptTcpHandler进行accept
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }

// 随后 server执行loop循环, 如果有连接到来则触发sockfd调用acceptTcpHandler处理
    aeSetBeforeSleepProc(server.el,beforeSleep);
    aeSetAfterSleepProc(server.el,afterSleep);
    aeMain(server.el);  // 执行loop
    aeDeleteEventLoop(server.el);
```

有连接到来则触发sockfd调用acceptTcpHandler处理
```cpp
// 调用anetTcpAccept执行accept返回cfd,
// 调用acceptCommonHandler 处理cfd, 封装成client
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
    char cip[NET_IP_STR_LEN];

    while(max--) {
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);   // 接受到来的连接cfd, 最大max
        ...
        serverLog(LL_VERBOSE,"Accepted %s:%d", cip, cport);
        acceptCommonHandler(cfd,0,cip); // 处理到来的cfd, 封装成client和消息回调
    }
}

// 主要是执行createClient对cfd创建一个client对象
static void acceptCommonHandler(int fd, int flags, char *ip) {
    client *c;
    if ((c = createClient(fd)) == NULL) {   // 对fd创建一个client对象
        serverLog(LL_WARNING,
            "Error registering fd event for the new client: %s (fd=%d)",
            strerror(errno),fd);
        close(fd); /* May be already closed, just ignore errors */
        return;
    }

    if (listLength(server.clients) > server.maxclients) {   // 连接过多的处理
        char *err = "-ERR max number of clients reached\r\n";

        /* That's a best effort error message, don't check write errors */
        if (write(c->fd,err,strlen(err)) == -1) {
            /* Nothing to do, Just to avoid the warning... */
        }
        server.stat_rejected_conn++;
        freeClient(c);
        return;
    }
}

client *createClient(int fd) {  // 根据fd创建client, 其中包含将fd 可读事件注册到epoll事件列表中
    client *c = zmalloc(sizeof(client));
    if (fd != -1) {
        anetNonBlock(NULL,fd);
        anetEnableTcpNoDelay(NULL,fd);
        if (server.tcpkeepalive)
            anetKeepAlive(NULL,fd,server.tcpkeepalive); // 设置keepalive
        // 将cfd注册到epoll中， 回调函数是readQueryFromClient
        if (aeCreateFileEvent(server.el,fd,AE_READABLE,
            readQueryFromClient, c) == AE_ERR)  // 将fd 可读事件注册到epoll事件列表中, 回调函数为readQueryFromClient
        {
            close(fd);
            zfree(c);
            return NULL;
        }
    }
```

当client发消息来, cfd可读, 调用readQueryFromClient处理之。
```cpp
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {   // 用户连接成功后的可读处理事件,接受用户发来的指令
    client *c = (client*) privdata; // 基于fd的用户已经封装成了client
    int nread, readlen;
    size_t qblen;
    UNUSED(el);
    UNUSED(mask);

    readlen = PROTO_IOBUF_LEN;
    /* If this is a multi bulk request, and we are processing a bulk reply
     * that is large enough, try to maximize the probability that the query
     * buffer contains exactly the SDS string representing the object, even
     * at the risk of requiring more read(2) calls. This way the function
     * processMultiBulkBuffer() can avoid copying buffers to create the
     * Redis Object representing the argument. */
    if (c->reqtype == PROTO_REQ_MULTIBULK && c->multibulklen && c->bulklen != -1
        && c->bulklen >= PROTO_MBULK_BIG_ARG)
    {
        ssize_t remaining = (size_t)(c->bulklen+2)-sdslen(c->querybuf);

        /* Note that the 'remaining' variable may be zero in some edge case,
         * for example once we resume a blocked client after CLIENT PAUSE. */
        if (remaining > 0 && remaining < readlen) readlen = remaining;
    }

    qblen = sdslen(c->querybuf);
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    // 先读取cfd内容到q->querybuf
    nread = read(fd, c->querybuf+qblen, readlen);   // 读取用户fd的内容到c->querybuf中

    ...
    processInputBufferAndReplicate(c);  // 处理用户请求内容(已经读取到c->querybuf中)
}
```

值得注意的是, redis的对每个命令字符串和执行命令的函数指针进行了映射, 这样只需要命令字符串就可以直接找到需要执行的函数
```cpp
typedef void redisCommandProc(client *c);   // 定义函数指针类型
struct redisCommand {
    char *name;
    redisCommandProc *proc; // 命令需要执行的函数
    int arity;
    char *sflags; /* Flags as string representation, one char per flag. */
    int flags;    /* The actual flags, obtained from the 'sflags' field. */
    /* Use a function to determine keys arguments in a command line.
     * Used for Redis Cluster redirect. */
    redisGetKeysProc *getkeys_proc;
    /* What keys should be loaded in background when calling this command? */
    int firstkey; /* The first argument that's a key (0 = no keys) */
    int lastkey;  /* The last argument that's a key */
    int keystep;  /* The step between first and last key */
    long long microseconds, calls;
};

struct redisCommand redisCommandTable[] = { // redis 字符串和命令对象的映射, 字符串是映射到一个命令函数, 该函数可被执行结果是
    {"module",moduleCommand,-2,"as",0,NULL,0,0,0,0,0},
    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
    {"setnx",setnxCommand,3,"wmF",0,NULL,1,1,1,0,0},
    {"setex",setexCommand,4,"wm",0,NULL,1,1,1,0,0},
    {"psetex",psetexCommand,4,"wm",0,NULL,1,1,1,0,0},
    {"append",appendCommand,3,"wm",0,NULL,1,1,1,0,0},
    {"strlen",strlenCommand,2,"rF",0,NULL,1,1,1,0,0},
    {"del",delCommand,-2,"w",0,NULL,1,-1,1,0,0},
    {"unlink",unlinkCommand,-2,"wF",0,NULL,1,-1,1,0,0},
    {"exists",existsCommand,-2,"rF",0,NULL,1,-1,1,0,0},
    {"setbit",setbitCommand,4,"wm",0,NULL,1,1,1,0,0},

typedef struct client {
    uint64_t id;            /* Client incremental unique ID. */
    int fd;                 /* Client socket. */
    redisDb *db;            /* Pointer to currently SELECTed DB. */
    robj *name;             /* As set by CLIENT SETNAME. */
    // 用户输入的字符放入querybuf中
    sds querybuf;           /* Buffer we use to accumulate client queries. */
    size_t qb_pos;          /* The position we have read in querybuf. */
    sds pending_querybuf;   /* If this client is flagged as master, this buffer
                               represents the yet not applied portion of the
                               replication stream that we are receiving from
                               the master. */
    size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size. */
    // 根据用户命令要执行的函数, 参数等
    int argc;               /* Num of arguments of current command. */
    robj **argv;            /* Arguments of current command. */
    struct redisCommand *cmd, *lastcmd;  /* 执行的命令Last command executed. */
    listNode *client_list_node; /* list node in client list */

    /* Response buffer, 结果放入client->buf中 */
    int bufpos;
    char buf[PROTO_REPLY_CHUNK_BYTES];
} client;
```

命令执行完的结果放在`client->buf`中, 这样对于client消息的回调函数就执行完毕了。

最后将执行完的结果`client->buf`发送到客户端, 发生在loop循环之中, 就是`eventLoop->beforesleep(eventLoop)`。

```cpp
void aeMain(aeEventLoop *eventLoop) {   // 事件执行主函数
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);  // 这里调用beforesleep将c->buf数组发送给客户端
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);  // 调用aeProcessEvents
    }
}

void beforeSleep(struct aeEventLoop *eventLoop) {   // eventloop的before sleep函数
    moduleHandleBlockedClients();

    /* Try to process pending commands for clients that were just unblocked. */
    if (listLength(server.unblocked_clients))
        processUnblockedClients();

    /* Write the AOF buffer on disk */
    flushAppendOnlyFile(0);

    /* Handle writes with pending output buffers. */
    handleClientsWithPendingWrites();   // 将输出buffers的数据发送给客户端

    if (moduleCount()) moduleReleaseGIL();
}

int handleClientsWithPendingWrites(void) {
    listIter li;
    listNode *ln;
    int processed = listLength(server.clients_pending_write);

    listRewind(server.clients_pending_write,&li);   // 将li指向server.clients双向链表, 所有的client节点用双向链表维护
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_WRITE;
        listDelNode(server.clients_pending_write,ln);

        /* If a client is protected, don't do anything,
         * that may trigger write error or recreate handler. */
        if (c->flags & CLIENT_PROTECTED) continue;

        /* Try to write buffers to the client socket. */
        if (writeToClient(c->fd,c,0) == C_ERR) continue;    // c->buf的数据发送给客户端
        }
    }
    return processed;
}
```

以上, 一次监听连接, 建立连接, 信息传输两个过程完成