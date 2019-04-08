# Berkeley sockets

## 什么是socket？

- 套接字是网络通信路径的本地端点的抽象表示(句柄)。Berkeley sockets API将其表示为Unix中的一个文件描述符(文件句柄)，它为数据流的输入和输出提供了一个公共接口。
- 简单来说：它们是利用标准 UNIX file descriptors（文件描述符）与其它程序沟通的一种方式。

## BSD和POSIX套接字有差别的API表

| 作用	        		    | BSD                                                         | POSIX       |
| ------------------------- | :---------------------------------------------------------: | ----------- |
| 从文本地址到压缩地址的转换 	| inet_aton	                                                  | inet_pton   |     
| 从压缩地址到文本地址的转换	| inet_ntoa                                                   | inet_ntop   |
| 向前查找主机名/服务器       | gethostbyname, gethostbyaddr, getservbyname, getservbyport  | getaddrinfo |
| 主机名/服务器的反向查找     |  gethostbyaddr, getservbyport	                              | getnameinfo |

## 所含头文件
Berkeley套接字接口在几个头文件中定义。这些文件的名称和内容在实现之间略有不同。一般来说，包括:
|    头文件	   |       头文件说明                                                                                    |
| ------------ | -------------------------------------------------------------------------------------------------- |
| sys/socket.h | 核心套接字函数和数据结构。                                                                            |
| netinet/in.h | AF_INET和AF_INET6分别处理ipv4和ipv6相应的协议族，PF_INET和PF_INET6。这些包括标准IP地址和TCP和UDP端口号。	| 
| sys/un.h     | PF_UNIX和PF_LOCAL地址族。用于同一计算机上运行的程序之间的本地通信。                                      |
| arpa/inet.h  | 用于操作数字IP地址的函数。                                                                            |
| netdb.h      | 用于将协议名和主机名转换为数字地址以及搜索本地数据和名称服务。											|

## 套接字函数总结
![](InternetSocketBasicDiagram.png)
套接字API通常提供以下功能:
- socket()函数创建一个特定类型的套接字(由整数编号标识)，并将系统资源分配给它。
- bind()通常用于服务器端，并将套接字与套接字地址结构相关联，即指定的本地IP地址和端口号。
- listen()在服务器端使用，并让绑定的TCP套接字进入监听状态。
- connect()在客户端使用，并将空闲的本地端口号分配给套接字。对于TCP套接字，它会尝试建立新的TCP连接。
- accept()在服务器端使用。它接收从远程客户机创建的新TCP连接的接收传入请求，并创建与此连接的套接字地址相关联的新套接字。
- send(),recv(),sendto()以及recvfrom()用于发送和接收数据。还可以使用标准函数write()和read()。
- close() 使系统释放分配给套接字的资源。对于TCP而言作用是终止连接。
- gethostbyname()和gethostbyaddr() 于解析主机名和地址。仅支持IPv4。
- select()用于挂起程序，等待套接字列表中的一个或多个套接字去准备读/写或出现错误而继续挂起。
- poll() 用于检查套接字中的套接字的状态。这个函数可以测试套接字集合，看看是否有套接字可以写入/读取或是否发生错误。
- getsockopt() 用于检索指定套接字的特定套接字选项的当前值。
- setsockopt() 用于为指定的套接字设置特定套接字选项。

### socket
```
#include <sys/socket.h>
int socket(int family, int type, int protocol);
```
函数socket()创建用于通信的端点，并返回套接字的文件描述符。它使用了三个参数:

#### family指定所创建套接字的协议族。例如:
- AF_INET用于网络协议IPv4(仅IPv4)
- AF_INET6对于IPv6(在某些情况下，向后兼容IPv4)
- AF_UNIX用于本地套接字(使用文件)

#### type
- SOCK_STREAM(可靠的面向流的服务或流套接字)
- SOCK_DGRAM(数据报服务或数据报套接字,多用于UDP)
- SOCK_SEQPACKET(可靠的顺序分组服务)
- SOCK_RAW(网络层上的原始协议)

#### protocol指定要使用的实际传输协议。最常见的是IPPROTO_TCP、IPPROTO_SCTP、IPPROTO_UDP、IPPROTO_DCCP。这些协议在netinet/in.h文件中指定。值0可用于从选定的域和类型中选择默认协议。

如果发生错误，函数返回-1。否则，它返回一个整数，表示新分配的描述符。

### bind
```
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *my_addr, socklen_t addrlen);
```
bind()将套接字与地址关联起来。当使用套接字()创建套接字时，只给它一个协议族，而不给它分配地址。在套接字可以接受来自其他主机的连接之前，必须执行此关联。该函数有三个参数:
- sockfd，表示套接字的描述符
- my_addr，指向表示要绑定到的地址的sockaddr结构的指针。
- addrlen，类型为socklen_t的字段，指定sockaddr结构的大小。
Bind()成功时返回0，出错时返回-1。

### listen
```
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```
在套接字与地址关联之后，listen()为传入连接准备套接字。然而，这只对面向流的(面向连接的)数据模式是必要的，即，用于套接字类型(SOCK_STREAM、SOCK_SEQPACKET)。listen()需要两个参数:
- sockfd，一个有效的套接字描述符。
- backlog，一个整数，表示可以在任何时间排队的挂起连接的数量。操作系统通常对这个值设置一个上限。
一旦连接被接受，它就会退出队列。成功时，返回0。如果发生错误，返回-1。

### accept
```
#include <sys/socket.h>
int accept(int sockfd,struct sockaddr *cliaddr, socklen_t *addrlen);
```
当应用程序侦听来自其他主机的面向流的连接时，它会收到此类事件的通知(cf. select()函数)，
并且必须使用accept()函数初始化连接。它为每个连接创建一个新的套接字，并从监听队列中删除连接。该函数有以下参数:
- sockfd，具有排队连接的监听套接字的描述符。
- cliaddr,一个指向sockaddr结构的指针，用于接收客户端的地址信息。
- addrlen，指向socklen_t位置的指针，该位置指定传递给accept()的客户机地址结构的大小。
当accept()返回时，这个位置包含结构的大小(以字节为单位)。

accept()返回接收连接的新套接字描述符，如果发生错误，返回-1。所有与远程主机的进一步通信现在都通过这个新的套接字进行。数据报套接字不需要accept()处理，因为接收者可以使用监听套接字立即响应请求。

### connect
connect()建立到特定远程主机的直接通信连接，该远程主机通过套接字(套接字由文件描述符标识)的地址标识。
```
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);
```
connect参数与bind类似
- connect()建立到特定远程主机的直接通信连接，该远程主机通过套接字(套接字由文件描述符标识)的地址标识。
- 当使用面向连接的协议时，这将建立连接。某些类型的协议是无连接的，尤其是用户数据报协议。当与无连接协议一起使用时，connect定义发送和接收数据的远程地址，允许使用send和recv等函数。在这些情况下，connect函数阻止接收来自其他源的数据报。
- connect()返回一个表示错误代码的整数:0表示成功，-1表示错误。从历史上看,在bsd获得系统中,一个套接字描述符定义的状态如果电话连接失败(因为它是单一Unix中指定的规范),因此,便携式应用程序应该立即关闭套接字描述符和获得一个新的与套接字描述符(),在调用connect()失败。

## C/S使用TCP连接的例子
### Server
建立一个简单的TCP服务器需要以下步骤:
1. 创建一个TCP套接字，并调用套接字()。
2. 使用bind()调用将套接字绑定到侦听端口。在调用bind()之前，程序员必须声明一个sockaddr_in结构，清除它(使用memset())和sin_family (AF_INET)，并填充它的sin_port(监听端口，按网络字节顺序)字段。可以通过调用函数htons() (host to network short)将一个短int转换为网络字节顺序。
3. 准备套接字监听连接(使其成为侦听套接字)，并调用listen()。
4. 通过调用accept()接受传入连接。让服务器进入阻塞，直到接收到传入连接，然后返回接受连接的套接字描述符。初始描述符仍然是侦听描述符，并且可以在任何时候使用这个套接字再次调用accept()，直到它关闭为止。
5. 与远程主机通信，可以通过send()和recv()或write()和read()来完成。
6. 最后，使用close()关闭已打开的每个套接字(一旦不再需要)。

### 下面的程序在端口号1100上创建一个TCP服务器:
```
  #include <sys/types.h>
  #include <sys/socket.h>
  #include <netinet/in.h>
  #include <arpa/inet.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <unistd.h>
  
  int main(void)
  {
    struct sockaddr_in sa;
    int SocketFD = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (SocketFD == -1) {
      perror("cannot create socket");
      exit(EXIT_FAILURE);
    }
  
    memset(&sa, 0, sizeof sa);
  
    sa.sin_family = AF_INET;
    sa.sin_port = htons(1100);
    sa.sin_addr.s_addr = htonl(INADDR_ANY);
  
    if (bind(SocketFD,(struct sockaddr *)&sa, sizeof sa) == -1) {
      perror("bind failed");
      close(SocketFD);
      exit(EXIT_FAILURE);
    }
  
    if (listen(SocketFD, 10) == -1) {
      perror("listen failed");
      close(SocketFD);
      exit(EXIT_FAILURE);
    }
  
    for (;;) {
      int ConnectFD = accept(SocketFD, NULL, NULL);
  
      if (0 > ConnectFD) {
        perror("accept failed");
        close(SocketFD);
        exit(EXIT_FAILURE);
      }
  
      /* 执行读写操作 ... 
      read(ConnectFD, buff, size)
      */
  
      if (shutdown(ConnectFD, SHUT_RDWR) == -1) {
        perror("shutdown failed");
        close(ConnectFD);
        close(SocketFD);
        exit(EXIT_FAILURE);
      }
      close(ConnectFD);
    }

    close(SocketFD);
    return EXIT_SUCCESS;  
}
```

### Client
编写TCP客户机应用程序涉及以下步骤:
1. 创建一个TCP套接字，并调用socket()。
2. 使用connect()连接到服务器，传递一个sockaddr_in结构，将sin_family设置为AF_INET，将sin_port设置为端点正在监听的端口(以网络字节顺序)，并将sin_addr设置为监听服务器的IP地址(也以网络字节顺序)。
3. 通过使用send()和recv()或write()和read()与服务器通信。
4. 终止连接并通过调用close()清理。
```
  #include <sys/types.h>
  #include <sys/socket.h>
  #include <netinet/in.h>
  #include <arpa/inet.h>
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <unistd.h>
  
  int main(void)
  {
    struct sockaddr_in sa;
    int res;
    int SocketFD;

    SocketFD = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (SocketFD == -1) {
      perror("cannot create socket");
      exit(EXIT_FAILURE);
    }
  
    memset(&sa, 0, sizeof sa);
  
    sa.sin_family = AF_INET;
    sa.sin_port = htons(1100);
    res = inet_pton(AF_INET, "192.168.1.3", &sa.sin_addr);

    if (connect(SocketFD, (struct sockaddr *)&sa, sizeof sa) == -1) {
      perror("connect failed");
      close(SocketFD);
      exit(EXIT_FAILURE);
    }
  
    /* 执行读写操作 ... */
  
    shutdown(SocketFD, SHUT_RDWR);
  
    close(SocketFD);
    return EXIT_SUCCESS;
  }
```

## C/S使用UDP连接的例子

### Server
应用程序可以在端口号7654上设置UDP服务器，如下所示。程序包含一个无限循环，使用recvfrom()接收UDP数据报。
```
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <unistd.h> /* close()函数的头文件，用来关闭socket */ 
#include <stdlib.h>

int main(void)
{
  int sock;
  struct sockaddr_in sa; 
  char buffer[1024];
  ssize_t recsize;
  socklen_t fromlen;

  memset(&sa, 0, sizeof sa);
  sa.sin_family = AF_INET;
  sa.sin_addr.s_addr = htonl(INADDR_ANY);
  sa.sin_port = htons(7654);
  fromlen = sizeof sa;

  sock = socket(PF_INET, SOCK_DGRAM, IPPROTO_UDP);
  if (bind(sock, (struct sockaddr *)&sa, sizeof sa) == -1) {
    perror("error bind failed");
    close(sock);
    exit(EXIT_FAILURE);
  }

  for (;;) {
    recsize = recvfrom(sock, (void*)buffer, sizeof buffer, 0, (struct sockaddr*)&sa, &fromlen);
    if (recsize < 0) {
      fprintf(stderr, "%s\n", strerror(errno));
      exit(EXIT_FAILURE);
    }
    printf("recsize: %d\n ", (int)recsize);
    sleep(1);
    printf("datagram: %.*s\n", (int)recsize, buffer);
  }
}
```
### Client
一个简单的客户端程序，发送一个UDP包，其中包含字符串“Hello World!”发送到地址127.0.0.1和端口号7654:
```
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <unistd.h>
#include <arpa/inet.h>

int main(void)
{
  int sock;
  struct sockaddr_in sa;
  int bytes_sent;
  char buffer[200];
 
  strcpy(buffer, "hello world!");
 
  /* 创建一个套接字使用UDP协议 */
  sock = socket(PF_INET, SOCK_DGRAM, IPPROTO_UDP);
  if (sock == -1) {
      /* 如果套接字初始化失败，退出 */
      printf("Error Creating Socket");
      exit(EXIT_FAILURE);
  }
 
  /* 零位套接字地址 */
  memset(&sa, 0, sizeof sa);
  
  /* 地址是IPv4的版本 */
  sa.sin_family = AF_INET;
 
   /* IPv4地址是一个uint32_t类型，将八位字符的字符串表示形式转换为适当的值 */
  sa.sin_addr.s_addr = inet_addr("127.0.0.1");
  
  /* 套接字是unsigned shorts类型，htons(x)确保x是在网络字节类型(强制转换)，设置端口为7654 */
  sa.sin_port = htons(7654);
 
  bytes_sent = sendto(sock, buffer, strlen(buffer), 0,(struct sockaddr*)&sa, sizeof sa);
  if (bytes_sent < 0) {
    printf("Error sending packet: %s\n", strerror(errno));
    exit(EXIT_FAILURE);
  }
 
  close(sock); /* 关闭套接字 */
  return 0;
}
```