---
title: muduo 线程池和loop
date: 2021-11-16
updated: 2021-11-16
tags: 
  - muduo
  - cpp
categories:
  - source code
aplayer: true
---

### EventLoopThread

基于`one loop one thread`原则, 每个线程都要拥有一个eventloop对象, 因此线程池里的线程是已经创建好eventloop对象的线程, 因此叫eventloopthread, 

EventLoopThread.h, EventLoopThread类包含要执行的任务函数threadFunc, 具有的loop*, 以及锁和条件变量。
```cpp
class EventLoop;

class EventLoopThread : noncopyable
{
 public:
 /// eventloop thread创建之后的回调函数, 参数为void(EventLoop*)
  typedef std::function<void(EventLoop*)> ThreadInitCallback;

  EventLoopThread(const ThreadInitCallback& cb = ThreadInitCallback(),
                  const string& name = string());
  ~EventLoopThread();
  EventLoop* startLoop();

 private:
  void threadFunc();

  /// 线程指向的loop对象
  EventLoop* loop_ GUARDED_BY(mutex_);
  bool exiting_;
  
  Thread thread_; // EventloopThread对象持有thread对象
  MutexLock mutex_; // 线程持有锁对象
  Condition cond_ GUARDED_BY(mutex_); // 持有条件变量对象, 使用之必须先获得mutex_
  ThreadInitCallback callback_;
};
```
<!-- more -->

EventLoopThread.cc, thread会维护成员变量thread_对象, 创建一个线程; 在新线程栈上创建该线程的eventloop对象, EventLoopThread对象持有新线程栈eventloop对象的指针。

```cpp
EventLoopThread::EventLoopThread(const ThreadInitCallback& cb,  
                                 const string& name)  // 用线程初始化回调函数构造函数
  : loop_(NULL),  // loop_是指针, 指向子线程所有的loop_对象
    exiting_(false),
    thread_(std::bind(&EventLoopThread::threadFunc, this), name), //构造线程对象 注册绑定线程执行的函数, EventLoopThread::threadFunc
    mutex_(),
    cond_(mutex_),
    callback_(cb) // 回调函数
{}

EventLoop* EventLoopThread::startLoop() // (主线程)开启线程(执行函数)
{
  assert(!thread_.started());
  thread_.start(); // 新线程t执行start() , 执行threadFunc, 主线程进行向下执行

  EventLoop* loop = NULL;
  {
    MutexLockGuard lock(mutex_);
    /// 条件变量主线程等待loop_
    while (loop_ == NULL)
    {
      cond_.wait();
    }
    // loop对象有了(新线程创建的), 返回loop指针(给loops_列表)
    loop = loop_;
  }
  return loop;
}

void EventLoopThread::threadFunc() // 新线程创建后执行的函数
{
  // 新线程栈上创建eventloop对象, 关键啊, 这里也会配置好loop对象的threadId_
  EventLoop loop;

  {
    MutexLockGuard lock(mutex_); 
    loop_ = &loop;     // loop_指向子线程内部的loop对象
    cond_.notify(); // 唤醒主线程已经有了loop_(主线程有指向子线程loop对象的指针, 可以退出阻塞)
  }
  // 新线程执行loop循环, 子线程会阻塞在这里, loop不会析构
  loop.loop();
  // 子线程执行loop()完毕
  MutexLockGuard lock(mutex_);
  loop_ = NULL;
  // 退出, 子线程析构loop对象
}
```

### 线程池 EventLoopThreadPool

EventLoopThreadPool.h, 线程池对象就是使用一个队列维护多个EventLoopThread, 也就是`std::vector<std::unique_ptr<EventLoopThread>> threads_`, 由于one thread one loop, 还会有个`loop*`列表`std::vector<EventLoop*> loops_;`。 当用户请求时只需要给用户分配一个可用的`loop*`, 因为loop对象持有创建loop子线程的tid。

```cpp
class EventLoop;
class EventLoopThread;  

class EventLoopThreadPool : noncopyable
{
 public:
  typedef std::function<void(EventLoop*)> ThreadInitCallback;  // 线程初始化回调函数

  EventLoopThreadPool(EventLoop* baseLoop, const string& nameArg);
  ~EventLoopThreadPool();
  void setThreadNum(int numThreads) { numThreads_ = numThreads; }
  void start(const ThreadInitCallback& cb = ThreadInitCallback());

  EventLoop* getNextLoop(); // 返回一个可用的loop*

  /// with the same hash code, it will always return the same EventLoop
  EventLoop* getLoopForHash(size_t hashCode);

  std::vector<EventLoop*> getAllLoops();  // Evnetloop线程池对象持有指向所有loop对象的vector

  bool started() const
  { return started_; }

  const string& name() const
  { return name_; }

 private:

  EventLoop* baseLoop_;
  string name_;
  
  bool started_;
  int numThreads_;
  int next_;

  std::vector<std::unique_ptr<EventLoopThread>> threads_; // eventloopthread线程指针列表, 线程对象在堆上
  std::vector<EventLoop*> loops_; // eventloop对象指针的vector, loop对象建在线程对象内部
};
```

EventLoopThreadPool.cc, start()函数用来创建指定数量的eventloopthread和loop到列表中, 同时eventloopthread执行loop()函数阻塞在epoll_wait()(`loops_.push_back(t->startLoop())` 子线程创建loop对象执行loop(), 主线程返回指向该对象的指针)
```cpp
EventLoopThreadPool::EventLoopThreadPool(EventLoop* baseLoop, const string& nameArg)  // 用baseloop(主线程的loop)构建eventloopthreadpool对象
  : baseLoop_(baseLoop),
    name_(nameArg),
    started_(false),
    numThreads_(0),
    next_(0)
{
}
void EventLoopThreadPool::start(const ThreadInitCallback& cb) // 线程池的start, 传入线程创建好的回调函数
{
  assert(!started_);
  baseLoop_->assertInLoopThread();  // 执行的线程必须是baseLoop所在线程, 也就是main线程
  started_ = true;
  for (int i = 0; i < numThreads_; ++i) // 依次创建线程
  {

    char buf[name_.size() + 32];
    snprintf(buf, sizeof buf, "%s%d", name_.c_str(), i);  // 线程名储存到buf中
    EventLoopThread* t = new EventLoopThread(cb, buf);  // 创建一个EventLoopThread
    threads_.push_back(std::unique_ptr<EventLoopThread>(t));  // 线程指针t用unique_ptr维护, 加入的线程列表中
    loops_.push_back(t->startLoop()); // t->startLoop()返回的loop_指针加入到执行loop列表中
  }
  if (numThreads_ == 0 && cb) // 执行线程池创建好的回调函数cb
  {
    cb(baseLoop_);
  }
}

EventLoop* EventLoopThreadPool::getNextLoop() // 返回下一个loop指针(可用的指针), 给tcpserver和client
{
  baseLoop_->assertInLoopThread();
  assert(started_);
  EventLoop* loop = baseLoop_;

  if (!loops_.empty())  // 如果loop不为空
  {
    loop = loops_[next_];
    ++next_;
    if (implicit_cast<size_t>(next_) >= loops_.size())
    {
      next_ = 0;  // 重置next_
    }
  }
  return loop;  // loop对象指针
}
```

### eventloop

eventloop时muduo使用reactor+one loop, one thread模式的核心, 起始就是一个loop对象。每个线程都会创建一个eventloop对象, 主线程为主loop, 注册sockfd到poll中, 如果响应说明有连接到来同时返回一个fd处理; 子线程为子loop, 主线程接收连接fd分配到子线程, 以后子线程子loop即负责该fd连接的通信处理。

除了poll的网络任务, 还有一些任务可能也需要子线程执行, 例如定时器任务以及子线程将Channel连接封装连接成TcpConnection, 这些任务有可能是主线程让子线程执行的(主线程接受到任务但是它想让子线程执行), 这时调用`runInLoop()`。如果调用进程是loop所属的子线程那自然它执行, 如果不是则先将任务放入到子线程的任务队列中(要加锁), 因为有可能子线程正阻塞在`epoll_wait`中, 所以主线程再发送一个eventfd唤醒子线程的等待, 子线程最后执行任务队列的任务即完成了以上的目标。总之主线程只负责，接受连接，分发任务，执行逻辑和任务交由线程池的子线程处理。

```cpp
class Channel;
class Poller;
class TimerQueue;

class EventLoop : noncopyable
{
 public:
  typedef std::function<void()> Functor;

  EventLoop();
  ~EventLoop();  // force out-line dtor, for std::unique_ptr members.

  /// Loops forever.
  void loop();
  void quit();

  Timestamp pollReturnTime() const { return pollReturnTime_; }  // poll触发返回的时间戳
  int64_t iteration() const { return iteration_; }
  void runInLoop(Functor cb); // 保证在loop所属线程中执行某函数, 如果在线程立即执行, 如果不在调用queueInLoop
  void queueInLoop(Functor cb); // 放入到loop对象的等待队列中并触发loop对象所属线程执行

  size_t queueSize() const;

  // timers, 设置定时器任务
  TimerId runAt(Timestamp time, TimerCallback cb);  // 某个时刻执行定时任务
  TimerId runAfter(double delay, TimerCallback cb); // 再过某个时间段执行的定时任务
  TimerId runEvery(double interval, TimerCallback cb);  // 设置一个循环定时器
  void cancel(TimerId timerId); // 取消定时器

  // internal usage
  /// 更新channel，就是更新需要poll的连接，因此调用poller内部函数实现
  void wakeup();
  void updateChannel(Channel* channel);
  void removeChannel(Channel* channel);
  bool hasChannel(Channel* channel);

  // pid_t threadId() const { return threadId_; }
  void assertInLoopThread()
  {
    if (!isInLoopThread())
    {
      abortNotInLoopThread();
    }
  }
  // 当前线程idCurrentThread::tid() 为 构建thread_loop线程的id
  bool isInLoopThread() const { return threadId_ == CurrentThread::tid(); } 
  // bool callingPendingFunctors() const { return callingPendingFunctors_; }
  bool eventHandling() const { return eventHandling_; }

  void setContext(const boost::any& context)
  { context_ = context; }

  const boost::any& getContext() const
  { return context_; }

  boost::any* getMutableContext()
  { return &context_; }

  static EventLoop* getEventLoopOfCurrentThread();

 private:

  void abortNotInLoopThread();   // EventLoop对象创建者并非本线程
  void handleRead();    // wakefd触发的回调函数
  void doPendingFunctors();   // 运行等待的任务
  void printActiveChannels() const; // 打印当前被触发的channel

  typedef std::vector<Channel*> ChannelList;  // 已经使用poll注册监听的channel
  bool looping_; // 是否处于循环
  std::atomic<bool> quit_;  // 设置原子的(这里没啥用, 因为loop对象不是线程共享的)
  bool eventHandling_; // 在处理事件
  bool callingPendingFunctors_; // 在执行PendingFunctors_
  int64_t iteration_; // loop的迭代次数
  const pid_t threadId_;    /// tid, loop对象所属线程的tid
  Timestamp pollReturnTime_;   // pollReturnTime_ poll返回的时间戳
  std::unique_ptr<Poller> poller_;  // poller_, IO多路复用
  std::unique_ptr<TimerQueue> timerQueue_;  // 定时器队列

  int wakeupFd_;
  // unlike in TimerQueue, which is an internal class,
  // we don't expose Channel to client.
  std::unique_ptr<Channel> wakeupChannel_;
  boost::any context_;

  // scratch variables
  ChannelList activeChannels_;
  Channel* currentActiveChannel_;

  mutable MutexLock mutex_;

  std::vector<Functor> pendingFunctors_ GUARDED_BY(mutex_);   // 任务队列, 建立在线程栈中
};
```

EventLoop.cc
```cpp
__thread EventLoop* t_loopInThisThread = 0; // 线程局部变量, 本线程的loop对象指针

const int kPollTimeMs = 10000;  // poll的时间, 毫秒

// eventfd在内核里的核心是一个计数器counter，它是一个uint64_t的整形变量counter，初始值为initval。
// read操作 如果当前counter > 0，那么read返回counter值，并重置counter为0；如果当前counter等于0，那么read 1)阻塞直到counter大于0
// write操作 write尝试将value加到counter上。write可以多次连续调用，但read读一次即可清零
int createEventfd()
{
  int evtfd = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC); // 创建eventfd, 用于唤醒正在epoll_wait的子线程
  if (evtfd < 0)
  {
    LOG_SYSERR << "Failed in eventfd";
    abort();
  }
  return evtfd;
}
EventLoop* EventLoop::getEventLoopOfCurrentThread() // 得到当前线程的loop指针(线程内部变量)
{
  return t_loopInThisThread;
}

EventLoop::EventLoop()
  : looping_(false),
    quit_(false),
    eventHandling_(false),
    callingPendingFunctors_(false),
    iteration_(0),
    threadId_(CurrentThread::tid()),  // 创建eventloop对象的子线程(eventloop为线程栈对象), eventLoop在子线程的线程池中就已经创建
    poller_(Poller::newDefaultPoller(this)),  // 用loop*创建pooler, 是pool存在用unique_ptr维护的指针
    timerQueue_(new TimerQueue(this)),  // 通过loop*创建timerQueue, 赋给timerQueue_指针
    wakeupFd_(createEventfd()), // 创建wakeupFd_
    wakeupChannel_(new Channel(this, wakeupFd_)), // 基于loop* 和wakeupFd创建wakeup通道
    currentActiveChannel_(NULL) // poll之后的活跃通道
{
  LOG_DEBUG << "EventLoop created " << this << " in thread " << threadId_;
  if (t_loopInThisThread) // 线程中存在loop指针(构造eventloop线程不应该持有loop指针)
  {
    LOG_FATAL << "Another EventLoop " << t_loopInThisThread
              << " exists in this thread " << threadId_;
  }
  else
  {
    t_loopInThisThread = this;  // 设置线程变量t_loopInThisThread指向本eventloop对象
  }
  wakeupChannel_->setReadCallback(
      std::bind(&EventLoop::handleRead, this)); // wakeupfd可读的回调函数为&EventLoop::handleRead
  wakeupChannel_->enableReading();  // 设置wakeupChannel_可读触发, 注册到poll中
}

EventLoop::~EventLoop()
{
  LOG_DEBUG << "EventLoop " << this << " of thread " << threadId_
            << " destructs in thread " << CurrentThread::tid();
  wakeupChannel_->disableAll(); // wakeup通道disable和remove
  wakeupChannel_->remove();
  ::close(wakeupFd_); // 关闭wakeupfd
  t_loopInThisThread = NULL;  // 线程不持有t_loopInThisThread
}

void EventLoop::loop()  // 主要函数, 线程内部线程栈有eventloop对象, 线程一直处于loop循环中。
{
  assert(!looping_);
  assertInLoopThread(); // 线程内部loop只能由这个线程执行
  looping_ = true;
  quit_ = false;
  LOG_TRACE << "EventLoop " << this << " start looping";

  while (!quit_)  // 只要quit不为true就循环
  {

    activeChannels_.clear();  // 先清空std::vector<Channel*> ChannelList 活跃channel
    pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);  // 线程一般会阻塞在poll中, 内部是epoll_wait, 等待触发的事件, 返回的触发事件列表
    /// 打印活跃的channel
    if (Logger::logLevel() <= Logger::TRACE)  // 打印活跃的channel
    {
      printActiveChannels();
    }
    eventHandling_ = true;
    for (Channel* channel : activeChannels_)
    {
      currentActiveChannel_ = channel;
      /// 处理handleEvent函数
      currentActiveChannel_->handleEvent(pollReturnTime_); // 调用channel的handleEvent函数, 实际上根据poll的事件类型调用channel的回调函数。回调函数包括应用层逻辑
    }
    // 清空ActiveChannel_
    currentActiveChannel_ = NULL;
    eventHandling_ = false;
    doPendingFunctors();  // 执行需要该线程执行的任务队列, 包括Tcp连接建立&TcpConnection::connectEstablished, 定时器任务等
  }

  LOG_TRACE << "EventLoop " << this << " stop looping";
  looping_ = false;
}

void EventLoop::quit() // 退出loop循环, 这个主线程执行
{
  quit_ = true; // 设置quit_ = true,阻断loop()循环
  if (!isInLoopThread())
  {
    wakeup(); // 这里防止子线程阻塞在epoll_wait, 唤醒之
  }
}

void EventLoop::runInLoop(Functor cb)
{
  if (isInLoopThread()) // 如果线程是loop所在线程, 执行之
  {
    cb();
  }
  else
  {
    queueInLoop(std::move(cb)); // 线程不是loop所在线程, 调用queueInLoop加入到loop线程的loop对象的执行队列中
  }
}

/// 将任务加入到任务队列中
void EventLoop::queueInLoop(Functor cb)
{
  {
  MutexLockGuard lock(mutex_);
  pendingFunctors_.push_back(std::move(cb));  // 多个线程可以将任务加入到同一个线程的pendingFunctors_, 因此需要同步
  }
  // 这里还是主线程, 调用wakeup唤醒子线程(阻塞在epollwait呢)
  if (!isInLoopThread() || callingPendingFunctors_) // 执行的不是loop所属线程线程或者, loop线程在执行pendingFunctors(没有阻塞)
  {
    wakeup(); // 调用wakeup唤醒epoll_wait的子线程
  }
}


TimerId EventLoop::runAt(Timestamp time, TimerCallback cb)
{
  return timerQueue_->addTimer(std::move(cb), time, 0.0); // 将cb, time, 0.0表示在time时刻执行cb的定时
}

TimerId EventLoop::runEvery(double interval, TimerCallback cb)
{
  Timestamp time(addTime(Timestamp::now(), interval));
  return timerQueue_->addTimer(std::move(cb), time, interval);  // 有间隔的时间戳加入定时器
}

void EventLoop::updateChannel(Channel* channel) // 调用poller更新channel->fd的事件监听清空
{
  assert(channel->ownerLoop() == this);
  assertInLoopThread();

  poller_->updateChannel(channel);
}

void EventLoop::wakeup()  // sockets::write(wakeupfd)发送一个写操作, 以触发wakeupFd_的可读事件。解除loop阻塞在epoll_wait。
{
  uint64_t one = 1;
  /// 这里是主线程调用, 但wakeupFd_却是子线程的, 结果就是主线程调sockets::write(wakeupFd_, 被对应子线程的poller接收, 起到了唤醒的作用
  ssize_t n = sockets::write(wakeupFd_, &one, sizeof one);
  if (n != sizeof one)
  {
    LOG_ERROR << "EventLoop::wakeup() writes " << n << " bytes instead of 8";
  }
}

void EventLoop::handleRead()  // 读wakeupFd_回调
{
  uint64_t one = 1;
  ssize_t n = sockets::read(wakeupFd_, &one, sizeof one);
  if (n != sizeof one)
  {
    LOG_ERROR << "EventLoop::handleRead() reads " << n << " bytes instead of 8";
  }
}

void EventLoop::doPendingFunctors() // 运行等待队列的函数
{
  /// 新建一个functor
  std::vector<Functor> functors;
  callingPendingFunctors_ = true;
  /// CAS操作， 有可能线程正在执行pendingFunctors_, 因此需要CAS防止竞态
  {
  MutexLockGuard lock(mutex_);
  functors.swap(pendingFunctors_);
  }

  /// 全部执行完
  for (const Functor& functor : functors)
  {
    functor();
  }
  callingPendingFunctors_ = false;
}
```