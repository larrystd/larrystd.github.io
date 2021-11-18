---
title: muduo和tcp, http
date: 2021-09-13
updated: 2021-09-13
tags: 
  - muduo
  - cpp
categories:
  - source code
aplayer: true
---

之前在`muduo`系列中剖析了`muduo`网络库的实现过程。网络的层次介于socket和应用层之间。而常用的框架直接封装到了业务逻辑, 例如java servelet, 在处理请求时可以直接写业务逻辑。

![avatar](../../assets/images/muduo/6.png)

muduo网络库封装socket层到Channel, TcpConnection, Eventloop层, 其实是一个连接一个Channel, 一个TcpConnection; 同时根据增加了多线程eventloop, poller进行IO多路复用同时监听多个连接。那么多个连接怎么处理呢? 其实就是TcpServer和HttpServer是维护多个TcpConnection的更高层次的抽象。

* wsl2内部ubuntu和windows的端口映射设置

```
1. 获取虚拟机ip
wsl -- ifconfig eth0
2. 端口映射
# netsh interface portproxy add v4tov4 listenport=[win10端口] listenaddress=0.0.0.0 connectport=[虚拟机的端口] connectaddress=[虚拟机的ip]
netsh interface portproxy add v4tov4 listenport=80 listenaddress=0.0.0.0 connectport=80 connectaddress=172.29.41.233

3. 检查是否成功
netsh interface portproxy show all
```

以下是基于Java Serverlet编写的http处理逻辑。
```cpp
public class HelloWorld extends HttpServlet {
 
  private String message;

  public void init() throws ServletException
  {
      // 执行必需的初始化
      message = "Hello World";
  }
  public void doGet(HttpServletRequest request,
                    HttpServletResponse response)
            throws ServletException, IOException
  {
      // 设置响应内容类型
      response.setContentType("text/html");

      // 实际的逻辑是在这里
      PrintWriter out = response.getWriter();
      out.println("<h1>" + message + "</h1>");
  }
}

/// 配置url映射
<web-app>      
    <servlet>
        <servlet-name>HelloWorld</servlet-name>
        <servlet-class>HelloWorld</servlet-class>
    </servlet>

    <servlet-mapping>
        <servlet-name>HelloWorld</servlet-name>
        <url-pattern>/HelloWorld</url-pattern>
    </servlet-mapping>
</web-app>  
```

<!-- more -->

这样的框架直接将http层到socket层全部封装，用户只需要写业务逻辑，不用管其他发送消息等操作。这种效果在`muduo`的tcpserver上层封装一个http层即可实现。

如果tcpconnection回调tcpserver的函数实现, 可以建立一个httpserver用tcpserver回调httpserver的实现, 这一切都是底层回调上层的基本实现。

### muduo的tcpserver

muduo的TcpServer主要包含, 一个loop主循环, 一个acceptor接受连接, 一个线程池threadPool分配, 若干tcpConnection通信连接, 一些用户设置的回调函数注册到tcpConnection中。

```cpp
class Acceptor;
class EventLoop;
class EventLoopThreadPool;

/// TCP server, supports single-threaded and thread-pool models.
class TcpServer : noncopyable
{
 public:
  typedef std::function<void(EventLoop*)> ThreadInitCallback;
  enum Option
  {
    kNoReusePort,
    kReusePort,
  };
  TcpServer(EventLoop* loop,
            const InetAddress& listenAddr,
            const string& nameArg,
            Option option = kNoReusePort);  // 构造函数
  ~TcpServer();  // force out-line dtor, for std::unique_ptr members.

  const string& ipPort() const { return ipPort_; }
  const string& name() const { return name_; }
  EventLoop* getLoop() const { return loop_; }

  void setThreadNum(int numThreads);  // 线程个数
  void setThreadInitCallback(const ThreadInitCallback& cb)
  { threadInitCallback_ = cb; }
  std::shared_ptr<EventLoopThreadPool> threadPool() // 线程池对象
  { return threadPool_; }
  void start();

  // 设置回调函数, 用户在httpserver中手动设置
  void setConnectionCallback(const ConnectionCallback& cb)
  { connectionCallback_ = cb; } // 连接完成回调函数
  void setMessageCallback(const MessageCallback& cb)
  { messageCallback_ = cb; }  // 通信回调函数
  void setWriteCompleteCallback(const WriteCompleteCallback& cb)
  { writeCompleteCallback_ = cb; }  // 写毕回调函数

 private:
  /// Not thread safe, but in loop, 新的连接
  void newConnection(int sockfd, const InetAddress& peerAddr);
  typedef std::map<string, TcpConnectionPtr> ConnectionMap; // string到tcpconnection的映射

  EventLoop* loop_;  // the acceptor loop
  const string ipPort_;
  const string name_;

  std::unique_ptr<Acceptor> acceptor_; // avoid revealing Acceptor
  std::shared_ptr<EventLoopThreadPool> threadPool_; // eventloopthread线程池

  // 回调函数, 对应于TcpConnection, 注册到Channel
  ConnectionCallback connectionCallback_;
  MessageCallback messageCallback_;
  WriteCompleteCallback writeCompleteCallback_;
  ThreadInitCallback threadInitCallback_;
  AtomicInt32 started_;

  int nextConnId_;  // 下一个连接
  ConnectionMap connections_;   // map维护的connections, 包含若干TcpConnection
};
```

TcpServer.cc
```cpp
// TcpServer主要包含, 一个loop主循环, 一个acceptor接受连接, 一个线程池threadPool分配, 若干tcpConnection通信连接, 一些用户设置的回调函数注册到tcpConnection中
TcpServer::TcpServer(EventLoop* loop,
                     const InetAddress& listenAddr,
                     const string& nameArg,
                     Option option)
  : loop_(loop),  // tcpserver的主loop
    ipPort_(listenAddr.toIpPort()),
    name_(nameArg),
    acceptor_(new Acceptor(loop, listenAddr, option == kReusePort)), // 初始化acceptor对象监听socket,(调用listen(才开始监听)
    threadPool_(new EventLoopThreadPool(loop, name_)),     // 构造threadPool对象
    /// 初始化connection回调函数和message回调函数
    connectionCallback_(defaultConnectionCallback),
    messageCallback_(defaultMessageCallback),
    nextConnId_(1)
{
  acceptor_->setNewConnectionCallback(
      std::bind(&TcpServer::newConnection, this, _1, _2)); // 新连接一旦到达, 自动回调TcpServer::newConnection, 在子进程中封装为connection, 设置对应的Channel
}

void TcpServer::setThreadNum(int numThreads)
{
  assert(0 <= numThreads);
  threadPool_->setThreadNum(numThreads);
}

void TcpServer::start() // server创建线程池loop, 运行监听
{
  if (started_.getAndSet(1) == 0)
  { 
    threadPool_->start(threadInitCallback_); // 线程池启动, 结果是创建若干线程, 每个线程创建loop对象, 子线程执行loop()阻塞到里poll中
    assert(!acceptor_->listening());
    loop_->runInLoop(
        std::bind(&Acceptor::listen, get_pointer(acceptor_)));  // 主线程运行&Acceptor::listen监听, 新连接到来会调用&TcpServer::newConnection创建TcpConnection对象
  }
}
void TcpServer::newConnection(int sockfd, const InetAddress& peerAddr)  // 新连接到来会调用该函数, 基于sockfd
{
  loop_->assertInLoopThread();   // 这个loop_是主线程的, 主线程执行
  // 这里根本不用处理线程池, 而是直接分配一个ioLoop指针, 就可以让对应的子线程自动处理对象
  EventLoop* ioLoop = threadPool_->getNextLoop(); // 从线程池中找到一个可用的ioLoop, 这个ioLoop指向的loop对象在子线程里, 子线程还阻塞在eventloop的epoll_wait
  char buf[64];
  snprintf(buf, sizeof buf, "-%s#%d", ipPort_.c_str(), nextConnId_);  
  ++nextConnId_;
  string connName = name_ + buf;  // 创建connName

  LOG_INFO << "TcpServer::newConnection [" << name_
           << "] - new connection [" << connName
           << "] from " << peerAddr.toIpPort();
  InetAddress localAddr(sockets::getLocalAddr(sockfd));   // 封装IP地址

  TcpConnectionPtr conn(new TcpConnection(ioLoop,
                                          connName,
                                          sockfd,
                                          localAddr,
                                          peerAddr)); // 来了一个连接就创建一个TcpConnection,用子线程的loop指针, 该对象用shared_ptr维护
  
  // 设置好TcpConnection的回调函数, 这些回调函数来自于用户编写的逻辑
  connections_[connName] = conn;  // coonection name->conn的map
  conn->setConnectionCallback(connectionCallback_); // 设置tcpconnection的连接回调函数, 来自用户自定义。以下同样
  conn->setMessageCallback(messageCallback_);
  conn->setWriteCompleteCallback(writeCompleteCallback_);
  conn->setCloseCallback(
      std::bind(&TcpServer::removeConnection, this, _1));

  ioLoop->runInLoop(std::bind(&TcpConnection::connectEstablished, conn)); // 主线程将&TcpConnection::connectEstablished加入到子线程的loop对象的任务队列中, 使子线程自动执行任务
}
```

### muduo的 httpserver

* httpServer维护一个TcpServer对象, 内部的逻辑功能实际上作为TcpServer的ReadCallBack可读回调函数。具体是`server_.setConnectionCallback`和`server_.setMessageCallback`

* httpServer中调用了httpRequest和httpResponse两个类, 基本思路时从tcpserver中获取封装来到的信息成httpRequest类, 用户逻辑处理httpRequest为httpResponse, 最后将httpResponse的内容封装发送到客户端。

* 整个框架类似, 取出inputBuffer中的字符串->封装成httpRequest对象->处理httpRequest对象，得到httpResponse(用户执行逻辑), 并添加响应状态码->response序列化到outputBuffer->发送到客户端。

```cpp
#include "muduo/net/TcpServer.h"

namespace muduo
{
namespace net
{
/// 具有HttpRequest类和HttpResponse类, 如同servelet的HttpServletRequest和HttpServletResponse
class HttpRequest;
class HttpResponse;

class HttpServer : noncopyable
{
 public:
  typedef std::function<void (const HttpRequest&,
                              HttpResponse*)> HttpCallback;  // 回调函数, 用户输入, 传入参数(const HttpRequest&,HttpResponse*)
  /// 具有eventloop, 监听地址
  HttpServer(EventLoop* loop,
             const InetAddress& listenAddr,
             const string& name,
             TcpServer::Option option = TcpServer::kNoReusePort);


  void setThreadNum(int numThreads)
  {
    server_.setThreadNum(numThreads);
  }
  /// 实际上调用tcpserver的start方法
  void start();

 private:
 /// 维护回调函数
  void onConnection(const TcpConnectionPtr& conn);
  void onMessage(const TcpConnectionPtr& conn,
                 Buffer* buf,
                 Timestamp receiveTime);
  void onRequest(const TcpConnectionPtr&, const HttpRequest&);

  TcpServer server_;
  HttpCallback httpCallback_;
};
```

TcpServer.cc, 当数据可读时, 首先将fd的数据read到inputBuffer中, 接着调用HttpServer::onMessage解析读取的数据生成HttpConext, 最后调用onRequest。

onRequest调用`httpCallback_(req, &response);`,  `HttpCallback`实际上就是用户写的逻辑函数, 通过`server.setHttpCallback(onRequest);`设置。

执行完用户逻辑就得到了HttpResponse对象, 将其序列化为http报文到outputBuffer中, 执行`conn->send(&buf);`将缓冲区数据发送到客户端。
```cpp
// 设置TcpServer的连接回调函数
HttpServer::HttpServer(EventLoop* loop,
                       const InetAddress& listenAddr,
                       const string& name,
                       TcpServer::Option option)
  : server_(loop, listenAddr, name, option),
    httpCallback_(detail::defaultHttpCallback)
{
  server_.setConnectionCallback(
      std::bind(&HttpServer::onConnection, this, _1));

  // messageCallback绑定到tcpserver中, 会调用httpCallback
  server_.setMessageCallback(
      std::bind(&HttpServer::onMessage, this, _1, _2, _3));
}

/// httpserver::start, 转调用tcpserver的start
void HttpServer::start()
{
  LOG_WARN << "HttpServer[" << server_.name()
    << "] starts listening on " << server_.ipPort();
  server_.start();
}

void setHttpCallback(const HttpCallback& cb)  // 用户自定义逻辑,(没错这就是用户的全部)
{
  httpCallback_ = cb;
}

/// 有连接. 调用连接
void HttpServer::onConnection(const TcpConnectionPtr& conn)
{
  if (conn->connected())
  {
    conn->setContext(HttpContext());
  }
}
// 封装到messageCallback绑定到tcpserver, 这里的buf实际是TcpConnection的inputBuffer, 是从fd read到的缓冲区
void HttpServer::onMessage(const TcpConnectionPtr& conn,
                           Buffer* buf,
                           Timestamp receiveTime)
{
  HttpContext* context = boost::any_cast<HttpContext>(conn->getMutableContext());   // conn->getMutableContext()返回一个boost::any类型, 转型为HttpContext*
  if (!context->parseRequest(buf, receiveTime))   // 从buf中解析请求, 得到HttpContext中的  HttpRequest request_;
  {
    conn->send("HTTP/1.1 400 Bad Request\r\n\r\n");
    conn->shutdown();
  }

  if (context->gotAll())
  {
    /// 执行处理请求req, 并封装到resp中
    onRequest(conn, context->request());
    context->reset();
  }
}

/// 处理请求,整理到响应中, 封装TcpConnectionPtr& conn用于发送响应
void HttpServer::onRequest(const TcpConnectionPtr& conn, const HttpRequest& req)
{
  const string& connection = req.getHeader("Connection");
  bool close = connection == "close" ||
    (req.getVersion() == HttpRequest::kHttp10 && connection != "Keep-Alive");
  /// 响应
  HttpResponse response(close);

  /// 执行
  httpCallback_(req, &response);
  /// 当前的response
    /*response 格式
  {<muduo::copyable> = {<No data fields>}, headers_ = std::map with 2 elements = {["Content-Type"] = "text/html",
    ["Server"] = "Muduo"}, statusCode_ = muduo::net::HttpResponse::k200Ok, statusMessage_ = "OK",
  closeConnection_ = false,
  body_ = "<html><head><title>This is title</title></head><body><h1>Hello</h1>Now is 20210913 08:12:43.152553</body></html>"}
  */
  Buffer buf;
  response.appendToBuffer(&buf);   // 将response序列化成http报文到outputBuffer中
  conn->send(&buf);   // 将buf数据发送到客户端
  if (response.closeConnection())
  {
    conn->shutdown();
  }
}
```

#### httpRequest

HttpRequest是封装的Http请求对象, 就是封装的http请求报文, 包括请求方法，协议版本，请求路径, 报文body内容等

```cpp
class HttpRequest : public muduo::copyable
{
 public:
  enum Method // http请求方法
  {
    kInvalid, kGet, kPost, kHead, kPut, kDelete
  };
  enum Versio // http协议版本
  {
    kUnknown, kHttp10, kHttp11
  };

  HttpRequest()
    : method_(kInvalid),
      version_(kUnknown)
  {
  }

  Method method() const
  { return method_; }

  const char* methodString() const
  {
    const char* result = "UNKNOWN";
    switch(method_)
    {
      case kGet:
        result = "GET";
        break;
      case kPost:
        result = "POST";
        break;
      case kHead:
        result = "HEAD";
        break;
      case kPut:
        result = "PUT";
        break;
      case kDelete:
        result = "DELETE";
        break;
      default:
        break;
    }
    return result;
  }

  const string& path() const
  { return path_; }

  const string& query() const
  { return query_; }

  void setReceiveTime(Timestamp t)
  { receiveTime_ = t; }

  void setBody(const char* begin, const char* end) {  // 报文主体
    body_ = string(begin, end);
  }

  Timestamp receiveTime() const
  { return receiveTime_; }


  string getHeader(const string& field) const
  {
    string result;
    std::map<string, string>::const_iterator it = headers_.find(field);
    if (it != headers_.end())
    {
      result = it->second;
    }
    return result;
  }

  const std::map<string, string>& headers() const
  { return headers_; }


  string body_; /// http请求 body的内容, 用户自己解析吧
 private:
  /// 属性
  Method method_; /// 请求方法 get/post
  Version version_; /// http版本
  string path_; /// 路径
  string query_;

  Timestamp receiveTime_;
  std::map<string, string> headers_;
};
```

#### httpContext

httpContext主要是解析方法, 作用是从解析inputBuffer的字符串数据，封装到httpRequest对象。因此HttpContext内部有个HttpRequest

```cpp
class Buffer;

class HttpContext : public muduo::copyable
{
 public:
 /// http 请求的解析状态
  enum HttpRequestParseState
  {
    kExpectRequestLine,
    kExpectHeaders,
    kExpectBody,
    kGotAll,
  };

  /// 设置初始化状态为kExpectRequestLine
  HttpContext()
    : state_(kExpectRequestLine)
  {
  }

  const HttpRequest& request() const
  { return request_; }

  HttpRequest& request()
  { return request_; }

 private:
  bool processRequestLine(const char* begin, const char* end);

  HttpRequestParseState state_; //当前解析状态
  HttpRequest request_; // 封装的HttpRequest
};


// return false if any error
/// 解析http请求
/*
请求行, 请求头部, 请求报文
GET /favicon.ico HTTP/1.1\r\nHost: 172.20.109.213:9006\r\nConnection: keep-alive\r\nPragma: no-cache\r\nCache-Control: no-cache\r\nUser-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/93.0.4577.82 Safari/537.36\r\nAccept: image/avif,image/webp,image/apng,image/svg+xml,image/*
*/
bool HttpContext::parseRequest(Buffer* buf, Timestamp receiveTime)
{
  bool ok = true;
  bool hasMore = true;
  /// 一行一行解析
  /// state_ = muduo::net::HttpContext::kExpectRequestLine
  while (hasMore)
  {
    /// 解析请求行
    if (state_ == kExpectRequestLine)
    {
      /// 找CR LR \r\n, 也就是请求行结束的位置
      const char* crlf = buf->findCRLF();
      if (crlf)
      {
        /// 从buf中找到crlf
        // buf中存储的字符如"GET / HTTP/1.1\r\nHost: 127.0.0.1:8000\r\nUser-Agent: curl/7.61.0\r\nAccept: 
        /// 可以解析出method, httpversion
        ok = processRequestLine(buf->peek(), crlf); 
        if (ok)
        {
          request_.setReceiveTime(receiveTime);
          /// 读后更新buffer索引, 也就是更新buf->peek()可读位置
          buf->retrieveUntil(crlf + 2);
          /// kExpectHeaders
          state_ = kExpectHeaders;
        }
        else
        {
          hasMore = false;
        }
      }
      else
      {
        hasMore = false;
      }
    }
    /// 该解析请求头了
    else if (state_ == kExpectHeaders)  // 解析请求头
    {
      /// 请求头结束的地方
      const char* crlf = buf->findCRLF();
      if (crlf)
      {
        /// 在buf->peek()到crlf寻找:, peek是可读buf地址
        const char* colon = std::find(buf->peek(), crlf, ':');  
        if (colon != crlf)  // 如果可以找到
        {
          request_.addHeader(buf->peek(), colon, crlf);
        }
        else
        {
          /// 这一行说明该到Body了
          // empty line, end of header
          state_ = kExpectBody;
          
        }
        /// crlf + 位置已经读完
        buf->retrieveUntil(crlf + 2); /// 跳过\r\n
      }
      else
      {
        hasMore = false;
      }
    }

    else if (state_ == kExpectBody)
    {
      /// 这个直接让用户解析, 把Body给用户
      //if (buf->peek() != buf->beginWrite())

      if (buf->readableBytes() > 0 ) /// 还没有读完
        request_.setBody(buf->peek(), buf->beginWrite());

      state_ = kGotAll; /// 已经读完
      hasMore = false;
    }
  }
  return ok;
}
```

#### httpResponse

HttpResponse是HttpServer根据HttpRequest和处理函数执行逻辑之后，结果封装成的响应，需要返回给客户端。

HttpResponse.n
```cpp
class Buffer;
class HttpResponse : public muduo::copyable
{
 public:
 // http响应状态码
  enum HttpStatusCode
  {
    kUnknown,
    k200Ok = 200,
    k301MovedPermanently = 301,
    k400BadRequest = 400,
    k404NotFound = 404,
  };

  explicit HttpResponse(bool close)
    : statusCode_(kUnknown),
      closeConnection_(close)
  {
  }

  void setStatusCode(HttpStatusCode code)
  { statusCode_ = code; }

  void setStatusMessage(const string& message)
  { statusMessage_ = message; }

  void setCloseConnection(bool on)
  { closeConnection_ = on; }

  bool closeConnection() const
  { return closeConnection_; }

  //// 响应, contentType
  void setContentType(const string& contentType)
  { addHeader("Content-Type", contentType); }
  /// header，响应头
  void addHeader(const string& key, const string& value)
  { headers_[key] = value; }
  /// body, 响应报文(html文档等)
  void setBody(const string& body)
  { body_ = body; }

  void appendToBuffer(Buffer* output) const;

  string body_;
 private:
  std::map<string, string> headers_;
  HttpStatusCode statusCode_;
  string statusMessage_;
  bool closeConnection_;
};

/// 将要回复的状态码等信息放置到output buffer中, 也就是Reponse对象根据Http报文格式序列化到Buffer中
void HttpResponse::appendToBuffer(Buffer* output) const
{
  char buf[32];
  snprintf(buf, sizeof buf, "HTTP/1.1 %d ", statusCode_);
  output->append(buf);
  output->append(statusMessage_);
  output->append("\r\n");

  if (closeConnection_)
  {
    output->append("Connection: close\r\n");
  }
  else
  {
    snprintf(buf, sizeof buf, "Content-Length: %zd\r\n", body_.size());
    output->append(buf);
    output->append("Connection: Keep-Alive\r\n");
  }

  /// 响应头部
  for (const auto& header : headers_)
  {
    output->append(header.first);
    output->append(": ");
    output->append(header.second);
    output->append("\r\n");
  }

  output->append("\r\n");
  /// 设置Body
  output->append(body_);
}
```