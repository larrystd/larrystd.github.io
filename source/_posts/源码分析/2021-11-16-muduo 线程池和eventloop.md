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

EventLoopThread.h
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