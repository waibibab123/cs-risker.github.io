# 前言
内容来源：本文章为阅读《Linux高性能服务器编程》第九章.I/O复用所记录的**部分**笔记。

**I/O 复用技术**是提升程序性能、解决高并发问题的关键手段，它允许单个程序进程同时监听多个文件描述符，从而高效地处理客户端的多连接请求、用户输入与网络交互，以及服务器端的跨协议（TCP/UDP）和多端口服务。尽管 I/O 复用能感知多个事件的就绪状态，但其系统调用本身具有阻塞性，且在处理多个就绪事件时默认是串行执行的；因此，在实际开发中，常需结合多线程或多进程来实现真正的并行并发。在 Linux 环境下，这一技术主要通过 `select`、`poll` 以及基于内核事件表的更高性能的 `epoll` 系统调用来实现。

# 1.select系统调用
## 1.1 select API
select的核心作用：在指定时间内，**同时监听多个文件描述符（如 socket中的listenfd和connfd）的 “可读、可写、异常” 事件** ，实现单进程 / 线程处理多个 I/O 操作。
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260107110321.png)
**参数说明**：
1. `nfds`：被监听的文件描述符的**最大值 + 1**（因为文件描述符从 0 开始计数）；
2. `readfds/writefds/exceptfds`：分别指向 “关注可读、可写、异常事件” 的文件描述符集合（`fd_set`类型）；
	`fd_set`是`select`用于管理文件描述符的结构：
	![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260107110538.png)
	`fd_set`结构体仅包含一个整型数组，该数组每个元素的每一位标记一个文件描述符，其所能容纳的文件描述符数量由FD_SETSIZE指定，这就限制了select能同时处理的文件描述符的总量，例如：假设`__fd_mask`是 8 字节→64 位，`FD_SETSIZE=1024`，我们想要存储fd=3和fd=70两个文件描述符，`fd_set`的内部状态为：
	![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260107112241.png)
	![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260107111013.png)
	**举一个例子**：
```c
#include <stdio.h>
#include <sys/select.h>

int main() {
    // 1. 定义一个fd_set集合
    fd_set my_fd_set;

    // 2. 清空集合（必须先清空，否则初始值是随机的）
    FD_ZERO(&my_fd_set);

    // 3. 装载（添加）fd到集合中
    int fd1 = 3;  // 假设是listenfd
    int fd2 = 5;  // 假设是connfd
    FD_SET(fd1, &my_fd_set);
    FD_SET(fd2, &my_fd_set);

    // 4. 判断某个fd是否在集合中
    if (FD_ISSET(fd1, &my_fd_set)) {
        printf("fd=%d 已被装载到集合中\n", fd1);
    }
    if (FD_ISSET(fd2, &my_fd_set)) {
        printf("fd=%d 已被装载到集合中\n", fd2);
    }
    if (!FD_ISSET(4, &my_fd_set)) {
        printf("fd=4 不在集合中\n");
    }

    // 5. 从集合中移除fd
    FD_CLR(fd2, &my_fd_set);
    if (!FD_ISSET(fd2, &my_fd_set)) {
        printf("fd=%d 已从集合中移除\n", fd2);
    }

    return 0;
}
```
3. `timeout`：设置select函数的超时时间
	![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260107112937.png)
    - timeout的成员变量均设`0`：立即返回；
    - timeout设为`NULL`：永久阻塞，直到有某个文件描述符就绪。
    select成功时返回就绪文件描述符的总数，如果在超过时间内没有任何文件描述符就绪，select返回0，select失败会返回-1
## 1.2 文件描述符就绪条件
 一、socket “可读” 的条件（`readfds`就绪）
当满足以下任意一种情况时，`select`会标记该 socket 为 “可读”：
1. **接收缓存区有数据**：
    socket 内核接收缓存区的字节数 ≥ 低水位标记`SO_RCVLOWAT`（默认通常是 1 字节），此时调用`read/recv`可以无阻塞地读到数据（返回字节数 > 0）。
2. **对方关闭连接**：
    通信对方关闭连接（发了 FIN 包），此时调用`read/recv`会返回 0（表示连接已关闭）。
3. **监听 socket 有新连接**：
    `listenfd`对应的监听 socket 上有新客户端完成三次握手，此时调用`accept`可以取出新连接。
4. **socket 有未处理错误**：
    socket 发生错误（如连接失败），此时可通过`getsockopt`读取并清除错误。

二、socket “可写” 的条件（`writefds`就绪）
当满足以下任意一种情况时，`select`会标记该 socket 为 “可写”：
1. **发送缓存区有空间**：
    socket 内核发送缓存区的可用字节数 ≥ 低水位标记`SO_SNDLOWAT`（默认通常是 1 字节），此时调用`write/send`可以无阻塞地发送数据（返回字节数 > 0）。
2. **写操作被关闭**：
    socket 的写端被关闭（如调用`shutdown(sockfd, SHUT_WR)`），此时执行写操作会触发`SIGPIPE`信号。
3. **非阻塞 connect 完成**：
    用非阻塞`connect`发起的连接，无论成功或失败（超时），都会标记 socket 为 “可写”。
4. **socket 有未处理错误**：
    同 “可读” 的第 4 条，socket 发生错误时也会标记为 “可写”。

三、socket “异常” 的条件（`exceptfds`就绪）
`select`中 socket 的异常事件**只有一种场景**：
- socket 接收到**带外数据（OOB）**（即对方用`send(..., MSG_OOB)`发送的紧急数据）。
## 1.3 处理带外数据
一、核心逻辑：区分普通数据与带外数据的就绪状态
`select`中，socket 接收**普通数据**会触发 “可读事件（`readfds`）”，接收 **带外数据（OOB）** 会触发 “异常事件（`exceptfds`）”—— 通过同时监听这两个事件集合，就能分别处理两类数据。
二、代码实现步骤
1. **初始化监听集合**：
    定义`read_fds`（监听普通数据）和`except_fds`（监听带外数据），并通过`FD_ZERO`清空集合。
2. **循环监听事件**：
    - 每次`select`前，都要重新用`FD_SET`将`connfd`加入`read_fds`和`except_fds`（因为`select`会修改集合，需重新设置）；
    - 调用`select`阻塞等待事件（仅监听`read_fds`和`except_fds`）。
3. **处理事件**：
    - 若`connfd`在`read_fds`中：用普通`recv(..., 0)`读取**普通数据**；
    - 若`connfd`在`except_fds`中：用`recv(..., MSG_OOB)`读取**带外数据**。
三、关键细节
- **每次`select`前需重新设置集合**：因为`select`返回后会修改`fd_set`（只保留就绪的 fd），所以下次调用前必须重新将`connfd`加入集合；
- **带外数据的读取标志**：必须用`MSG_OOB`标志调用`recv`，才能正确读取带外数据；
- **异常事件的唯一性**：`select`中 socket 的异常事件仅对应 “带外数据到达”，因此`except_fds`就绪时直接处理带外数据即可。
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260107115223.png)
# 2.poll系统调用
`poll` 与 `select` 类似，用于在指定时间内轮询一定数量的文件描述符（fd），以测试其中是否有就绪事件（如可读、可写、异常等）。
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260115210255.png)
- **`fds`** ：指向一个 `pollfd` 结构体数组的指针。
- **`nfds`** ：数组中元素的个数。
- **`timeout`** ：超时时间，单位是**毫秒**。

核心数据结构：`struct pollfd`
这是 `poll` 与 `select` 最显著的区别。它通过结构体而非位图（bitmask）来管理事件，更加清晰直观。
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260115210448.png)
- **`events`（输入）** ：用户设置的感兴趣的事件。
- **`revents`（输出）** ：内核修改，返回实际发生的事件。由于输入和输出分离，**不需要像 `select` 那样每次调用前重置**。

常用事件类型（`poll` 事件表）
- **`POLLIN`** ：数据可读（包括普通数据和优先级数据）。
- **`POLLOUT`** ：数据可写。
- **`POLLRDHUP`** （重点）：自 Linux 2.6.17 起引入，用于**检测 TCP 连接被对方关闭**或对方关闭了写操作。使用时需定义 `_GNU_SOURCE`。
- **`POLLERR`** ：发生错误。
- **`POLLHUP`** ：挂起（如管道写端关闭，读端将收到此事件）。
- **`POLLNVAL`** ：文件描述符没有打开。

代码案例：
```cpp
struct pollfd fds[2];
// 监听socket的可读事件（比如是否有客户端连接、是否有数据发送过来）
fds[0].fd = sockfd;
fds[0].events = POLLIN;

// 监听文件描述符的可写+错误事件
fds[1].fd = filefd;
fds[1].events = POLLOUT | POLLERR;

int ret = poll(fds, 2, 5000); // 监听2个fd，超时5秒
if(ret > 0){
    // 检查第一个fd是否触发了可读事件
    if(fds[0].revents & POLLIN){
        // 执行socket读操作，比如accept或者recv
        handle_socket_read(fds[0].fd);
    }
    // 检查第二个fd是否触发了可写事件
    if(fds[1].revents & POLLOUT){
        // 执行文件写操作
        handle_file_write(fds[1].fd);
    }
    // 检查是否出现错误
    if(fds[1].revents & POLLERR){
        // 处理错误逻辑
        handle_error(fds[1].fd);
    }
}
```
# 3.epoll系统调用
## 内核事件表
一、 epoll 概述 (epoll Overview)
- **特性**：Linux 特有的 I/O 复用函数。
- **核心差异**：与 `select` 和 `poll` 不同，`epoll` 使用**一组函数**来完成任务，而不是单个函数。
- **内核事件表**：`epoll` 在内核中维护一个事件表，记录用户关心的文件描述符（fd）及其事件。
    - **优势**：无需像 `select/poll` 那样每次调用都重复传入文件描述符集或事件集，大幅减少了数据拷贝的开销。
- **句柄**：`epoll` 需要一个额外的文件描述符来唯一标识内核中的这个事件表。

二、 核心 API：epoll_create
用于创建一个内核事件表。

```
#include <sys/epoll.h>
int epoll_create(int size);
```
- **参数 `size`** ：现在不起实际作用，仅给内核一个提示，建议事件表的大小。
- **返回值**：成功时返回一个文件描述符，作为后续所有 epoll 系统调用的第一个参数（用于指定访问的内核事件表）。

三、 核心 API：epoll_ctl
用于操作（添加、修改、删除）内核事件表中的事件。
```
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
1. 操作类型 (`op`)
- **`EPOLL_CTL_ADD`** ：往事件表中注册 `fd` 上的事件。
- **`EPOLL_CTL_MOD`** ：修改 `fd` 上已注册的事件。
- **`EPOLL_CTL_DEL`** ：从事件表中删除 `fd` 上的注册事件。
1. 重要结构体：`struct epoll_event`
```
struct epoll_event {
    __uint32_t events;    /* epoll 事件类型 */
    epoll_data_t data;    /* 用户数据 */
};
```
- **`events` 成员**：
    - 描述事件类型（如 `EPOLLIN` 表示可读）。
    - **关键类型**：`EPOLLET`（边缘触发）和 `EPOLLONESHOT`（保证同一fd只被一个线程处理）。
- **`data` 成员**：类型为 `epoll_data_t`（联合体），用于存储用户数据。
3. 用户数据联合体：`epoll_data_t`
```
typedef union epoll_data {
    void* ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
```
- **使用注意**：
    - 它是**联合体**，不能同时使用 `ptr` 和 `fd`。
    - **常用方式**：通常使用 `fd` 指定事件所属的目标文件描述符；若需要关联更多用户数据，则使用 `ptr`（并在数据结构中包含 `fd`）。
四、 返回值总结
- **`epoll_ctl`** ：成功返回 `0`，失败返回 `-1` 并设置 `errno`。
## epoll_wait函数
 一、 epoll_wait 函数原型
`epoll_wait` 是 epoll 系列系统调用的核心接口，用于在一段超时时间内等待一组文件描述符上的事件。
```
#include <sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
```
1. 参数详解 (从后往前)
- **`timeout`** ：指定等待的超时时间（毫秒）。含义与 `poll` 的 timeout 参数相同。
- **`maxevents`** ：指定最多监听多少个事件，必须大于 0。
- **`events` (核心)** ：
    - 这是一个**输出型参数**。
    - 如果检测到就绪事件，内核会将所有就绪事件从内核事件表复制到这个数组中。
- **`epfd`** ：由 `epoll_create` 创建的 epoll 文件描述符，指定要访问的内核事件表。
1. 返回值
- **成功**：返回就绪的文件描述符个数。
- **失败**：返回 `-1` 并设置 `errno`。
二、 epoll_wait 的设计优势
与 `select` 和 `poll` 相比，`epoll_wait` 在处理就绪事件时效率极高：
- **不仅是输入，更是输出**：`select/poll` 传入的数组既包含要监听的事件，又在返回时被覆盖为就绪事件。而 `epoll` 的 `events` 数组**只用于输出**检测到的就绪事件。
- **内核到用户的拷贝**：无需像 `select/poll` 那样每次调用都重新传入整个事件集合，只需从内核取出已就绪的部分。
![image.png](https://raw.githubusercontent.com/waibibab-cs/blog_img/main/cdnimg/20260119173739.png)
## LT和ET模式
一、 基本概念
`epoll` 对文件描述符的操作有两种工作模式，决定了内核何时以及如何通知应用程序事件。
1. LT 模式 (Level Trigger, 电平触发)
- **地位**：默认工作模式。
- **机制**：类似于一个高效率的 `poll`。只要文件描述符上有事件（如读缓存中有数据），每次调用 `epoll_wait` 都会触发通知。
- **特点**：即使应用程序在得到通知后不立即处理，下次调用 `epoll_wait` 时仍会再次通知，直到该事件被处理完。
2. ET 模式 (Edge Trigger, 边沿触发)
- **地位**：`epoll` 的高效工作模式。
- **启用**：在注册事件时通过位或 `EPOLLET` 宏开启。
- **机制**：内核只在状态发生变化时通知一次（例如：数据从无到有）。
- **特点**：应用程序收到通知后**必须立即处理**该事件。因为后续的 `epoll_wait` 调用将不再向应用程序发送关于该事件的重复通知。
二、 核心差异对比

|特性|LT 模式 (Level Trigger)|ET 模式 (Edge Trigger)|
|---|---|---|
|**通知频率**|只要满足条件就会重复触发|仅在状态改变时触发一次|
|**编程难度**|较低，允许只处理部分数据|较高，必须一次性完成处理|
|**性能效率**|性能不错，但系统调用次数可能较多|极高，大幅降低了重复触发次数|
|**I/O 模式**|阻塞/非阻塞均可|**必须使用非阻塞 I/O**|


三、 编程实现要点 (基于代码清单 9-3)
1. 非阻塞设置 (`setnonblocking`)
在 ET 模式下，必须通过 `fcntl` 将文件描述符设置为非阻塞模式，否则读或写操作将会因为没有后续的事件而一直处于阻塞状态
- **对于阻塞 I/O**：如果你用 `epoll` 监听 10000 个 `sockfd`，但你使用了阻塞 I/O。当你去 `read` 其中一个连接时，万一由于某种原因数据没读全，导致程序被阻塞在该 fd 上，另外 9999 个连接就算有新消息，你也处理不了了。
- **对于非阻塞 I/O**：通过 `fcntl` 设置了 `O_NONBLOCK` 后，`epoll_wait` 告诉你哪个 fd 有数据，你才去读。即使你读得太勤（读到了缓冲区为空），也会因为非阻塞特性立即返回，让你能赶紧回到 `epoll_wait` 去看护其他 9999 个“孩子”
```
int old_option = fcntl(fd, F_GETFL);
int new_option = old_option | O_NONBLOCK;
fcntl(fd, F_SETFL, new_option);
```
2. ET 模式下的读取逻辑
由于 ET 模式只通知一次，如果读缓存中数据很多，一次 `recv` 读不完，`epoll_wait` 就不会再报了。
- **解决方案**：必须配合 **`while(1)` 循环** 读取，直到数据全部读出。
- **退出循环条件**：当 `recv` 返回 `-1` 且 `errno` 为 `EAGAIN` 或 `EWOULDBLOCK` 时，说明缓冲区已空。
2. LT 模式下的读取逻辑
- 比较简单，每次触发 `EPOLLIN` 后调用一次 `recv` 即可。如果还有剩余数据，内核会在下一次循环中再次通知。
## EPOLLONESHOT事件
一、 I/O 模式：阻塞 vs 非阻塞
- **阻塞 I/O (Blocking)** ：调用 `recv` 时若无数据，线程挂起死等。
    - 缺点：单线程只能处理一个连接，并发能力差。
- **非阻塞 I/O (Non-blocking)** ：调用 `recv` 时若无数据，立即返回 `-1` 并设置 `errno = EAGAIN`。
    - 优点：配合多路复用（epoll）可实现单线程管理万级连接。
 二、 触发模式：LT vs ET
- **LT (水平触发 - Level Triggered)** ：
    - **逻辑**：只要缓冲区有数据，`epoll_wait` 就会不断通知。
    - **特点**：编程简单，安全可靠；但内核通知次数多，开销大。
- **ET (边沿触发 - Edge Triggered)** ：
    - **逻辑**：只有状态发生变化（数据从无到有、新数据到达）时才通知一次。
    - **特点**：效率极高，减少内核通知次数。
    - **硬性要求**：必须配合**非阻塞 I/O**，且必须使用 `while` 循环读完所有数据（读到 `EAGAIN` 为止）。
三、 核心难点：为什么 ET 模式下仍会触发多次？
- **现象**：在 ET 模式下，如果线程正在处理旧数据时突然有**新数据**到达，内核会将其判定为一个新的“边沿”，导致 `epoll_wait` 再次被触发。
- **风险**：在多线程开发中，这会导致**不同线程同时操作同一个 socket**，引发竞态条件（Race Condition），导致数据读写错乱。
四、 解决方案：EPOLLONESHOT 事件
- **定义**：一种特殊的 epoll 事件标志。
- **作用机制**：
    1. **触发一次**：内核触发该 fd 的事件后，立即将其从内核事件表中“禁用”（不再监控）。
    2. **独占性**：无论该 socket 是否有新数据，`epoll_wait` 均不再通知，确保同一时刻**只有一个线程**在处理该 socket。
    3. **人工重置**：当工作线程处理完该 socket 的逻辑后，必须手动调用 `epoll_ctl` 使用 `EPOLL_CTL_MOD` 命令重置（重新激活）该事件。
- **注意**：`listenfd`（监听 socket）通常**不设置** `EPOLLONESHOT`，否则只能接收到一个客户端连接。
五、 编程最佳实践（代码逻辑模型）
1. **设置非阻塞**：`fcntl(fd, F_SETFL, O_NONBLOCK)`。
2. **注册事件**：`event.events = EPOLLIN | EPOLLET | EPOLLONESHOT`。
3. **循环读取**：工作线程接收到通知后，用 `while` 循环 `recv`。
4. **识别结束**：
    - `ret > 0`：继续读。
    - `ret == 0`：对端关闭连接，执行 `close(fd)`。
    - `ret < 0 && errno == EAGAIN`：读完了，调用 `reset_oneshot()` 恢复监控。
# 4.三组I/O复用函数的比较
一、 I/O 复用技术大比拼 (select, poll, epoll)

| 特性        | select           | poll             | epoll                 |
| --------- | ---------------- | ---------------- | --------------------- |
| **数据结构**  | 3个 fd_set（位图）    | pollfd 结构体数组     | 内核事件表                 |
| **索引复杂度** | O(n)：需遍历整个 fd 集合 | O(n)：需遍历整个 fd 集合 | **O(1)** ：直接返回就绪事件    |
| **最大连接数** | 有限制（通常 1024）     | 无限制（65535+）      | 无限制（65535+）           |
| **内核实现**  | **轮询**方式扫描所有 fd  | **轮询**方式扫描所有 fd  | **回调**方式（Callback）    |
| **工作模式**  | 仅支持 LT (水平触发)    | 仅支持 LT (水平触发)    | **支持 ET (边沿触发) 和 LT** |
| **参数传递**  | 每次调用需重置 fd 集合    | 无需重置事件参数         | 无需重复传入事件，仅需添加/修改      |

二、 epoll 的进阶神器：`EPOLLONESHOT`
1. 核心背景：多线程并发冲突
在多线程环境下，即便使用 ET（边沿触发）模式，如果一个线程在处理某个 Socket 期间又有新数据到达，内核可能会唤醒另一个线程来处理同一个 Socket。
- **后果**：导致数据交织、状态混乱、竞态条件。
2. `EPOLLONESHOT` 的原理
一旦某个文件描述符（fd）上注册了该事件：
- **一次性触发**：内核触发读、写或异常事件中的任意一个后，会立即**“禁用”** 该 fd 的所有后续通知。
- **状态冻结**：除非程序员手动重置，否则该 fd 永远不会再通过 `epoll_wait` 返回。
1. 核心机制：重置（Reset）
- **目的**：通知内核该 fd 已经处理完毕，可以重新开始监控。
- **操作方法**：调用 `epoll_ctl` 并指定 `EPOLL_CTL_MOD` 命令。
- **本质逻辑**：
    - 即便修改后的事件参数与之前完全一样，调用 `MOD` 也会**清除内核内部的禁用标记**。
    - 这相当于给“保险丝”推闸复位。
三、 典型并发处理流（工作线程模式）
1. **注册**：主线程将 Socket 注册为 `EPOLLIN | EPOLLET | EPOLLONESHOT`。
2. **分发**：`epoll_wait` 收到通知，唤醒一个工作线程处理该 Socket。
3. **处理**：
    - 由于 `EPOLLONESHOT`，其他线程绝不会抢占该 Socket。
    - 工作线程循环读取数据直到返回 `EAGAIN`。
4. **复位**：处理完成后，工作线程调用 `reset_oneshot`（即 `epoll_ctl(MOD)`）。
5. **循环**：内核重新监控，等待下一次事件。
四、 关键结论与建议
- **性能权衡**：`epoll` 在连接数多、但活跃连接少的情况下表现最佳；如果连接极少且极其活跃，`select/poll` 性能未必更差。
- **安全防范**：凡是**多线程**操作同一个 epoll 实例中的相同 fd，**必须**加 `EPOLLONESHOT`，否则无法保证线程安全。
- **细节提醒**：注册了 `EPOLLONESHOT` 的 fd，一旦处理完业务逻辑，千万记得 `MOD` 回去，否则该 Socket 会变成“死连接”（永不触发）。
