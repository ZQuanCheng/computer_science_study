# C++网络编程



## TCP连接的三次握手与四次挥手

### 建立连接：TCP三次握手

在没进行连接的情况下，客户端的TCP状态处于`CLOSED`状态，服务端的TCP处于`CLOSED`（未开启监听）或者`LISTEN`（开启监听）状态。

TCP中，服务端与客户端建立连接的过程如下：

- 客户端主动发起连接（在socket编程中则为调用`connect`函数），此时客户端向服务端发送一个SYN包
- 这个SYN包可以看作是一个小数据包，不过其中没有任何实际数据，仅有诸如TCP首部和TCP选项等协议包必须数据。可以看作是客户端给服务端发送的一个信号
- 此时客户端状态从`CLOSED`切换为`SYN_SENT`
- 服务端收到SYN包，并返回一个针对该SYN包的响应包（ACK包）和一个新的SYN包。
- 在socket编程中，服务端能收到SYN包的前提是，服务端已经调用过`listen`函数使其处于监听状态（也就是说，其必须处于`LISTEN`状态），并且处于`accept`函数等待连接的阻塞状态。
- 此时服务端状态从`LISTEN`切换为`SYN_RCVD`
- 客户端收到服务端发来的两个包，并返回针对新的SYN包的ACK包。
- 此时客户端状态从`SYN_SENT`切换至`ESTABLISHED`，该状态表示可以传输数据了。
- 服务端收到ACK包，成功建立连接，`accept`函数返回出客户端套接字。
- 此时服务端状态从`SYN_RCVD`切换至`ESTABLISHED`

###  

### 收发数据

当连接建立之后，就可以通过客户端套接字进行收发数据了。

### 断开连接：TCP四次挥手

在收发数据之后，如果需要断开连接，则断开连接的过程如下：

- 双方中有一方（假设为A，另一方为B）主动关闭连接（调用`close`，或者其进程本身被终止等情况），则其向B发送FIN包

- 此时A从`ESTABLISHED`状态切换为`FIN_WAIT_1`状态

- B接收到FIN包，并发送ACK包

- 此时B从`ESTABLISHED`状态切换为`CLOSE_WAIT`状态

- A接收到ACK包

- 此时A从`FIN_WAIT_1`状态切换为`FIN_WAIT_2`状态

- 一段时间后，B调用自身的`close`函数，发送FIN包

- 此时B从`CLOSE_WAIT`状态切换为`LAST_ACK`状态

- A接收到FIN包，并发送ACK包

- 此时A从`FIN_WAIT_2`状态切换为`TIME_WAIT`状态

- B接收到ACK包，关闭连接

- 此时B从`LAST_ACK`状态切换为`CLOSED`状态

- A等待一段时间（两倍的最长生命周期）后，关闭连接

- 此时A从`TIME_WAIT`状态切换为`CLOSED`状态

  

  ## 常用 Berkeley Sockets API 一览表

  |   函数名称    |                 函数简单描述                 |              附加说明              |
  | :-----------: | :------------------------------------------: | :--------------------------------: |
  |    socket     |             创造某种类型的套接字             |                                    |
  |     bind      | 将一个 socket 绑定到一个 ip 与端口的二元组上 |                                    |
  |    listen     |          将一个 socket 变为侦听状态          |                                    |
  |    connect    |            试图建立一个 TCP 连接             |           一般用于客户端           |
  |    accept     |               尝试接收一个连接               |           一般用于服务端           |
  |     send      |          通过一个 socket  发送数据           |                                    |
  |     recv      |           通过一个socket 收取数据            |                                    |
  |    select     |      判断一组 socket 上的读写和异常事件      |                                    |
  | gethostbyname |             通过域名获取机器地址             |                                    |
  |     close     |   关闭一个套接字，回收该 socket 对应的资源   | Windows 系统中对应的是 closesocket |
  |   shutdown    |            关闭 socket 收或发通道            |                                    |
  |  setsockopt   |              设置一个套接字选项              |                                    |
  |  getsockopt   |              获取一个套接字选项              |                                    |

  对于服务器，其通信流程一般有如下步骤：

  ```
  1. 调用 socket 函数创建 socket（侦听socket）
  2. 调用 bind 函数 将 socket绑定到某个ip和端口的二元组上
  3. 调用 listen 函数 开启侦听
  4. 当有客户端请求连接上来后，调用 accept 函数接受连接，产生一个新的 socket（客户端 socket）
  5. 基于新产生的 socket 调用 send 或 recv 函数开始与客户端进行数据交流
  6. 通信结束后，调用 close 函数关闭侦听 socket
  ```

  对于客户端，其通信流程一般有如下步骤：

  ```
  1. 调用 socket函数创建客户端 socket
  2. 调用 connect 函数尝试连接服务器
  3. 连接成功以后调用 send 或 recv 函数开始与服务器进行数据交流
  4. 通信结束后，调用 close 函数关闭侦听socket
  ```

  上述流程可以绘制成如下图示：

  ![](F:\C++网络编程实战训练营\7th\20181213192725.png)

## socket函数

socket函数用于创建套接字。其实更严谨的讲是创建一个**套接字描述符**（以下简称sockfd）。

套接字描述符本质上类似于文件描述符，文件通过文件描述符供程序进行读写，而套接字描述符本质上也是提供给程序可以对其缓存区进行读写，程序在其写缓存区写入数据，写缓存区的数据通过网络通信发送至另一端的相同套接字的读缓存区，另一端的程序使用相同的套接字在其读缓存区上读取数据，这样便完成了一次网络数据传输。

而`socket`函数的参数便是用于设置这个套接字描述符的属性。

该函数的原型如下：

```C++
#include <sys/socket.h>
int socket(int family, int type, int protocol);
```

### family参数

该参数指明要创建的sockfd的协议族，一般比较常用的有两个：

- `AF_INET`：IPv4协议族
- `AF_INET6`：IPv6协议族

###  

### type参数

该参数用于指明套接字类型，具体有：

- `SOCK_STREAM`：**字节流套接字**，适用于TCP或SCTP协议
- `SOCK_DGRAM`：**数据报套接字**，适用于UDP协议
- `SOCK_SEQPACKET`：有序分组套接字，适用于SCTP协议
- `SOCK_RAW`：原始套接字，适用于绕过传输层直接与网络层协议（IPv4/IPv6）通信



### protocol参数

该参数用于指定协议类型。

如果是TCP协议的话就填写`IPPROTO_TCP`，UDP和SCTP协议类似。

也可以直接填写0，这样的话则会默认使用`family`参数和`type`参数组合制定的默认协议

（参照上面type参数的适用协议）

### 返回值

`socket`函数在成功时会返回套接字描述符，失败则返回-1。

失败的时候可以通过输出`errno`来详细查看具体错误类型。

### 关于errno

通常一个内核函数运行出错的时候，它会定义全局变量`errno`并赋值。

当我们引入`errno.h`头文件时便可以使用这个变量。并利用这个变量查看具体出错原因。

一共有两种查看的方法：

- 直接输出`errno`，根据输出的错误码进行Google搜索解决方案
- 当然也可以直接翻man手册
- 借助`strerror()`函数，使用`strerror(errno)`得到一个具体描述其错误的字符串。一般可以通过其描述定位问题所在，实在不行也可以拿这个输出去Google搜索解决方案

## bind函数

bind函数与两个socket有关，一个负责侦听是否有外来连接请求   如果有 创建第二个socket 进行数据传输

bind函数用于将套接字与一个`ip::port`绑定。或者更应该说是**把一个本地协议地址赋予一个套接字**。

该函数的原型如下：

```c++
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);
```

这个函数的参数表比较简单：第一个是套接字描述符，第二个是**套接字地址结构体**，第三个是套接字地址结构体的长度。其含义就是将第二个的套接字地址结构体赋给第一个的套接字描述符所指的套接字。

## 套接字地址结构体

在bind函数的参数表中出现了一个名为`sockaddr`的结构体，这个便是用于存储将要赋给套接字的地址结构的**通用套接字地址结构**。其定义如下：

```C++
#include <sys/socket.h>
struct sockaddr
{
    uint8_t     sa_len;
    sa_family_t sa_family;      // 地址协议族
    char        sa_data[14];    // 地址数据
};
```

当然，我们一般不会直接使用这个结构来定义套接字地址结构体，而是使用更加特定化的**IPv4套接字地址结构体**或**IPv6套接字地址结构体**。这里只讲前者。

IPv4套接字地址结构体的定义如下：

```C++
#include <netinet/in.h>
struct in_addr
{
    in_addr_t       s_addr;         // 32位IPv4地址
};
struct sockaddr_in
{
    uint8_t         sin_len;        // 结构长度，非必需
    sa_family_t     sin_family;     // 地址族，一般为AF_****格式，常用的是AF_INET
    in_port_t       sin_port;       // 16位TCP或UDP端口号
    struct in_addr  sin_addr;       // 32位IPv4地址
    char            sin_zero[8];    // 保留数据段，一般置零
};
```

值得注意的是，一般而言一个`sockaddr_in`结构对我们来说有用的字段就三个：

- `sin_family`
- `sin_addr`
- `sin_port`

可以看到在第一节的代码中也是只赋值了这三个成员：

```C++
#define DEFAULT_PORT 16555
// ...
struct sockaddr_in servaddr;    // 定义一个IPv4套接字地址结构体
// ...
bzero(&servaddr, sizeof(servaddr));    // 将该结构体的所有数据置零
servaddr.sin_family = AF_INET;    // 指定其协议族为IPv4协议族
servaddr.sin_addr.s_addr = htonl(INADDR_ANY);    // 指定IP地址为通配地址
servaddr.sin_port = htons(DEFAULT_PORT);    // 指定端口号为16555
// 调用bind，注意第二个参数使用了类型转换，第三个参数直接取其sizeof即可
if (-1 == bind(sockfd, (struct sockaddr*)&servaddr, sizeof(servaddr)))
{
    printf("Bind error(%d): %s\n", errno, strerror(errno));
    return -1;
}
```

其中有三个细节需要注意：

- 在指定IP地址的时候，一般就是使用像上面那样的方法指定为通配地址，此时就交由内核选择IP地址绑定。指定特定IP的操作在讲connect函数的时候会提到。
- 在指定端口的时候，可以直接指定端口号为0，此时表示端口号交由内核选择（也就是进程不指定端口号）。但一般而言对于服务器来说，不指定端口号的情况是很罕见的，因为服务器一般都需要暴露一个端口用于让客户端知道并作为连接的参数。
- 注意到不管是赋值IP还是端口，都不是直接赋值，而是使用了类似`htons()`或`htonl()`的函数，这便是**字节排序函数**。

## 字节排序函数

首先，不同的机子上对于多字节变量的字节存储顺序是不同的，有**大端字节序**和**小端字节序**两种。

那这就意味着，将机子A的变量原封不动传到机子B上，其值可能会发生变化（本质上数据没有变化，但如果两个机子的字节序不一样的话，解析出来的值便是不一样的）。这显然是不好的。

故我们需要引入一个通用的规范，称为**网络字节序**。引入网络字节序之后的传递规则就变为：

- 机子A先将变量由自身的字节序转换为网络字节序
- 发送转换后的数据
- 机子B接到转换后的数据之后，再将其由网络字节序转换为自己的字节序

其实就是很常规的**统一标准中间件**的做法。

在Linux中，位于`<netinet/in.h>`中有四个用于主机字节序和网络字节序之间相互转换的函数：

```C++
#include <netinet/in.h>
uint16_t htons(uint16_t host16bitvalue);    //host to network, 16bit
uint32_t htonl(uint32_t host32bitvalue);    //host to network, 32bit
uint16_t ntohs(uint16_t net16bitvalue);     //network to host, 16bit
uint32_t ntohl(uint32_t net32bitvalue);     //network to host, 32bit
```

## listen函数

listen函数的作用就是开启套接字的监听状态，也就是将套接字从`CLOSE`状态转换为`LISTEN`状态。

该函数的原型如下：

```text
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```

其中，`sockfd`为要设置的套接字，`backlog`为服务器处于`LISTEN`状态下维护的队列长度和的最大值。

### 关于backlog

这是一个**可调参数**。

其意义为，服务器套接字处于`LISTEN`状态下所维护的**未完成连接队列（SYN队列）**和**已完成连接队列(Accept队列)**的长度和的最大值。

↑ 这个是原本的意义，现在的`backlog`仅指**Accept队列的最大长度**，SYN队列的最大长度由系统的另一个变量决定。

这两个队列用于维护与客户端的连接，其中：

- 客户端发送的SYN到达服务器之后，服务端返回SYN/ACK，并将该客户端放置SYN队列中（第一次+第二次握手）
- 当服务端接收到客户端的ACK之后，完成握手，服务端将对应的连接从SYN队列中取出，放入Accept队列，等待服务器中的accept接收并处理其请求（第三次握手）

###  

### backlog调参

`backlog`是由程序员决定的，不过最后的队列长度其实是`min(backlog, /proc/sys/net/core/somaxconn , net.ipv4.tcp_max_syn_backlog )`，后者直接读取对应位置文件就有了。

不过由于后者是可以修改的，故这里讨论的`backlog`实际上是这两个值的最小值。

至于如何调参，可以参考这篇博客：

[https://ylgrgyq.github.io/2017/05/18/tcp-backlog/](https://link.zhihu.com/?target=https%3A//ylgrgyq.github.io/2017/05/18/tcp-backlog/)

事实上`backlog`仅仅是与**Accept队列的最大长度**相关的参数，实际的队列最大长度视不同的操作系统而定。例如说MacOS上使用传统的Berkeley算法基于`backlog`参数进行计算，而Linux2.4.7上则是直接等于`backlog+3`。

### 返回值

若成功则返回0，否则返回-1并置相应的`errno`。

##  connect函数

该函数用于客户端跟绑定了指定的ip和port并且处于`LISTEN`状态的服务端进行连接。

在调用connect函数的时候，调用方（也就是客户端）便会主动发起TCP三次握手。

该函数的原型如下：

```C++
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);
```

其中第一个参数为客户端套接字，第二个参数为用于指定服务端的ip和port的套接字地址结构体，第三个参数为该结构体的长度。

操作上比较类似于服务端使用bind函数（虽然做的事情完全不一样），唯一的区别在于指定ip这块。服务端调用bind函数的时候无需指定ip，但客户端调用connect函数的时候则需要指定服务端的ip。

在客户端的代码中，令套接字地址结构体指定ip的代码如下：

```C++
inet_pton(AF_INET, SERVER_IP, &servaddr.sin_addr);
```

这个就涉及到**ip地址的表达格式与数值格式相互转换**的函数。

## IP地址格式转换函数

IP地址一共有两种格式：

- 表达格式：也就是我们能看得懂的格式，例如`"192.168.19.12"`这样的字符串
- 数值格式：可以存入套接字地址结构体的格式，数据类型为整型

显然，当我们需要将一个IP赋进套接字地址结构体中，就需要将其转换为数值格式。

在`<arpa/inet.h>`中提供了两个函数用于IP地址格式的相互转换：

```text
#include <arpa/inet.h>
int inet_pton(int family, const char *strptr, void *addrptr);
const char *inet_ntop(int family, const void *addrptr, char *strptr, size_t len);
```

其中：

- `inet_pton()`函数用于将IP地址从表达格式转换为数值格式

- 第一个参数指定协议族（`AF_INET`或`AF_INET6`）

- 第二个参数指定要转换的表达格式的IP地址

- 第三个参数指定用于存储转换结果的指针

- 对于返回结果而言：

- - 若转换成功则返回1
  - 若表达格式的IP地址格式有误则返回0
  - 若出错则返回-1

- `inet_ntop()`函数用于将IP地址从数值格式转换为表达格式

- 第一个参数指定协议族

- 第二个参数指定要转换的数值格式的IP地址

- 第三个参数指定用于存储转换结果的指针

- 第四个参数指定第三个参数指向的空间的大小，用于防止缓存区溢出

- - 第四个参数可以使用预设的变量：

```text
#include <netinet/in.h>
#define INET_ADDRSTRLEN    16  // IPv4地址的表达格式的长度
#define INET6_ADDRSTRLEN 46    // IPv6地址的表达格式的长度
```

- 对于返回结果而言

- - 若转换成功则返回指向返回结果的指针
  - 若出错则返回NULL

其中connect函数会出错的几种情况：

- 若客户端在发送SYN包之后长时间没有收到响应，则返回`ETIMEOUT`错误

- - 一般而言，如果长时间没有收到响应，客户端会重发SYN包，若超过一定次数重发仍没响应的话则会返回该错误

  - 可能的原因是目标服务端的IP地址不存在

    

- 若客户端在发送SYN包之后收到的是RST包的话，则会立刻返回`ECONNREFUSED`错误

- - 当客户端的SYN包到达目标机之后，但目标机的对应端口并没有正在`LISTEN`的套接字，那么目标机会发一个RST包给客户端

  - 可能的原因是目标服务端没有运行，或者没运行在客户端知道的端口上

    

- 若客户端在发送SYN包的时候在中间的某一台路由器上发生ICMP错误，则会发生`EHOSTUNREACH`或`ENETUNREACH`错误

- - 事实上跟处理未响应一样，为了排除偶然因素，客户端遇到这个问题的时候会保存内核信息，隔一段时间之后再重发SYN包，在多次发送失败之后才会报错
  - 路由器发生ICMP错误的原因是，路由器上根据目标IP查找转发表但查不到针对目标IP应该如何转发，则会发生ICMP错误
  - 可能的原因是目标服务端的IP地址不可达，或者路由器配置错误，也有可能是因为电波干扰等随机因素导致数据包错误，进而导致路由无法转发



由于connect函数在发送SYN包之后就会将自身的套接字从`CLOSED`状态置为`SYN_SENT`状态，故当connect报错之后需要主动将套接字状态置回`CLOSED`。此时需要通过调用close函数主动关闭套接字实现。

故原版的客户端代码需要做一个修改：

```C++
if (-1 == connect(sockfd, (struct sockaddr*)&servaddr, sizeof(servaddr)))
{
    printf("Connect error(%d): %s\n", errno, strerror(errno));
    close(sockfd);        // 新增代码，当connect出错时需要关闭套接字
    return -1;
}
```

