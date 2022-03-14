---
title: boost.asio分析
date: 2022-02-20
updated: 2022-02-20
tags: 
  - boost.Asio
categories:
  - network
aplayer: true
---

### 结构

```
boost::asio：这是核心类和函数所在的地方。重要的类有io_service和streambuf。类似read, read_at, read_until方法，它们的异步方法，它们的写方法和异步写方法等自由函数也在这里。
boost::asio::ip：这是网络通信部分所在的地方。重要的类有address, endpoint, tcp, udp和icmp，重要的自由函数有connect和async_connect。要注意的是在boost::asio::ip::tcp::socket中间，socket只是boost::asio::ip::tcp类中间的一个typedef关键字。
boost::asio::error：这个命名空间包含了调用I/O例程时返回的错误码
boost::asio::ssl：包含了SSL处理类的命名空间
boost::asio::local：这个命名空间包含了POSIX特性的类
boost::asio::windows：这个命名空间包含了Windows特性的类
```

#### io_service

io_servie 实现了一个任务队列，这里的任务就是void(void)的函数。Io_servie最常用的两个接口是post和run，post向任务队列中投递任务，run是执行队列中的任务，直到全部执行完毕，并且run可以被N个线程调用。Io_service是完全线程安全的队列。

```cpp
class io_service: private noncopyable
{
private:
  typedef detail::io_service_impl impl_type;
  ...
  // The service registry.
  boost::asio::detail::service_registry* service_registry_;
  // The implementation.
  impl_type& impl_;
public:
   ...
   // Run the io_service object's event processing loop.
   BOOST_ASIO_DECL std::size_t run();
   // Run the io_service object's event processing loop to execute ready handlers.
   BOOST_ASIO_DECL std::size_t poll();
 };
```

<!-- more -->

* Post向队列中投递任务，然后激活空闲线程执行任务。 

wake_one_thread_and_unlock尝试唤醒当前空闲的线程，其实现中特别之处在于，若没有空闲线程，但是有线程在执行task->run，即阻塞在epoll_wait上，那么先中断epoll_wait执行任务队列完成后再执行epoll_wait。

first_idle_thread_维护了所有当前空闲线程，实际上使用了Leader/Follower模式，每次唤醒时只唤醒空闲线程的第一个。

* Run方法执行队列中的所有任务，直到任务执行完毕。

do_one每次执行一个任务。首先检查队列是否为空，若空将此线程追加到first_idle_thread_的首部，然后阻塞在条件变量上，直到被唤醒。

run方法实际上执行的是epoll_wait，run阻塞在epoll_wait上等待事件到来，并且处理完事件后将需要回调的函数push到io_servie的任务队列中，虽然epoll_wait是阻塞的，但是它提供了interrupt函数, 方法就是通过eventfd_select_interrupter

Run执行的一般逻辑, 若没有任务，阻塞在epoll_wait上等待io事件; 若有新任务到来，并且没有空闲线程，那么先中断epoll_wait,先执行任务

#### Reactor和epoll

Asio在Linux的Proactor实现是通过封装epoll, 其调用一次epoll_wait，得到相应的IO事件, 若是基本IO事件，依次检查其IN、OUT事件，except事件会首先检测，将此事件对应的队列上的操作全部执行完毕

task_io_service是proactor模式在Linux下的具体实现，主要功能有两个：
1. 对IO是否就绪的进行扫描
2. 事件到达后对线程池的统一调度

```cpp
class task_io_service : public boost::asio::detail::service_base<task_io_service>
{
public:
   ...
   typedef task_io_service_operation operation;
  // Run the event loop until interrupted or no more work.
  BOOST_ASIO_DECL std::size_t run(boost::system::error_code& ec);
private:
   ...
  // The task to be run by this service.
  reactor* task_;
  // The count of unfinished work.
  atomic_count outstanding_work_;
  // The queue of handlers that are ready to be delivered.
  op_queue<operation> op_queue_;  // 回调函数对象列表, 每个操作选择一个空闲的线程
}
```

task_io_service的run函数会调用Linux下的epoll_reactor的run函数。epoll_reactor的run()方法最终调用的是epoll的epoll_wait的，通过epoll_wait，将就绪的事件放入ops，等待处理。

```cpp
void epoll_reactor::run(bool block, op_queue<operation>& ops)
{ 
    ... 
    // Block on the epoll descriptor. 
    epoll_event events[128]; 
    int num_events = epoll_wait(epoll_fd_, events, 128, timeout); 
    ...
   descriptor_state* descriptor_data = static_cast<descriptor_state*>(ptr); 
   descriptor_data->set_ready_events(events[i].events);  
   ops.push(descriptor_data);
}
```

#### Proactor

Proactor相比于Reactor最大区别是, Proactor在事件对应的操作完成时，通知调用者, 而Reactor当事件发生时即通知调用者。

Reactor中注册读事件，那么文件描述符可读时，需要调用者自己调用read系统调用读取数据，若工作在Preactor模式，注册读事件，同时提供一个buffer用于存储读取的数据，那么Preactor通过回调函数通知用户时，用户无需在调用系统调用读取数据，因为数据已经存储在buffer中了。显然epoll是Reactor的。

Boost.Asio中对TCP类的封装如下：
```cpp
class tcp
{
public:
  /// The type of a TCP endpoint.
  typedef basic_endpoint<tcp> endpoint;
  /// Construct to represent the IPv4 TCP protocol.
  static tcp v4()
  {
    return tcp(BOOST_ASIO_OS_DEF(AF_INET));
  }
...
  /// The TCP socket type.
  typedef basic_stream_socket<tcp> socket;
  /// The TCP acceptor type.
  typedef basic_socket_acceptor<tcp> acceptor;
  /// The TCP resolver type.
  typedef basic_resolver<tcp> resolver;
#if !defined(BOOST_ASIO_NO_IOSTREAM)
  /// The TCP iostream type.
  typedef basic_socket_iostream<tcp> iostream;
#endif // !defined(BOOST_ASIO_NO_IOSTREAM)
...
private:
  // Construct with a specific family.
  explicit tcp(int protocol_family)
    : family_(protocol_family)
  {
  }
  int family_;
};
```

basic_stream_socket<tcp>类似于TCP中的socket，basic_socket_acceptor<tcp>用于监听套接字，它们均有许多异步方法，如async_receive，async_send等

```cpp
template <typename MutableBufferSequence, typename ReadHandler>
  BOOST_ASIO_INITFN_RESULT_TYPE(ReadHandler,
      void (boost::system::error_code, std::size_t))
  async_receive(const MutableBufferSequence& buffers,
      socket_base::message_flags flags,
      BOOST_ASIO_MOVE_ARG(ReadHandler) handler)
  {
    // If you get an error on the following line it means that your handler does
    // not meet the documented type requirements for a ReadHandler.
    BOOST_ASIO_READ_HANDLER_CHECK(ReadHandler, handler) type_check;
    return this->get_service().async_receive(this->get_implementation(),
        buffers, flags, BOOST_ASIO_MOVE_CAST(ReadHandler)(handler));
  }
```

通过reactor，将epoll与事件处理联系起来了，在reactive_socket_service_base中，async_receive的实现为
```cpp
// Start an asynchronous receive. The buffer for the data being received
  // must be valid for the lifetime of the asynchronous operation.
  template <typename MutableBufferSequence, typename Handler>
  void async_receive(base_implementation_type& impl,
      const MutableBufferSequence& buffers,
      socket_base::message_flags flags, Handler& handler)
  {
    bool is_continuation =
      boost_asio_handler_cont_helpers::is_continuation(handler);
    // Allocate and construct an operation to wrap the handler.
    typedef reactive_socket_recv_op<MutableBufferSequence, Handler> op; // op
    typename op::ptr p = { boost::asio::detail::addressof(handler),
      boost_asio_handler_alloc_helpers::allocate(
        sizeof(op), handler), 0 };
    p.p = new (p.v) op(impl.socket_, impl.state_, buffers, flags, handler);
    BOOST_ASIO_HANDLER_CREATION((p.p, "socket", &impl, "async_receive"));
    start_op(impl,
        (flags & socket_base::message_out_of_band)
          ? reactor::except_op : reactor::read_op,
        p.p, is_continuation,
        (flags & socket_base::message_out_of_band) == 0,
        ((impl.state_ & socket_ops::stream_oriented)
          && buffer_sequence_adapter<boost::asio::mutable_buffer,
            MutableBufferSequence>::all_empty(buffers)));
    p.v = p.p = 0;
  }
```

reactive_socket_recv_op对象封装用户回调函数，设置事件状态；start_op调用epoll_reactor的start_op将read_op操作注册到epoll的文件描述符中。

在这两个过程中，reactive_socket_recv_op对象op起到了桥梁作用，一方面通过epoll检查对应描述符事件是否就绪，另一方面在就绪后进行数据IO操作，并触发用户注册的回调函数。可见用reactor模拟的proactor用户面对的接口一致，但数据的复制是在用户态而非内核态完成。但注意windows下具有原生的proactor而难以实现reactor(与Linux相反, 而reactor可以模拟proactor, 所以为了跨平台Asio只有实现Proactor模式了

### 异步编程

io_context的主要功能：接受任务，处理io事件，回调。io_context在windows平台具体实现是win_iocp_io_context，实现了原生的proactor，其他平台则是一个scheduler。

```cpp
// file: <boost/asio/io_context.hpp>
...
#if defined(BOOST_ASIO_HAS_IOCP)
typedef class win_iocp_io_context io_context_impl;
class win_iocp_overlapped_ptr;
#else
typedef class scheduler io_context_impl;
#endif
...
```

scheduler包含一个reactor，scheduler通过reactor模拟proactor：用户面对的接口一致，但数据的复制是在用户态而非内核态完成。reactor在不同平台的实现也不同，其中linux实现为基于epoll的epoll_reactor，macOS实现为基于kqueue的kqueue_reactor。

```cpp
// file: <boost/asio/detail/reactor_fwd.hpp>
...
#if defined(BOOST_ASIO_HAS_IOCP) || defined(BOOST_ASIO_WINDOWS_RUNTIME)
typedef class null_reactor reactor;
#elif defined(BOOST_ASIO_HAS_IOCP)
typedef class select_reactor reactor;
#elif defined(BOOST_ASIO_HAS_EPOLL)
typedef class epoll_reactor reactor;
#elif defined(BOOST_ASIO_HAS_KQUEUE)
typedef class kqueue_reactor reactor;
#elif defined(BOOST_ASIO_HAS_DEV_POLL)
typedef class dev_poll_reactor reactor;
#else
typedef class select_reactor reactor;
#endif
...
```

异步编程
```cpp

#include <boost/asio.hpp> 
#include <iostream> 
 
void handler(const boost::system::error_code &ec) 
{ 
   std::cout << "5 s." << std::endl; 
} 
 
int main() 
{ 
   boost::asio::io_service io_service; 
   boost::asio::deadline_timer timer(io_service, boost::posix_time::seconds(5)); 
   timer.async_wait(handler); 
   io_service.run(); 
}
```

使用 Boost.Asio 进行异步数据处理的应用程序基于两个概念：I/O 服务和 I/O 对象, I/O服务也就是io_service。 而对于I/O对象, 类 boost::asio::ip::tcp::socket 用于通过网络发送和接收数据，而类 boost::asio::deadline_timer 则提供了一个计时器，用于测量某个固定时间点到来或是一段指定的时长过去了。

函数 main() 首先定义了一个 I/O 服务 io_service，用于初始化 I/O 对象 timer。 我们通过调用方法 async_wait() 并传入 handler() 函数的名字作为唯一参数，可以让 Asio 启动一个异步操作。调用 async_wait() 之后，又在 I/O 服务之上调用了一个名为 run() 的方法。

async_wait() 的好处是，该函数调用会立即返回，而不是等待五秒钟。 一旦定时时间到，作为参数所提供的函数就会被相应调用。

对于异步io来说，用户无法直接等待io完成后进行后续的操作，这类"操作"需要保存起来，在io到达某个特定状态后由库自动调用。网络功能是异步处理的一个很好的例子，因为通过网络进行数据传输可能会需要较长时间，从而不能直接获得确认或错误条件。C++的异步操作包含的类型有callbacks、future、resumable functions及协程。

异步编程比同步编程要简单, 只需要注册回调函数就不用管了, 而协程比异步编程还简单, 因为它只需要同步代码, 回调函数可以认为是用来唤醒协程执行某个区域的代码。