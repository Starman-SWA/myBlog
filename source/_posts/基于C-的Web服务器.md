---
title: 基于C++的Web服务器
toc: true
date: 2023-11-15 17:17:09
tags:
- 后端
- 网络编程
categories:
- C++
---

# 项目简介

## 编程模型

- epoll I/O多路复用

- ET边沿触发+EPOLLONESHOT

- 非阻塞I/O

- Reactor模式

## 实现功能

- HTTP GET请求

- HTTP长连接+超时处理

- 日志

# 数据结构

## 线程池

```cpp
class ThreadPool {
public:
    explicit ThreadPool(std::size_t _thread_num, std::size_t _max_request_num) : \
        threads(_thread_num), request_queue(_max_request_num) {
        for (auto& thread : threads) {
            thread = std::thread(&ThreadPool::worker, this);
        }
    }
    ThreadPool(ThreadPool&) = delete;
    ThreadPool(ThreadPool&&) = delete;
    ThreadPool& operator=(ThreadPool&) = delete;
    ThreadPool& operator=(ThreadPool&&) = delete;
    ~ThreadPool();

    bool append(RequestHandler* p_req);

    void worker();
private:
    std::vector<std::thread> threads;
    ThreadSafeQueue<RequestHandler*> request_queue;
    bool stop = false;
};
```

采用`std::thread`线程库创建若干个工作线程。线程池包含一个线程安全队列`ThreadSafeQueue`，保存工作线程待处理的事件队列。实现`append`和`worker`方法，`append`方法将新事件加入线程安全队列，待线程池处理；`worker`方法为每个工作线程的工作函数，包含一个循环，工作线程从队列当中循环读取事件，并根据事件类型，执行读处理函数或写处理函数。

## RequestHandler

```cpp
class RequestHandler {
public:
    explicit RequestHandler(std::size_t read_buf_size = 2048, std::size_t write_buf_size = 2048) : read_state(read_buf_size),
                                                                                     write_state(write_buf_size) {
        signal(SIGPIPE, SIG_IGN);
    }
    RequestHandler(RequestHandler&) = delete;
    RequestHandler(RequestHandler&&) = default;
    RequestHandler& operator=(RequestHandler&) = delete;
    RequestHandler& operator=(RequestHandler&&) = delete;
    ~RequestHandler() = default;

    void reset(int conn_fd, TimerTask* timer_task);
    void process_read();
    void process_write();

    static void close_conn_static(int conn_fd, TimerTask* timer_task);
    void close_conn(int conn_fd);
    [[nodiscard]] int get_conn_fd() const;

    static void mod_fd_to_epoll(int epoll_fd, int fd, uint32_t events);

    static int epoll_fd;

    using TaskType = enum {READ, WRITE};
    TaskType task_type = READ;

    static TimerHeap* timer_heap;
    TimerTask* timer_task = nullptr;
protected:
    int conn_fd = -1;

    ReadState read_state;
    WriteState write_state;

    bool keep_alive = false;

    void write_status_code_to_buf(int status_code, const std::string &err_msg, size_t body_length = 0);
    void write_file_info_to_buf(const std::string &url);
    bool read_socket_to_buf();
    bool write_buf_to_socket();
    bool parse_one_request();
};
```

`RequestHandler`为请求处理类，一个`RequestHandler`实例处理一个HTTP请求。`RequestHandler`类包含一个`ReadState`成员和一个`WriteState`成员，分别包含该socket的读、写状态信息和对应的缓冲区。`RequestHandler`类还包含当前触发事件类型（读或写）和相应的处理函数`process_read()`和`process_write()`。`RequestHandler`还包含超时处理的相关结构，将在后面的章节单独讨论，这里暂且不表。

### ReadState和WriteState

- `ReadState`包含一段由`std::unique_ptr`管理的堆内存作为缓冲区，以及读指针。

- `WriteState`包含一段由`std::unique_ptr`管理的堆内存作为缓冲区，以及写指针。由于HTTP的写响应过程包含写请求头和写请求体（文件），因此`WriteState`还包含客户端所请求的文件描述符和状态变量。

## Server

```cpp
class Server {
public:
    explicit Server(ServerConfig  _config) : config(std::move(_config)), worker_thread_pool(config.thread_num, config.threadpool_max_request_num), \
    timer_heap(config.timerheap_default_time_slot)
    {
        listen_sock = socket(PF_INET, SOCK_STREAM, 0);
        if (listen_sock == -1) {
            throw std::runtime_error("socket() err");
        }

        if (setsockopt(listen_sock, SOL_SOCKET, SO_REUSEADDR, &config.op_so_reuseaddr, sizeof(config.op_so_reuseaddr))) {
            throw std::runtime_error("setsockopt() err");
        }

        sockaddr_in serv_addr{};
        serv_addr.sin_family = AF_INET;
        serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
        serv_addr.sin_port = htons(std::stoi(config.port));

        if (bind(listen_sock, (sockaddr*)&serv_addr, sizeof(serv_addr)) == -1) {
            throw std::runtime_error("bind() err");
        }

        if (listen(listen_sock, config.listen_queue_len) == -1) {
            throw std::runtime_error("listen() err");
        }

        if (pipe(sig_pipe) < 0) {
            throw std::runtime_error("pipe() err");
        }

        request_handlers = std::vector<RequestHandler>(config.max_fd);
        timer_tasks = std::vector<TimerTask>(config.max_fd);
    }
    Server(Server&) = delete;
    Server(Server&&) = delete;
    Server& operator=(Server&) = delete;
    Server& operator=(Server&&) = delete;
    ~Server();

    void run();
private:
    using EpollSigHandlerReturnType = struct {bool stop; bool timeout;};

    ServerConfig config;

    int listen_sock;
    static int sig_pipe[2];
    char sig_buf[4];
    TimerHeap timer_heap;
    std::vector<RequestHandler> request_handlers;
    std::vector<TimerTask> timer_tasks;

    static int fd_set_non_blocking(int fd);
    static void add_fd_to_epoll(int epoll_fd, int fd, uint32_t events);

    static void sig_handler(int sig);
    static void add_sig(int sig);
    [[nodiscard]] EpollSigHandlerReturnType epoll_sig_handler();

    void timer_heap_ticker();

    ThreadPool worker_thread_pool;
};
```

`Server`为最外层的服务器类，包含一个工作线程池，一个`RequestHandler`数组。`Server`类在构造时，首先执行服务器编程必备的`socket`、`listen`、`bind`三件套，然后在`request_handlers`数组中构造`config.max_fd`个`RequestHandler`对象，与`config.max_fd`个文件描述符一一对应。当文件描述符fd accept了一个新的连接时，服务器会调用`request_handlers[fd].reset()`函数重置用于这个文件描述符的`RequestHandler`对象，使其做好处理新连接的准备。

# 流程

## 主线程

主线程将监听socket的可读事件注册到epoll当中，然后循环调用`epoll_wait`函数获取就绪事件：

- 如果监听到监听socket的可读事件，则`accept`一个新连接，建立连接之后，socket并不一定处于可读状态（客户端还未发送数据），如果这个时候调用读处理函数，工作线程很有可能读不到内容直接返回，白白浪费CPU资源。因此，我们在这个地方直接将新连接的可读事件注册到epoll当中即可。

```cpp
add_fd_to_epoll(epoll_fd, conn_sock, EPOLLET | EPOLLONESHOT | EPOLLIN);
fd_set_non_blocking(conn_sock);
```

- 如果监听到连接socket的可读或可写事件，则设置`RequestHandler`的状态为`READ`或`WRITE`，然后加入线程池。

```cpp
if (ep_events[i].events & EPOLLIN) {
    request_handlers[event_fd].task_type = RequestHandler::READ;
} else if (ep_events[i].events & EPOLLOUT) {
    request_handlers[event_fd].task_type = RequestHandler::WRITE;
}

worker_thread_pool.append(&request_handlers[event_fd]);
```

## 工作线程

### 可读事件处理

- 在ET边沿触发模式下，每个可读或可写事件只会被触发一次，因此，事件处理函数需要循环读或写socket，直到没有东西可读/可写（返回EAGAIN / EWOULDBLOCK），或者连接被关闭（返回0）为止。上述循环位于`read_socket_to_buf`和`write_buf_to_socket`函数中。

- TCP是基于流的协议，这就意味着一次完整的读操作之后，缓冲区内有可能包含一个或多个HTTP请求，也有可能只包含一个请求的一部分。因此，我们也需要循环解析缓冲区的内容，并针对每个请求发送响应报文。

```cpp
void RequestHandler::process_read() {
    if (!read_socket_to_buf()) { // 读socket，直到没有东西可读
        return;
    }

    // loop to deal with possible multiple requests in read buffer
    while (true) {
        if (!parse_one_request()) { // 解析一个请求，并将响应报文写到写缓冲区中
            return;
        }

        if (!write_buf_to_socket()) { // 发送写缓冲区内的所有内容
            return;
        }
    }
}
```

问题来了。前面说到，在`accept`一个连接之后连接不一定可读，应该注册可读事件而不是直接执行读处理函数，否则工作线程可能读不到东西。那在这个地方，解析完一个HTTP请求之后，是否应该认为连接不一定可读，从而需要可读事件呢？

答案是否定的。socket的可读可写条件如下：

> 当如下任一情况发生时，会产生套接字的可读事件：
> 
> - 该套接字的接收缓冲区中的数据字节数大于等于套接字接收缓冲区低水位标记的大小；
> - 该套接字的读半部关闭（也就是收到了FIN），对这样的套接字的读操作将返回0（也就是返回EOF）；
> - 该套接字是一个监听套接字且已完成的连接数不为0；
> - 该套接字有错误待处理，对这样的套接字的读操作将返回-1。
> 
> 当如下任一情况发生时，会产生套接字的可写事件：
> 
> - 该套接字的发送缓冲区中的可用空间字节数大于等于套接字发送缓冲区低水位标记的大小；
> - 该套接字的写半部关闭，继续写会产生SIGPIPE信号；
> - 非阻塞模式下，connect返回之后，该套接字连接成功或失败；
> - 该套接字有错误待处理，对这样的套接字的写操作将返回-1。

可读和可写条件的后三条都是异常情况，我们只关注第一条。socket可读的条件是接收缓冲区的数据大于低水位标记，在`accept`之后，这个条件不一定会满足，因为我们不知道客户端在连接成功后到服务器调用`read`之前这段时间内会写入多少东西，因此服务器需要注册可读事件而不是直接读。然而，可写的条件是发送缓冲区的**可用**空间大于低水位标记，在第一次执行完`parse_one_request()`之后，这个条件是满足的。

因此，我们先尝试进行发送，如果遇到发送失败的情况，再注册可写事件然后退出。

### 可写事件处理

同样地，可写事件处理函数的第一步操作是尝试将写缓冲区的内容全部发送，如果发送失败，则注册可写事件并退出。

如果写缓冲区的内容全部清空了，是否意味着我们可以退出函数了？

还没。我们当前正在处理的可写事件，有可能是在`process_read()`的while循环中注册的。如果这个可写事件没有被注册，那么`process_read()`函数还会继续执行`parse_one_request()`，解析剩下的报文。这就意味着，在`process_write()`执行时，读缓冲区中同样可能留存有未被解析的报文，我们需要写一个同样的while循环进行解析。

```cpp
void RequestHandler::process_write() {
    // write buffer and file
    if (!write_buf_to_socket()) {
        return;
    }

    // loop to parse read buffer and write
    while (true) {
        if (!parse_one_request()) {
            return;
        }

        if (!write_buf_to_socket()) {
            return;
        }
    }
}
```

# 长连接的实现

HTTP支持长连接，客户端的HTTP请求报文中，如果`Connection`字段为`keep-alive`，表示客户端希望保持这个TCP连接不被断开，在发送下一次请求的时候仍然使用这个连接。在服务器当中，需要判断客户端是否请求建立长连接，然后在读缓冲区为空的时候执行不同的操作。

```cpp
bool RequestHandler::parse_one_request() {
    if (read_state.read_idx == 0) {
        if (keep_alive) {
            mod_fd_to_epoll(epoll_fd, conn_fd, EPOLLET | EPOLLIN | EPOLLONESHOT); // connection:keep-alive
        } else {
            close_conn(conn_fd); // connection:closed
        }

        return false;
    }

    ...
}
```

如果不是长连接，则在读缓冲区为空时关闭连接；否则重新向epoll注册可读事件。

# 超时处理

在使用HTTP长连接时，如果客户端长时间没有关闭连接，那么服务器应当主动关闭连接，避免资源被过多的非活跃连接占用。即使是在使用短连接的情况下，也有可能由于各种原因出现非活跃连接。为此，需要为每个活跃的连接设置一个定时器，在定时器超时时，服务器主动断开连接。

## 数据结构——惰性重置

定时器的数据结构应该如何设计？我们采用一种常见的定时器——堆定时器。堆定时器应实现如下接口：

- 插入一个定时任务

- 取出超时时刻最早的定时任务

- 重置一个定时任务的超时时刻

显然前两个接口可以通过`std::priority_queue`或`std::multiset`实现，只需要将超时时刻的先后作为比较两个定时任务的标准。

> P.S. 之所以需要用`std::multiset`而不是`std::set`，是因为同一个超时时刻有可能对应多个定时任务。

现在考虑第三个接口的实现。首先考虑一种最简单的实现方式：直接修改定时器任务对象的超时时刻。然而，我们首先要问一个问题：当元素被插入`std::priority_queue`或`std::multiset`之后，是否还能改变元素的key值？

- `std::priority_queue`的实现是二叉堆，对于元素的排序操作发生在插入元素和取出根节点的时候。如果我们修改了根节点的key值，这个根节点的位置不会发生变化，还是会被首先取出；如果我们更改了非根节点的key值（使它的超时时刻变得最早），在下一次调用pop的时候也无法取出这个节点。

- `std::multiset`的实现是红黑树，其中所有节点的位置在每一个时刻都是确定的，且取决于每一个节点的key。如果在某一时刻修改了一个节点的key，将会破坏红黑树的结构。

可见，无论采用何种STL容器（适配器），定时器的超时时刻都不能够被随意修改。为了解决这个问题，本项目设计了一种惰性重置的方式。

```cpp
// 基于multiset
void TimerTask::reset() {
    is_reset = true;
    new_expired_time = std::time(nullptr) + timeout;

std::time_t TimerHeap::tick() {
    ...

    while (!s.empty() && (*s.begin())->get_expired_time() <= std::time(nullptr)) {
        if ((*s.begin())->is_valid()) {
            if ((*s.begin())->is_reset) {
                (*s.begin())->is_reset = false;
                (*s.begin())->expired_time = (*s.begin())->new_expired_time;
                auto ptr = *s.begin();
                s.erase(s.begin());
                s.insert(ptr);
            } else {
                (*s.begin())->callback();
                s.erase(s.begin());
            }
        } else {
            s.erase(s.begin());
        }
    }

    ...
}
```

定时任务包含`is_reset`字段和新的超时时刻`new_expired_time`字段，当需要充值定时任务时，不改变原始的`expired_time`字段，而是将`is_reset`置为`true`，并将新的超时时刻写入`new_expired_time`变量。当定时器进行`tick`时，在取出队首元素之后会判断是否进行了重置，若是，则修改其时间之后重新插入堆中。

能否在`TimerTask::reset`函数当中执行删除和重新插入的操作？还是不行。注意到`tick`函数中我是用迭代器删除了首部元素。如果要在`reset`函数中进行删除，元素不一定在首部，只能通过key值删除。然而一个key值（超时时刻）会对应多个定时任务，我们无法通过一行代码直接删除目标元素，还需要遍历相同key的不同定时任务进行查找。

如果采用的是`std::priority_queue`就更不用说了，根本没有删除中间元素的接口。

因此，惰性删除是较好的选择。

## std::multiset or std::priority_queue

使用惰性删除时，由于只需要取首元素，因此`std::priority_queue`和`std::multiset`均可。

## 启动和reset定时器的位置？

- 启动定时器：accept时

- reset定时器：监听到可读/可写事件时

一种细粒度的实现是每次读/写成功的时候都重启定时器，但是会过于消耗资源，并且提高编程复杂度。本项目的定时器粒度较粗。

# 遇到的其他问题

## 注册EPOLLONESHOT的作用？

保证一个socket只由一个工作线程处理。

如果不注册`EPOLLONESHOT`，在工作线程读一个socket的过程中，即使采用了ET边沿触发，主线程仍然有可能再次触发`EPOLLIN`事件，并交由另一个工作线程处理，造成两个线程同时读一个socket的局面。

有人做了实验证明，ET模式，不注册`EPOLLONESHOT`，在读缓冲区溢出的时候会重复注册可读事件：[EPOLLONESHOT及其引发的EPOLL在ET能被多次触发吗？ - 掘金 (juejin.cn)](https://juejin.cn/post/7102252724212727844)

## RequestHandler应当静态分配还是动态分配？

在项目的初始版本中，由于担心静态分配的`RequestHandler`占用太多内存，我采用的是动态分配的方式。每一个请求来了之后`make_unique`一个`RequestHandler`对象，然后将指针`std::move`给工作线程。然后问题来了：第一，缓冲区怎么设计？这时，一个`RequestHandler`对象并不是对应一个连接，而是对应一个连接的一次事件触发，在`RequestHandler`对象析构后，有可能读缓冲区还有不完整的报文，需要等待下一次可读事件。为此，需要在`Server`类中维护一个读写缓冲区集合。由于错误估计了内存占用，对于读写缓冲区，我仍然采用动态分配。。于是又涉及到如何线程安全的读写STL容器，以及确定缓冲区的删除时机的问题。在删除缓冲区时，又涉及到主线程和工作线程的通信，为此我还设计了一个通信队列……除此之外，由于定时器我也是动态分配的，所以又要走一套上面的流程。。

编程上的复杂性还不够，动态分配`RequestHandler`还会带来`RequestHandler`的构造析构开销。

经过了上述尝试，我又将`RequestHandler`改为静态分配了。事实上，现有实现中，`RequestHandler`对象占用空间不超过5KB，数组长度为65536，占用内存仅有320MB，这对于一台Web服务器来说完全可以接收。

## HTTP的格式问题

在DEBUG的过程中，我看到网上好一些博客都说HTTP的`Content-Length`字段是八进制的，为此还懊恼了好久。后来调着调着发现用八进制结果也不对啊（浏览器在显示完我的页面之后还会一直转圈）。

无奈只能查RFC，原文是这么说的：

> [14.13](https://datatracker.ietf.org/doc/html/rfc2616#section-14.13) Content-Length
> 
>    The Content-Length entity-header field indicates the size of the
>    entity-body, **in decimal number of OCTETs**, sent to the recipient or,
>    in the case of the HEAD method, the size of the entity-body that
>    would have been sent had the request been a GET.

博主们，你们只知道OCT是八进制，不知道decimal是十进制吗？？再说他也不是OCT，而是OCTETs啊，看看维基百科怎么说的：

> The **octet** is a [unit of digital information](https://en.wikipedia.org/wiki/Units_of_information "Units of information") in [computing](https://en.wikipedia.org/wiki/Computing "Computing") and [telecommunications](https://en.wikipedia.org/wiki/Telecommunication "Telecommunication") that consists of eight [bits](https://en.wikipedia.org/wiki/Bit "Bit"). The term is often used when the term *[byte](https://en.wikipedia.org/wiki/Byte "Byte")* might be ambiguous, as the byte has historically been used for storage units of a variety of sizes.

OCTET就是8比特，就是字节。也就是说，`Content-Length`就是用十进制表示的字节数。。建议大家遇到这种协议、规范、定义类的东西最好查原文，不要看博客。

# 参考资料

> - [1] 游双. Linux高性能服务器编程
