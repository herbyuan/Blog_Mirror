---
title: Socket编程基础
date: 2024-01-24 16:51:13
tags: C++
---

# Socket编程基础

简单来说, `socket` 可以看作网络中通信的方式, 虽然它也可以作为本地通信的方式 (比如`Unix Socket`). 跳过网上随处可见的介绍, 这里只强调一下 `socket` 位于应用层和传输层之间, 也就是说, `socket` 封装的是 `TCP/IP` 的数据包 (所以 `socket` 需要手动处理例如 `TCP` 三次握手之类的请求), 接口直接面向用户 (也就是进程直接把配置好的 `socket` 当做 IO 使用). 网上的教程和实例代码随便找有一大堆, 所以我来讲讲 `windows` 下的 `socket` 编程.

## echo 客户端与服务器

在这部分我们试图实现一个最简单的 `socket` 通信 server 与 client. server 总是返回 client 发送的相同内容.

### Server

最简单的例子就是这个, 服务端建立一个套接字在端口上监听, 等待客户端连接后 `echo` 客户端发来的所有内容. 开始之前我们仍然复习一下这张流程图.

![socket 通信流程](socket1.png)

先来说一下头文件, 在 `Linux` 中, 网络编程的头文件一般需要引入 `arpa/inet.h` 和 `sys/socket.h` (回头我测一下补上), 但是 `windows` 下不太一样. 而且 `windows` 的库需要显式初始化 所以最简单的预生成文件写法如下
```C++
#ifndef _WIN32
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#else
#include <ws2tcpip.h>
#pragma comment (lib, "Ws2_32.lib")
#endif
#include <iostream>

int main()
{
    // Initialize Winsock
    WSADATA wsaData;
    int iResult = WSAStartup(MAKEWORD(2,2), &wsaData);
    if (iResult != 0) {
        printf("WSAStartup failed with error: %d\n", iResult);
        return 1;
    }
    return 0;
}
```

对于服务器来说, 建立一个 `socket` 需要知道在什么地址的哪个端口上监听请求, 以及这个地址是 `ipv4` 还是 `ipv6`, 通讯的协议是 `TCP` 还是 `UDP` 还是其他什么, 有了这些信息我们才能构建好一个 `socket`. 结构体 `addrinfo` 完全包含了创建 `socket` 所需要的所有信息, 我们可以通过一个 `addrinfo` 创建 `socket`. `getaddrinfo` 函数提供从 ANSI 主机名到地址的与协议无关的转换, 通过提供有关调用方支持的套接字类型的提示, 由这个函数返回指向包含有关主机的响应信息的一个或多个 `addrinfo` 结构的链接列表的指针.

```C++
INT WSAAPI getaddrinfo(
  [in, optional] PCSTR           pNodeName,     // 主机 (节点) 名称或数字主机地址字符串
  [in, optional] PCSTR           pServiceName,  // 该字符串包含表示为字符串的服务名称或端口号
  [in, optional] const ADDRINFOA *pHints,       // 该结构提供有关调用方支持的套接字类型的提示
  [out]          PADDRINFOA      *ppResult      // 指向包含有关主机的响应信息的一个或多个 addrinfo 结构的链接列表的指针
);
```

可以看出, 我们需要向函数提供指向两个 `addrinfo` 的指针, 意味着使用完后记得使用 `freeaddrinfo()` 释放对应资源以免造成内存泄漏. 有了完整的 `addrinfo`, 我们就可以创建一个 `socket`

```C++
    // Create a SOCKET for the server to listen for client connections.
    ListenSocket = socket(result->ai_family, result->ai_socktype, result->ai_protocol);
    if (ListenSocket == INVALID_SOCKET) {
        printf("socket failed with error: %ld\n", WSAGetLastError());
        freeaddrinfo(result);
        WSACleanup();
        return 1;
    }
```

可以看到, 创建 `socket` 只是指定了通讯的协议, 而与地址或端口无关, 下一步才是绑定端口, 并开始监听.

```C++
    // Setup the TCP listening socket
    iResult = bind(ListenSocket, result->ai_addr, (int)result->ai_addrlen);
    if (iResult == SOCKET_ERROR) {
        printf("bind failed with error: %d\n", WSAGetLastError());
        freeaddrinfo(result);
        closesocket(ListenSocket);
        WSACleanup();
        return 1;
    }
    iResult = listen(ListenSocket, SOMAXCONN);
    if (iResult == SOCKET_ERROR) {
        printf("listen failed with error: %d\n", WSAGetLastError());
        closesocket(ListenSocket);
        WSACleanup();
        return 1;
    }
```

监听的端口可以接受来自 `socket` 的连接, 并返回新的文件描述符, 之后就可以在这个文件描述符上操作了. 注意此时的 `ClientSocket` 与 `ListenSocket` 并不相同. 建立起连接后, 原先的 `ListenSocket` 还可以重新接受新的连接. 而此时 `accept` 的特殊行为 (比如没有客户端连接时是直接返回错误还是阻塞等待) 是可以定义的, 由此衍生出一大堆例如多路复用的理论, 暂且不在这里讨论.

```C++
    // Accept a client socket
    ClientSocket = accept(ListenSocket, NULL, NULL);
    if (ClientSocket == INVALID_SOCKET) {
        printf("accept failed with error: %d\n", WSAGetLastError());
        closesocket(ListenSocket);
        WSACleanup();
        return 1;
    }
```

拿到的 `ClientSocket` 可以看作是一个文件描述符, 在这个文件描述符上可以写入也可以读取, 设置好缓冲区就行. 但是网络通讯受到通信协议的影响, 未必是想读多长就有多长, 因此返回值作为长度或者调用成功与否的标记非常重要.

完整的 Server 端代码如下:

```C++
// #include <windows.h>
// #include <winsock2.h>
#include <ws2tcpip.h>
#include <iostream>

#define DEFAULT_BUFLEN 512
#define DEFAULT_PORT "6000"

int main() 
{
    WSADATA wsaData;
    int iResult;

    SOCKET ListenSocket = INVALID_SOCKET;
    SOCKET ClientSocket = INVALID_SOCKET;

    struct addrinfo *result = NULL;
    struct addrinfo hints;

    int iSendResult;
    char recvbuf[DEFAULT_BUFLEN];
    int recvbuflen = DEFAULT_BUFLEN;
    
    // Initialize Winsock
    iResult = WSAStartup(MAKEWORD(2,2), &wsaData);
    if (iResult != 0) {
        printf("WSAStartup failed with error: %d\n", iResult);
        return 1;
    }

    ZeroMemory(&hints, sizeof(hints));
    hints.ai_family = AF_INET;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_protocol = IPPROTO_TCP;
    hints.ai_flags = AI_PASSIVE;

    // Resolve the server address and port
    iResult = getaddrinfo(NULL, DEFAULT_PORT, &hints, &result);
    if ( iResult != 0 ) {
        printf("getaddrinfo failed with error: %d\n", iResult);
        WSACleanup();
        return 1;
    }

    // Create a SOCKET for the server to listen for client connections.
    ListenSocket = socket(result->ai_family, result->ai_socktype, result->ai_protocol);
    if (ListenSocket == INVALID_SOCKET) {
        printf("socket failed with error: %d\n", WSAGetLastError());
        freeaddrinfo(result);
        WSACleanup();
        return 1;
    }

    // Setup the TCP listening socket
    iResult = bind(ListenSocket, result->ai_addr, (int)result->ai_addrlen);
    if (iResult == SOCKET_ERROR) {
        printf("bind failed with error: %d\n", WSAGetLastError());
        freeaddrinfo(result);
        closesocket(ListenSocket);
        WSACleanup();
        return 1;
    }

    freeaddrinfo(result);

    iResult = listen(ListenSocket, SOMAXCONN);
    if (iResult == SOCKET_ERROR) {
        printf("listen failed with error: %d\n", WSAGetLastError());
        closesocket(ListenSocket);
        WSACleanup();
        return 1;
    }

    // Accept a client socket
    ClientSocket = accept(ListenSocket, NULL, NULL);
    if (ClientSocket == INVALID_SOCKET) {
        printf("accept failed with error: %d\n", WSAGetLastError());
        closesocket(ListenSocket);
        WSACleanup();
        return 1;
    }

    // No longer need server socket
    closesocket(ListenSocket);

    // Receive until the peer shuts down the connection
    do {

        iResult = recv(ClientSocket, recvbuf, recvbuflen, 0);
        if (iResult > 0) {
            printf("Bytes received: %s\n", recvbuf);

        // Echo the buffer back to the sender
            iSendResult = send( ClientSocket, recvbuf, iResult, 0 );
            if (iSendResult == SOCKET_ERROR) {
                printf("send failed with error: %d\n", WSAGetLastError());
                closesocket(ClientSocket);
                WSACleanup();
                return 1;
            }
            printf("Bytes sent: %d\n", iSendResult);
        }
        else if (iResult == 0)
            printf("Connection closing...\n");
        else  {
            printf("recv failed with error: %d\n", WSAGetLastError());
            closesocket(ClientSocket);
            WSACleanup();
            return 1;
        }

    } while (iResult > 0);

    // shutdown the connection since we're done
    iResult = shutdown(ClientSocket, SD_SEND);
    if (iResult == SOCKET_ERROR) {
        printf("shutdown failed with error: %d\n", WSAGetLastError());
        closesocket(ClientSocket);
        WSACleanup();
        return 1;
    }

    // cleanup
    closesocket(ClientSocket);
    WSACleanup();

    return 0;
}
```

### Client

Client 的总体思路和 Server 基本一样, 不过作为发起连接的客户, 我们必须要设置对方的 `IP`, 并尝试 `getaddrinfo` 返回链表上的每一个 `addrinfo`. 总体来说, 流程就是 Server 设置好自己的 `socket` 并开始 `listen`, 等到 Client 发起 `connect` 后 `accept` 这个连接. 如果这个过程没有出问题, 两方的连接就正式建立好了, 在各自的文件描述符上读写即可.

完整的 Client 代码如下:

```C++
#include <ws2tcpip.h>
#include <stdlib.h>
#include <stdio.h>
#pragma comment (lib, "Ws2_32.lib")
#define DEFAULT_BUFLEN 512
#define DEFAULT_PORT "6000"

int main(int argc, char **argv) 
{
    WSADATA wsaData;
    SOCKET ConnectSocket = INVALID_SOCKET;
    struct addrinfo *result = NULL,
                    *ptr = NULL,
                    hints;
    const char *sendbuf = "this is a test";
    char recvbuf[DEFAULT_BUFLEN];
    int iResult;
    int recvbuflen = DEFAULT_BUFLEN;
    
    // Initialize Winsock
    iResult = WSAStartup(MAKEWORD(2,2), &wsaData);
    if (iResult != 0) {
        printf("WSAStartup failed with error: %d\n", iResult);
        return 1;
    }

    ZeroMemory( &hints, sizeof(hints) );
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_protocol = IPPROTO_TCP;

    // Resolve the server address and port
    iResult = getaddrinfo("127.0.0.1", DEFAULT_PORT, &hints, &result);
    if ( iResult != 0 ) {
        printf("getaddrinfo failed with error: %d\n", iResult);
        WSACleanup();
        return 1;
    }

    // Attempt to connect to an address until one succeeds
    for(ptr=result; ptr != NULL ;ptr=ptr->ai_next) {
        // Create a SOCKET for connecting to server
        ConnectSocket = socket(ptr->ai_family, ptr->ai_socktype, 
            ptr->ai_protocol);
        if (ConnectSocket == INVALID_SOCKET) {
            printf("socket failed with error: %ld\n", WSAGetLastError());
            WSACleanup();
            return 1;
        }

        // Connect to server.
        iResult = connect( ConnectSocket, ptr->ai_addr, (int)ptr->ai_addrlen);
        if (iResult == SOCKET_ERROR) {
            closesocket(ConnectSocket);
            ConnectSocket = INVALID_SOCKET;
            continue;
        }
        break;
    }

    freeaddrinfo(result);

    if (ConnectSocket == INVALID_SOCKET) {
        printf("Unable to connect to server!\n");
        WSACleanup();
        return 1;
    }

    // Send an initial buffer
    printf("OK\n");
    iResult = send( ConnectSocket, sendbuf, (int)strlen(sendbuf), 0 );
    if (iResult == SOCKET_ERROR) {
        printf("send failed with error: %d\n", WSAGetLastError());
        closesocket(ConnectSocket);
        WSACleanup();
        return 1;
    }

    printf("Bytes Sent: %ld\n", iResult);

    // Receive until the peer closes the connection
    do {
        iResult = recv(ConnectSocket, recvbuf, recvbuflen, 0);
        if ( iResult > 0 )
            printf("Bytes received: %s\n", recvbuf);
        else if ( iResult == 0 )
            printf("Connection closed\n");
        else
            printf("recv failed with error: %d\n", WSAGetLastError());
        char str[100] = {0};
        scanf("%s", str);
        iResult = send( ConnectSocket, str, (int)strlen(str) + 1, 0 );
    } while( iResult > 0 );

    // cleanup
    closesocket(ConnectSocket);
    WSACleanup();

    return 0;
}
```

> 这里多插一句, windows 下的 `Ws2_32.lib` 库需要显式调用, 每次 `WSAStartup()` 相当于对这个库的引用计数加一, `WSACleanup()` 则减少一次引用, 引用为零时释放资源.


## libevent 与 http server 示例

如果手工控制这些流程, 说实话有些繁琐, 而且稍有不慎就容易造成内存泄漏, 何况这只是 `TCP` 最简单的连接. 作为开发人员, 了解通信的每个步骤确实重要, 而实现业务目的才是根本要求. 这里要介绍的是 `libevent`, 它将底层的网络协议封装为事件驱动的函数调用, 开发者只要关注在不同事件触发时所需的行为, 大大简化了编码的流程.

### 事件驱动与 reactor

在我浅薄的理解里, reactor 框架和事件驱动是紧紧关联的, 也就是代码需要设立触发的条件(event), 并设置好条件下的行为 (reactor). 这里的事件往往是不可预测时机的, 比如 server 开了端口并在监听, 谁也不知道 client 什么时候会来连接. 最傻逼的方法就是盲等, 我可以阻塞在这里, client 不来这个进程就挂在这儿了, 无法继续执行, 也可以不来就直接返回不成功, 我循环调用直到成功为止. 这都是用户层能做的事. 而底层操作系统有自己的办法(信号), 通过一些奇技淫巧, 我们可以让一个**进程**持续监听多个连接, 也可以直接挂起(切换到别的**线程**), 直到一个连接到来的信号重新唤醒它. 这样的事件推动着代码往下执行, 而对事件的反应是至关重要的.

至于条件的等待以及等待时如何节约资源, 这部分很大依赖于操作系统的内核 (系统调用的部分). 而不同的系统有自己的系统调用, 比如 windows 就不支持大名鼎鼎的 `epoll`, 因此想自己写通用的高效底层代码也是不现实的, 这就是底层封装的好处——`libevent`会根据系统自动选择最高效的底层实现.

`libevent` 的精髓在于 `callback`, 中文叫回调函数, 在编程中我们只管定义特殊事件下调用的函数长什么样, 然后把其他活交个 `libevent` 就好 (`XXX_dispatch()`).

### http server

这是一个 `libevent` 官方的例子, 运行起来后我们指定端口与根文件夹 (http-server.exe -p 8888 D:\html), 即可实现简单的本地服务器 (访问 127.0.0.1:8888/index.html). 

示例的代码很长, 我不想在这里浪费空间, 所以我只是列出主要的函数名


```C++
/* Callback used for the /dump URI, and for every non-GET request:
 * dumps all information to stdout and gives back a trivial 200 ok */
static void dump_request_cb(struct evhttp_request *req, void *arg);

/* This callback gets invoked when we get any http request that doesn't match
 * any other callback.  Like any evhttp server callback, it has a simple job:
 * it must eventually call evhttp_send_error() or evhttp_send_reply().
 */
static void send_document_cb(struct evhttp_request *req, void *arg);

int main(int argc, char **argv)
{
    //...

    base = event_base_new_with_config(cfg);

    //...

    http = evhttp_new(base);

    /* The /dump URI will dump all requests to stdout and say 200 ok. */
    evhttp_set_cb(http, "/dump", dump_request_cb, NULL);

    /* We want to accept arbitrary requests, so we need to set a "generic"
     * cb.  We can also add callbacks for specific paths. */
    evhttp_set_gencb(http, send_document_cb, &o);

    //...

    handle = evhttp_bind_socket_with_handle(http, "0.0.0.0", o.port);

    //...

    term = evsignal_new(base, SIGINT, (void (*)(long long int, short int, void*))do_term, base);

    event_base_dispatch(base);
    //...
}

```

可以看出, 我们设置了访问具体 `URL` 时发送内容的回调函数, 而在主程序中建立一个 `http` 对象并与回调函数关联, 最后我们直接将对象绑定到了 `ip:port` 上并开始了任务. 细碎的 `socket` 操作都由系统完成了. 

### 一个例子

这里我们着重来看一下 `evhttp_bind_socket_with_handle()` 函数, 并探讨其他的等效实现方式. 函数接收一个 `event` 对象, 并与指定的 IP 和端口绑定, 并返回文件描述符.

我们可以拆分成两部分, 先建立一个监听指定IP和端口的 `socket`, 然后把这个 `socket` 接收来自 `event` 对象的信息.

```C++
        std::string address = "0.0.0.0:9000";
        struct sockaddr_storage sa;
        struct sockaddr* addr = reinterpret_cast<struct sockaddr*>(&sa);
        int len = sizeof(sa);
        evutil_parse_sockaddr_port(address.c_str(), addr, &len);

        base = event_base_new();
        auto lev = evconnlistener_new_bind(base, NULL, NULL,
            LEV_OPT_CLOSE_ON_FREE | LEV_OPT_BIND_IPV4_AND_IPV6, -1,
            (struct sockaddr *)addr, sizeof(sockaddr_storage));

        if (!lev) {
            perror("Cannot create listener");
            ret = 1;
            goto err;
        }

        handle = evhttp_bind_listener(http, lev);
```

可以看到, 我们能把封装的函数拆成更细的部分, 并有更多自定义的空间 (比如这里我们设置了双栈支持), `libevent` 在各个层次上都实现了封装.

### 纯粹的底层实现

```C++
        struct addrinfo *result = NULL;
        struct addrinfo hints;
        ZeroMemory(&hints, sizeof(hints));
        hints.ai_family = AF_INET;
        hints.ai_socktype = SOCK_STREAM;
        hints.ai_protocol = IPPROTO_TCP;
        hints.ai_flags = AI_PASSIVE;

        // Resolve the server address and port
        int iResult = getaddrinfo(NULL, "9000", &hints, &result);
        if ( iResult != 0 ) {
            printf("getaddrinfo failed with error: %d\n", iResult);
            WSACleanup();
            return 1;
        }

        // Create a SOCKET for the server to listen for client connections.
        int fd = socket(result->ai_family, result->ai_socktype, result->ai_protocol);
        evutil_make_socket_nonblocking(fd);
        evutil_make_listen_socket_reuseable(fd);
        // int off = 0;
        // setsockopt(fd, IPPROTO_IPV6, IPV6_V6ONLY, (char*)&off, (ev_socklen_t)sizeof(off));
        int on =1;
        setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, (char*)&on, sizeof(on));
        if (fd == INVALID_SOCKET) {
            printf("socket failed with error: %ld\n", WSAGetLastError());
            freeaddrinfo(result);
            WSACleanup();
            return 1;
        }
        int res=bind(fd, result->ai_addr, (int)result->ai_addrlen);
        if (res != 0){
            printf("bind error!\n");
            return 1;
        }
        if (listen(fd, 0x7fffffff) == -1) {
            printf("listen error!\n");
            return 1;
        }
        handle = evhttp_accept_socket_with_handle(http, fd);
```



# 参考链接

https://learn.microsoft.com/zh-cn/windows/win32/winsock/complete-server-code

https://opensource.apple.com/source/ntp/ntp-124/sntp/libevent/include/event2/http.h.auto.html

