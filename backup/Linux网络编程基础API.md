# 前言
内容来源：本文章为阅读《Linux高性能服务器编程》第五章.Linux网络编程基础API所记录的部分笔记。
摘要：这篇文章围绕 Linux 网络编程基础 API 展开，介绍了 socket 地址（字节序、通用 / 专用地址结构、IP 转换）、socket 创建 / 命名 / 监听 / 接受 / 发起连接的核心函数（socket/bind/listen/accept/connect）、连接关闭的两种方式（close/shutdown），以及 TCP（recv/send、带外数据）和 UDP（recvfrom/sendto）的数据读写接口。
# 1.socket地址API
## 1.1 主机字节序和网络字节序
一、字节序的核心概念
1. **定义**：多字节数据（如 32 位整数）在内存中的存储顺序，分为两种：
    - **大端字节序**：高位字节存在内存低地址，低位字节存在高地址；
    - **小端字节序**：高位字节存在内存高地址，低位字节存在低地址。
2. **影响**：不同字节序的主机直接传递数据，会导致接收端解析错误。

二、主机字节序与网络字节序
1. **主机字节序**：当前机器的字节序（现代 PC 大多是小端）；
2. **网络字节序**：网络传输的统一标准（强制为大端），解决不同主机字节序不兼容的问题。

三、字节序的适用场景
- 跨主机通信：必须将数据转为网络字节序（大端）后传输，接收端再转为主机字节序；
- 同一主机的跨语言进程通信：比如 C（小端）与 Java（JVM 默认大端）通信，也需处理字节序。

四、Linux 下的字节序转换函数
头文件：`<netinet/in.h>`
4 个核心函数（作用是主机字节序 ↔ 网络字节序）：

| 函数名     | 含义（host ↔ network） | 处理数据类型           | 常用场景     |
| ------- | ------------------ | ---------------- | -------- |
| `htonl` | 长整型（32 位）主机→网络     | `unsigned long`  | 转换 IP 地址 |
| `htons` | 短整型（16 位）主机→网络     | `unsigned short` | 转换端口号    |
| `ntohl` | 长整型（32 位）网络→主机     | `unsigned long`  | 解析 IP 地址 |
| `ntohs` | 短整型（16 位）网络→主机     | `unsigned short` | 解析端口号    |
## 1.2 通用socket地址
一、基础 Socket 地址结构：`struct sockaddr`
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260105150701.png)
**定义与成员**：
    - 头文件：`<bits/socket.h>`；
    - 成员：
        - `sa_family`：地址族（与协议族对应，如`AF_INET`对应 IPv4）；
        - `sa_data[14]`：存放 Socket 地址值，但仅 14 字节，空间不足。
二、协议族与地址族的关系
- 协议族（`PF_*`）与地址族（`AF_*`）一一对应，且值完全相同（通常混用）；
- 常见对应关系：![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260105150801.png)
三、`struct sockaddr`的缺陷：空间不足
不同协议族的地址值长度超过`sa_data[14]`的容量：
- `PF_UNIX`：地址是路径名（最长 108 字节）；
- `PF_INET`：地址是 “16 位端口 + 32 位 IPv4”（共 6 字节，虽能放下，但其他协议不够）；
- `PF_INET6`：地址是 “端口 + 流标识 + IPv6 + 范围 ID”（共 26 字节）。
四、改进的通用 Socket 地址结构：`struct sockaddr_storage`
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260105151013.png)
为解决空间不足问题，Linux 新增该结构：
1. **定义与特点**：
    - 头文件：`<bits/socket.h>`；
    - 成员：
        - `sa_family`：地址族；
        - `__ss_align`：保证内存对齐；
        - `__ss_padding`：填充字段，总空间 128 字节（足够容纳所有协议族的地址）。
2. **作用**：提供足够大的空间存放任意协议族的地址，且内存对齐。
## 1.3 专用socket地址
一、专用 Socket 地址结构的设计原因
通用结构（`sockaddr`/`sockaddr_storage`）操作不便（比如设置 IP / 端口需要位操作），因此 Linux 为**每个协议族设计了专用地址结构**，字段更直观、易操作。
二、各协议族的专用地址结构
1. UNIX 本地域协议族：`struct sockaddr_un`
	![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260105151622.png)
- 头文件：`<sys/un.h>`；
- 成员：
    - `sun_family`：地址族，固定为`AF_UNIX`；
    - `sun_path[108]`：本地通信的文件路径名（UNIX 域套接字通过文件路径标识通信端点）。
2. TCP/IP 协议族：IPv4 专用`struct sockaddr_in`
	![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260105151654.png)
- 对应协议：TCP/IPv4；
- 成员：
    - `sin_family`：地址族，固定为`AF_INET`；
    - `sin_port`：16 位端口号（需用`htons()`转为网络字节序）；
    - `sin_addr`：嵌套的`struct in_addr`结构体（存储 IPv4 地址）；
- 嵌套结构`struct in_addr`：
    - `s_addr`：32 位 IPv4 地址（需用`htonl()`/`inet_addr()`转为网络字节序）。
3. TCP/IP 协议族：IPv6 专用`struct sockaddr_in6`
	![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260105151724.png)
- 对应协议：TCP/IPv6；
- 成员：
    - `sin6_family`：地址族，固定为`AF_INET6`；
    - `sin6_port`：16 位端口号（需用`htons()`转为网络字节序）；
    - `sin6_flowinfo`：流信息，通常设为 0；
    - `sin6_addr`：嵌套的`struct in6_addr`结构体（存储 IPv6 地址）；
    - `sin6_scope_id`：范围 ID，实验阶段字段；
- 嵌套结构`struct in6_addr`：
    - `sa_addr[16]`：128 位 IPv6 地址（需转为网络字节序）。
三、专用结构的使用规则
所有专用地址结构（如`struct sockaddr_in`）在调用 Socket API（如`bind()`/`connect()`）时，**必须强转为通用结构`struct sockaddr*`**—— 因为 Socket 接口的地址参数类型固定为`struct sockaddr`。
## 1.4 IP地址转换函数
一、IP 地址转换的必要性
人们习惯用**可读性字符串**表示 IP（如 IPv4 的 “点分十进制”、IPv6 的 “十六进制”），但编程中需将其转为**网络字节序的整数形式**才能使用；记录日志时则需反向转换。
二、IPv4 专用转换函数（旧版）
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260105152251.png)
头文件：`<arpa/inet.h>`，仅支持 IPv4：

|函数名|功能|注意事项|
|---|---|---|
|`inet_addr`|点分十进制字符串 → 网络字节序整数|失败返回`INADDR_NONE`|
|`inet_aton`|点分十进制字符串 → 网络字节序整数（结果存入`struct in_addr`）|成功返回 1，失败返回 0；比`inet_addr`更安全（避免`INADDR_NONE`的歧义）|
|`inet_ntoa`|网络字节序整数 → 点分十进制字符串|内部用**静态变量**存储结果，不可重入（多次调用会覆盖之前的结果）|

三、`inet_ntoa`的不可重入性
由于`inet_ntoa`用静态变量存结果，多次调用会导致前一次的结果被覆盖：
运行
```c
// 示例：两次调用inet_ntoa，szValue1和szValue2会指向同一个静态内存
char* szValue1 = inet_ntoa("1.2.3.4");
char* szValue2 = inet_ntoa("10.194.71.60");
// 最终szValue1和szValue2都会输出"10.194.71.60"
```
四、通用转换函数（新版，支持 IPv4/IPv6）
头文件：`<arpa/inet.h>`，同时支持 IPv4 和 IPv6：
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260105152404.png)

| 函数名         | 功能               | 参数说明                                                                                                                                  |
| ----------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `inet_pton` | 字符串 IP → 网络字节序整数 | - `af`：地址族（`AF_INET`=IPv4，`AF_INET6`=IPv6）<br>- `src`：字符串 IP<br>- `dst`：存储结果的内存                                                       |
| `inet_ntop` | 网络字节序整数 → 字符串 IP | - `af`：地址族<br>- `src`：整数形式的 IP<br>- `dst`：存储字符串的缓冲区<br>- `cnt`：缓冲区大小（可用宏`INET_ADDRSTRLEN`（IPv4，16 字节）/`INET6_ADDRSTRLEN`（IPv6，46 字节）） |

关键结论
- 旧版函数（`inet_addr`/`inet_aton`/`inet_ntoa`）仅支持 IPv4，且`inet_ntoa`不可重入；
- 新版函数（`inet_pton`/`inet_ntop`）是更推荐的方式：支持 IPv4/IPv6，无不可重入问题，兼容性更好。
# 2.创建socket
一、`socket()`的核心定位
在 UNIX/Linux 中，**套接字（socket）是一种文件描述符**，遵循 “一切皆文件” 的设计：可以像操作文件一样对 socket 执行读、写、控制、关闭等操作。
二、`socket()`函数的定义与头文件
运行
```c
#include <sys/types.h>
#include <sys/socket.h>
int socket( int domain, int type, int protocol );
```
- 成功返回：**socket 对应的文件描述符**（非负整数）；
- 失败返回：`-1`，并设置`errno`（错误标识）。
三、`socket()`的三个参数详解
1. `domain`：协议族（底层协议类型）
作用：指定 socket 使用的**协议族**，决定了地址结构的类型。
常见取值：
- `PF_INET`：TCP/IPv4 协议族；
- `PF_INET6`：TCP/IPv6 协议族；
- `PF_UNIX`：UNIX 本地域协议族（用于本机进程通信）。
2. `type`：服务类型（套接字类型）
作用：指定 socket 的**通信类型**，决定了传输层协议的特性。
核心取值：
- `SOCK_STREAM`：流服务，对应**TCP 协议**（可靠、面向连接、有序）；
- `SOCK_DGRAM`：数据报服务，对应**UDP 协议**（不可靠、无连接、快速）。
**扩展标志（Linux 2.6.17 + 支持）**：
可通过 “按位与” 添加额外属性：
- `SOCK_NONBLOCK`：将 socket 设为**非阻塞模式**（默认是阻塞）；
- `SOCK_CLOEXEC`：`fork`创建子进程时，自动在子进程中关闭该 socket。
1. `protocol`：具体协议
作用：在前两个参数确定的协议集合中，选择具体协议。
- 通常设为`0`：表示使用`domain+type`对应的**默认协议**（因为前两个参数已唯一确定协议，比如`PF_INET+SOCK_STREAM`的默认协议是 TCP）；
- 特殊场景才需指定非 0 值（极少用）。
# 3.命名socket
 一、`bind()`的核心作用
- **定义**：将套接字（`sockfd`）与具体的`socket地址`绑定，相当于给套接字 “命名”；
- **场景**：
    - 服务器必须调用`bind()`：绑定 IP + 端口后，客户端才能通过该地址连接服务器；
    - 客户端通常不需要：由操作系统自动分配匿名地址。
二、`bind()`函数的定义与头文件
```c
#include <sys/types.h>
#include <sys/socket.h>
int bind( int sockfd, const struct sockaddr* my_addr, socklen_t addrlen );
//例如：ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
```
- 参数：
    - `sockfd`：待绑定的套接字文件描述符；
    - `my_addr`：要绑定的`socket地址`（需强转为通用结构`struct sockaddr*`）；
    - `addrlen`：`my_addr`对应的地址结构长度；
- 返回值：成功返回`0`，失败返回`-1`并设置`errno`。
三、常见错误码（`errno`）
1. **`EACCES`**：
    - 原因：绑定的地址是 “受保护地址”，普通用户无权限；
    - 典型场景：普通用户绑定`0~1023`的知名端口（如 80、443）。
2. **`EADDRINUSE`**：
    - 原因：绑定的地址正在被使用；
    - 典型场景：地址处于`TIME_WAIT`状态（服务器刚关闭，端口未释放）。
# 4.监听socket
一、`listen()`的核心作用
`socket`被`bind()`命名后，需调用`listen()`将其转为**被动监听状态**，并创建**监听队列**来存放待处理的客户端连接 —— 只有调用`listen()`后，服务器才能通过`accept()`接收客户端连接。
二、`listen()`函数的定义与头文件
```c
#include <sys/socket.h>
int listen( int sockfd, int backlog );
```
- 参数：
    - `sockfd`：待监听的套接字文件描述符；
    - `backlog`：监听队列的最大长度（用于限制待处理的连接数）；
- 返回值：成功返回`0`，失败返回`-1`并设置`errno`。
三、`backlog`参数的含义（Linux 内核版本差异）
`backlog`限制的是 “待处理连接的数量”，但不同内核版本定义不同：
1. **内核 2.2 之前**：`backlog`是 “半连接（`SYN_RCVD`）+ 完全连接（`ESTABLISHED`）” 的总数上限；
2. **内核 2.2 之后**：`backlog`仅限制 “完全连接（`ESTABLISHED`）” 的数量；半连接的上限由内核参数`/proc/sys/net/ipv4/tcp_max_syn_backlog`控制。
- 典型值：`5`；实际中完全连接的上限通常是`backlog+1`（如示例中`backlog=5`时，完全连接最多 6 个）。
四、超出`backlog`的后果
若监听队列中待处理的连接数超过`backlog`，服务器会**拒绝新连接**，客户端会收到`ECONNREFUSED`错误。
# 5.接受连接
一、`accept()`的核心逻辑
`accept()`是从**监听队列**中取出一个已完成三次握手的连接，返回一个**新的连接 socket**（用于和客户端通信），同时通过`addr`参数获取客户端的远端地址。
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260106095616.png)
二、核心特性
- **职责单一**：只从「完全连接队列」取连接，不参与三次握手，不验证连接有效性；
- **无感知性**：客户端即使异常断开（断网 / 退出），只要连接已进入完全连接队列，`accept()`就会成功返回；
- **状态不影响**：连接 socket 的状态（`ESTABLISHED`/`CLOSE_WAIT`）不影响`accept()`的返回结果；
- **真正的断连检测**：服务器必须通过`read()`/`write()`操作才能发现客户端断开：
    - `read(connfd, buf, len)`返回`0` → 客户端正常断开（发了 FIN 包）；
    - `read()`返回`-1`且`errno=ECONNRESET` → 客户端异常断开（比如强制杀死进程）；
    - `write()`返回`-1`且`errno=EPIPE` → 客户端已断开，写数据失败。
三、三次握手的发生
我们以`testaccept`的实验场景为例，拆解每一步的时机：

| 时间阶段         | 服务器行为（应用层）                                                        | 客户端行为（应用层）                 | 内核行为（三次握手核心）                                                                                                                                                                                                    |
| ------------ | ----------------------------------------------------------------- | -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 阶段 1：服务器准备   | 调用`socket()`创建监听 socket → `bind()`绑定地址 → `listen()`进入监听状态（LISTEN） | 无                          | 服务器内核初始化 “半连接队列”“完全连接队列”，等待客户端连接请求                                                                                                                                                                              |
| 阶段 2：客户端发起连接 | 服务器处于`sleep(20)`（应用层暂停，内核仍在工作）                                    | 调用`telnet`（本质是`connect()`） | 1. 客户端内核发**SYN 包**给服务器 → 客户端进入`SYN_SENT`状态；<br><br>2. 服务器内核收到 SYN 包 → 发**SYN+ACK 包**给客户端 → 服务器该连接进入`SYN_RCVD`状态（加入半连接队列）；<br><br>3. 客户端内核收到 SYN+ACK 包 → 发**ACK 包**给服务器 → 客户端进入`ESTABLISHED`状态；<br><br>✅ 三次握手完成！ |
| 阶段 3：连接入队    | 服务器仍在`sleep(20)`                                                  | 客户端可能断网 / 退出（应用层操作）        | 服务器内核将这个 “完成三次握手的连接” 从半连接队列移到**完全连接队列** → 连接状态变为`ESTABLISHED`                                                                                                                                                   |
| 阶段 4：服务器取连接  | 服务器`sleep`结束 → 调用`accept()`                                       | 客户端已断连 / 退出（应用层）           |                                                                                                                                                                                                                 |
 
细节：三次握手的触发点
- 客户端：调用`connect()` → 内核立刻发起 SYN 包，触发三次握手；
- 服务器：调用`listen()`后，内核才具备 “接收 SYN 包、完成握手” 的能力（没调用`listen()`的话，服务器内核会直接丢弃客户端的 SYN 包）。
2. 三次握手和`accept()`的关系
- 三次握手 **先完成**，连接才会进入 “完全连接队列”；
- `accept()`只是 “从队列里取连接”，**不参与、也不等待** 三次握手 —— 哪怕`accept()`还没调用，只要三次握手完成，连接就会在队列里等着。
# 6.发起连接
客户端通过`connect()`**主动向服务器发起 TCP 连接请求**，是客户端建立网络通信的关键调用（对应服务器的`listen()`/`accept()`）。
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260106101211.png)
- `sockfd`：客户端调用`socket()`创建的套接字文件描述符（未绑定地址时，由系统自动分配匿名地址）；
- `serv_addr`：服务器的`socket地址`（包含服务器的 IP 和端口，需强转为通用结构`struct sockaddr*`）；
- `addrlen`：`serv_addr`对应的地址结构长度；
- 客户端调用`connect()`后，内核会自动完成**TCP 三次握手**（和服务器内核交互 SYN/ACK 包）；
- 连接建立成功后，`sockfd`就唯一标识了 “客户端与服务器的这条连接”，客户端可通过`read()`/`write()`读写`sockfd`与服务器通信。
常见错误码
- **`ECONNREFUSED`**：
    - 原因：服务器的目标端口未被监听（比如服务器没启动，或端口号错误），连接被服务器内核拒绝；
- **`ETIMEDOUT`**：
    - 原因：连接超时（比如服务器无响应、网络故障，导致三次握手无法完成）。

**sockfd在connect和accept区别**

| 对比维度           | connect中的sockfd（客户端）             | accept中的sockfd（服务器）                   |
|----------------|----------------------------------|---------------------------------------|
| 所属端            | 客户端的套接字                          | 服务器的套接字                               |
| 职责             | 代表 “客户端与服务器的这条连接”，用于和服务器通信       | 代表 “服务器与某一个客户端的这条连接”，用于和该客户端通信        |
| 来源             | 客户端调用socket()直接创建（无需bind/listen） | 服务器调用accept()后新返回的套接字（不是监听 socket）    |
| 与监听 socket 的关系 | 无（客户端没有监听 socket）                | 由服务器的监听 socket（listen()后的 socket）触发生成 |
| 数量             | 客户端通常只有 1 个（对应一条连接）              | 服务器会有多个（每accept一个客户端连接，就生成一个新的sockfd） |

# 7.关闭连接
 一、`close`：通用的文件描述符关闭（适用于 socket，但有局限性）
```c
#include <unistd.h>
int close(int fd);
```
- **核心逻辑**：
    `close`是通用的 “关闭文件描述符” 调用，对 socket 来说，它的本质是**将 socket 的 “引用计数” 减 1**—— 只有当引用计数变为 0 时，才会真正关闭连接。
- **多进程场景的坑**：
    若父进程`fork`子进程，子进程会继承父进程的 socket，导致该 socket 的引用计数 + 1。此时必须**父、子进程都调用`close`**，才能让引用计数归 0，真正关闭连接。
- **局限性**：
    无法单独关闭 “读” 或 “写”，只能同时关闭 socket 的读写能力。
二、`shutdown`：专门为网络设计的连接关闭（更灵活）
```c
#include <sys/socket.h>
int shutdown(int sockfd, int howto);
```
- **核心逻辑**：
    专门针对 socket 设计，直接操作连接本身（不依赖引用计数），可以**单独关闭连接的 “读” 或 “写”**，`howto`参数决定行为：
    ![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260106102456.png)
- **优势**：
    不受引用计数影响，即使 socket 被多进程共享，调用`shutdown`也能直接关闭连接的对应方向；且支持 “半关闭”（比如只关闭写，仍能读对方数据）。
三、`close` vs `shutdown`的核心区别

|维度|`close`|`shutdown`|
|---|---|---|
|依赖引用计数|是（计数归 0 才关）|否（直接操作连接）|
|支持半关闭|否（只能同时关读写）|是（可单独关读 / 写）|
|适用场景|单进程、简单关闭|多进程、需要灵活控制读写的场景|

四、注意客户端和服务器的区别
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260106102954.png)
五、注意监听socket和连接socket区别
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260106103221.png)
举个直观例子：
- 服务器启动：创建`listenfd`（引用数 = 1），监听 54321 端口；
- 客户端 A 连接：服务器`accept`得到`connfd1`（引用数 = 1），和 A 通信；
- 客户端 B 连接：服务器`accept`得到`connfd2`（引用数 = 1），和 B 通信；
- 服务器调用`close(listenfd)`：`listenfd`引用数 = 0 → 停止监听 54321 端口（新客户端连不上），但 A、B 的`connfd1`/`connfd2`仍正常，和 A、B 的通信不受影响；
- 服务器再调用`close(connfd1)`：`connfd1`引用数 = 0 → 断开和 A 的连接，B 的`connfd2`仍正常。
总结
✅ 服务器调用`close(listenfd)`时，**监听 socket 的引用数会减 1**（规则和连接 socket 一致）；
✅ 监听 socket 和连接 socket 的引用数完全独立，关闭其中一个不会影响另一个的引用数；
✅ 关闭监听 socket 只会停止接收新连接，不会断开已建立的客户端连接。
# 8.数据读写
## 8.1 TCP数据读写
一、TCP 数据读写：`recv`/`send`
1. **通用接口兼容**：
    对 socket 的读写可以直接用`read`/`write`（因为 UNIX “一切皆文件”），但`socket`提供了更灵活的专用接口`recv`/`send`。
2. **`recv`/`send`的定义**：
    ```c
    #include <sys/types.h>
    #include <sys/socket.h>
    ssize_t recv(int sockfd, void *buf, size_t len, int flags);
    ssize_t send(int sockfd, const void *buf, size_t len, int flags);
    ```
    - `flags`参数是核心：提供额外的读写控制（如非阻塞、带外数据等），默认填`0`时效果等同于`read`/`write`。
3. **`recv`的返回值含义**：
    - 正数：实际读取的字节数（可能小于`len`，需循环读取）；
    - `0`：对方关闭连接；
    - `-1`：出错（需检查`errno`）。
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260106105256.png)

二、示例：带外数据的发送与接收
1. 客户端代码（`testoobsend`）逻辑：
- 创建 socket 并连接服务器；
- 发送**正常数据**`"123"`（`flags=0`）；
- 发送**带外数据**`"abc"`（`flags=MSG_OOB`）；
- 发送**正常数据**`"123"`（`flags=0`）。
2. 服务器代码（`testoobrecv`）逻辑：
- 创建监听 socket，`accept`客户端连接；
- 用`flags=0`读取正常数据；
- 用`flags=MSG_OOB`读取带外数据；
- 用`flags=0`读取剩余正常数据。
3. 实验结果与关键结论：
服务器输出：
```plaintext
got 5 bytes of normal data '123ab'
got 1 bytes of oob data 'c'
got 3 bytes of normal data '123'
```
- **带外数据的 “单字节特性”**：
    客户端发送的 3 字节带外数据`"abc"`，服务器仅收到最后 1 字节`"c"`—— 因为 TCP 的带外数据是**1 字节的紧急数据**（紧急偏移指向最后 1 字节），实际只能传递 1 个字节的紧急信息。
- **带外数据会截断正常数据**：
    正常数据`"123"`和带外数据的前 2 字节`"ab"`被合并为`"123ab"`，说明带外数据会插入到正常数据流中，导致正常数据被截断，无法通过一次`recv`读取完整。
## 8.2 UDP数据读写
一、UDP 数据读写的专用系统调用
UDP 是无连接的协议，因此读写数据时需要**明确指定通信对方的地址**，对应的系统调用是：
```c
#include <sys/socket.h>
// 读取UDP数据（同时获取发送端地址）
ssize_t recvfrom(int sockfd, void* buf, size_t len, int flags, 
                 struct sockaddr* src_addr, socklen_t* addrlen);
// 发送UDP数据（指定接收端地址）
ssize_t sendto(int sockfd, const void* buf, size_t len, int flags, 
               const struct sockaddr* dest_addr, socklen_t addrlen);
```
二、核心参数与逻辑（适配 UDP 的无连接特性）
1. **`recvfrom`（读）**：
    - 因为 UDP 无连接，每次读数据都需要通过`src_addr`获取**发送端的 socket 地址**（IP + 端口）；
    - `addrlen`需先初始化（传入`src_addr`的内存大小），内核会返回实际地址长度。
2. **`sendto`（写）**：
    - 因为 UDP 无连接，每次写数据都需要通过`dest_addr`指定**接收端的 socket 地址**（IP + 端口）；
    - `addrlen`是`dest_addr`的地址长度
三、与 TCP 读写接口的关联
- `recvfrom`/`sendto`的`flags`参数、返回值含义，和 TCP 的`recv`/`send`完全一致（比如`flags=0`是默认行为，返回值是实际读写字节数）；
- 这两个接口**也可以用于 TCP（面向连接的 socket）**：只需将`src_addr`/`dest_addr`设为`NULL`（因为 TCP 已建立连接，地址已知），此时效果等同于`recv`/`send`。
关键结论
1. UDP 的无连接特性，决定了其读写必须通过`recvfrom`/`sendto`明确处理通信地址；
2. `recvfrom`/`sendto`是通用接口，既支持 UDP（无连接），也支持 TCP（面向连接）；
3. 核心区别：UDP 用这两个接口处理 “动态地址”，TCP 用它们时可以忽略地址参数。