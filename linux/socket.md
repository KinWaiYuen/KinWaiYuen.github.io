# socket

通信过程中,socket是成对出现的.
(socket应为是插座)

socket内部是通过一个fd管理发送和接收两个缓冲区(发缓冲,写缓冲)
```ditaa
┌───────socket─fd───┐                        ┌───socket fd───────┐
│                   │                        │                   │
│ ┌─────────────┐   │                        │   ┌────────────┐  │
│ │ send buffer │───┼────────────────────────┼──▶│rcvd buffer │  │
│ └─────────────┘   │                        │   └────────────┘  │
│                   │                        │                   │
│                   │                        │                   │
│  ┌────────────┐   │                        │   ┌───────────┐   │
│  │rcvd buffer │◀──┼────────────────────────┼───│send buffer│   │
│  └────────────┘   │                        │   └───────────┘   │
│                   │                        │                   │
└───────────────────┘                        └───────────────────┘
```

网络中使用大端解包.因此需要一次网络字节序和主机字节序的转换.

#### ip地址转换
转换函数:
htonl //host to network long 本地转到网络(ip)
htons //host to network short 本地转网络(port)
ntohl //network to host long 网络转本地(ip)
ntohs //netowrk to host short 网络转本地(port)

现在的ip转换
inet_pton 本机字符串类型的ip转为网络字节序的二进制ip
inet_ntop 

 <arpa/inet.h>
int inet_pton(int af, const char *src, void *dst);//本地字节序->网络字节序
参数:
- af 指定当前ip是什么协议 只有AF_INET AF_INET6
- src ip地址 点分十进制
- dst 回参,转换(网络字节序)ip地址
回参
- 成功返回1
- 如果src入参无效,返回0

int const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
参数:
- af 同上
- src 网络字节序的ip地址
- dst 本地字节序ip
- size dst大小

返回
- 成功 dst
- 失败 NULL

### sockaddr
 
```cpp

sockaddr                    sockaddr_in                 sockaddr_un                 sockaddr_in6
                            AF_INET                     AF_UNIX                     AF_INET6
┌───────────────┐         ┌───────────────┐         ┌───────────────┐           ┌───────────────┐
│16bit addr type│         │16bit addr type│         │16bit addr type│           │16bit addr type│
├───────────────┤         ├───────────────┤         ├───────────────┤           ├───────────────┤
│               │         │  16bit port   │         │               │           │  16bit port   │
│               │         ├───────────────┤         │               │           ├───────────────┤
│               │         │   32bit IP    │         │               │           │  32bit flow   │
│               │         │    address    │         │               │           │     label     │
│  14byte addr  │         │               │         │               │           │               │
│     data      │         ├───────────────┤         │               │           ├───────────────┤
│               │         │               │         │               │           │               │
│               │         │8 bytes padding│         │               │           │               │
│               │         │               │         │   108bytes    │           │               │
│               │         └───────────────┘         │   pathname    │           │               │
│               │                                   │               │           │               │
└───────────────┘                                   │               │           │  128 bite IP  │
                                                    │               │           │    address    │
                                                    │               │           │               │
                                                    │               │           │               │
                                                    │               │           │               │
                                                    │               │           │               │
                                                    │               │           │               │
                                                    └───────────────┘           │               │
                                                                                ├───────────────┤
                                                                                │     32bit     │
                                                                                │   scope ID    │
                                                                                │               │
                                                                                └───────────────┘
```
sockaddr_in ipv4的地址
sockaddr_un unix套接字地址
sockaddr_in6 ipv6的地址

sockaddr地址结构
    struct sockaddr_in addr addr
    bind(fd, (struct sockaddr*)&addr, ...)

man 7 ip 查看相关

使用bind accept等之前需要先初始化sockaddr
struct socketaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port=htons(xxx);
addr.sin_addr.saddr = htonl(INADDR_ANY) 取出系统中有效的任意IP地址,默认是二进制类型
inet_pton(AF_INET,"xxx.xxx.xxx.xxx",);

## socket创建流程图
整个建立连接过程,**svr会有两个socket**,cli会有一个.

![](../imgs/network/sock_con.png)
socket()
```ditaa
                                                                          ┌───────────────┐              ┌───────────────┐             ┌───────────────┐                                            
                                                                          │   socket()    │              │      fd       │────────────▶│    socket     │                                            
                                                                          └───────────────┘              └───────────────┘             └───────────────┘                                            
                                                                                  │                                                                                                                 
                                                                                  │                                                                                                                 
                                                                                  │                                                                                                                 
                                                                                  │                                                    ┌───────────────┐                                            
                                                                                  │                                                    │               │                                            
                                                                                  │                                                    │               │                                            
                                                                                  ▼                                                    │               │                                            
                                                                          ┌───────────────┐               ┌───────────────┐            │    socket     │                                            
                                                                          │    bind()     │               │      fd       │───────────▶│               │                                            
                                                                          └───────────────┘               └───────────────┘            ├───────────────┤                                            
                                                                                  │                                                    │   addr info   │                                            
                                                                                  │                                                    │   ip + port   │                                            
                                                                                  │                                                    └───────────────┘                                            
                                                                                  │                                                                                                                 
                                                                                  │                                                                                                                 
                                                                                  │                                                                                                                 
                                                                                  │                                                    ┌───────────────┐                                            
                                                                                  │                                                    │               │                                            
 ┌───────────────┐                                                                ▼                                                    │               │                                            
 │               │                                                        ┌───────────────┐                                            │               │                                            
 │               │                                                        │   listen()    │                 ┌───────────────┐          │    socket     │                                            
 │               │                 ┌───────────────┐                      └───────────────┘                 │      fd       │─────────▶│               │                                            
 │    socket     │                 │   socket()    │                               │                        └───────────────┘          ├───────────────┤                                            
 │               │                 └───────────────┘                               │                                                   │   addr info   │                                            
 │               │                         │                                       │                                                   │   ip + port   │                                            
 │               │                         │                                       │                                                   ├───────────────┤                                            
 │               │                         │                                       │                                                   │  maximum of   │                                            
 └───────────────┘                         │                                       │                                                   │  connections  │                                            
                                           │                                       │                                                   └───────────────┘                                            
                                           │                                       │                                                                                                                
                                           │                                       │                                                                                                                
                                           │                                       │                                                  ┌───────────────┐                                             
                                           │                                       │                                                  │               │                                             
                                           │                                       │                                                  │               │                                             
                                           ▼                                       │                                                  │               │                                             
                                   ┌───────────────┐                               │                       ┌───────────────┐          │    socket     │                                             
                                   │   connect()   │                               │                       │      fd       │─────────▶│               │──┐                                          
                                   └───────────────┘                               │                       └───────────────┘          ├───────────────┤  │                                          
                                           │                                       │                                                  │   addr info   │  │                                          
                                           │                                       │                                                  │   ip + port   │  │                                          
                                           │                                       │                                                  ├───────────────┤  │                                          
                                           │                                       │                                                  │  maximum of   │  │                                          
                                           │                                       │                                                  │  connections  │  │                                          
                                           │                                       │                                                  └─────────────block for                                       
                                           │                                       │                                                          ▲     accepting                                       
                                           │                                       │                                                          │     connection                                      
                                           │                                       │                                                          └──────────┘                                          
                                           │                                       │                                                                                                                
                                           │                                       │                                                                                                                
                                           │                                       │                                                                                                                
                                           │                                       │                                                                                                                
                                           │                                       │                                                                                                                
                                           │                                       │                                 ┌───────────────┐                                                              
                                           ▼                                       │                  ┌─────────────▶│  syns_queue   │                                                              
                                      ┌─────────┐                                  │                  │              ├───────────────┤                                                              
                                      │SYN_SENT │╲                                 │                  │              │               │    max limited by /proc/sys/net/ipv4/tcp_max_syn_backlog     
                                      └─────────┘ ╲                                │                  │              ├───────────────┤                                                              
                                                   ╲───────────────────────╲       ▼                  │              │               │                                                              
                                                                            ╲ ┌─────────┐             │              ├───────────────┤                                                              
                                                                             ▼│SYN_RCVD │─────────────┘              │               │                                                              
                                                                             ╱└─────────┘                            └───────────────┘                                                              
                                                                            ╱      │                                                                                                                
                                                    ╱──────────────────────╱       │                                                                                                                
                                                   ▼                               │                                                                                                                
                                   ┌──────────────┐                                │                                                                                                                
         ┌─────────────────────────│ ESTABLISHED  │╲                               │                                                                                                                
         │                         └──────────────┘ ╲                              │                                                                                                                
         │                                 │         ╲                             │                                                                                                                
         │                                 │          ╲                            │                                                                                                                
         │                                 │           ╲───────────────╲           │                                                                                                                
         │                                 │                            ╲          ▼                                                                                                                
         │                                 │                             ╲ ┌──────────────┐                           ┌───────────────┐                                                             
         │                                 │                              ▼│ ESTABLISHED  │──────────────────────────▶│ accept_queue  │   max limited by (backlog, /proc/sys/net/core/somaxconn).   
         │                                 │                               └──────────────┘                           ├───────────────┤   param somaxconn can be set. backlog is 2nd param set in   
         │                                 │                                       │                                  │               │                           listen()                          
         │                                 │                                       │                                  ├───────────────┤                                                             
         │                                 │                                       │                                  │               │                                                             
         │                                 │                                       ▼                                  ├───────────────┤                                                             
         ▼                                 │                               ┌───────────────┐                          │               │─────────┐                                                   
 ┌───────────────┐                         │                               │   accept()    │                          └───────────────┘         │                                                   
 │               │                         │                               └───────────────┘                                                    ▼                                                   
 │               │                         │                                       │                                                    ┌───────────────┐                                           
 │               │                         │                                       │                                                    │               │                                           
 │    socket     │                         │                                       │                                                    │  socket for   │                                           
 │               │◀────────────────────────┼───────────────────────────────────────┼───────────────────────────────────────────────────▶│  connection   │                                           
 │               │                         │                                       │                                                    │               │                                           
 │               │                         │                                       │                                                    └───────────────┘                                           
 │               │                         │                                       │                                                            ▲                                                   
 └───────────────┘                         │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                        generate                                                
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                            │                                                   
                                           │                                       │                                                    ┌───────────────┐                                           
                                           │                                       │                                                    │               │                                           
                                           │                                       │                                                    │               │                                           
                                           │                                       │                                                    │               │                                           
                                           │                                       │                         ┌───────────────┐          │    socket     │                                           
                                           │                                       │                         │      fd       │─────────▶│               │                                           
                                           │                                       │                         └───────────────┘          ├───────────────┤                                           
                                           │                                       │                                                    │   addr info   │                                           
                                           │                                       │                                                    │   ip + port   │──┐                                        
                                           │                                       │                                                    ├───────────────┤  │                                        
                                           │                                       │                                                    │  maximum of   │  │                                        
                                           │                                       │                                                    │  connections  │  │                                        
                                           │                                       │                                                    └───────────────┘  │                                        
                                           ▼                                       ▼                                                            ▲          │                                        
                                   ┌───────────────┐                       ┌───────────────┐                                                    │block for │                                        
                                   │  do_logic()   │                       │  do_logic()   │                                                    └accepting ┘                                        
                                   └───────────────┘                       └───────────────┘                                                     connection                                         
                                           │                                       │                                                                                                                
                                           │                                       │                                                                                                                
                                           │                                       │                                                                                                                
                                           │                                       │                                                                                                                
                                           │                                       │                                                                                                                
                                           │                                       │                                                                                                                
┌───────────────┐                          │                                       │                                                                                                                
│               │                          │                                       │                                                                                                                
│               │                          │                                       │                                                   ┌───────────────┐                                            
│               │                          ▼                                       ▼                                                   │               │                                            
│    socket     │                  ┌───────────────┐                       ┌───────────────┐                                           │  socket for   │                                            
│               │                  │   cloese()    │                       │   cloese()    │                                           │  connection   │                                            
│               │                  └───────────────┘                       └───────────────┘                                           │               │                                            
│               │                                                                                                                      └───────────────┘                                            
│               │                                                                                                                                                                                   
└───────────────┘                                                                                                                                                                                                                                                                                                                             
```
注意:
- listen()只是设置了套接字的**最大连接上限**,并没有进行阻塞.
- accept()进行了阻塞,等待cli的连接.因此tcp握手是在accept阻塞的时候进行的
- accept()在成功返回的时候会返回一个已经和cli建立连接的socket.用于监听的fd继续监听,而返回的socket继续进行逻辑操作.(所以cli一个socket,svr2个socket)
- close的时候需要svr感知到cli侧close

<sys/types.h>          /* See NOTES */
<sys/socket.h>
#### socket()
int socket(int domain, int type, int protocol);
入参
- domain: AF_INET, AF_INET6, AF_UNIX
- type: 数据传输协议
> - SOCK_STREAM 流式 tcp
> - SOCK_DGRAM 报文 udp
- protocol: 0

返回值
- 成功:新socket对应的fd
- -1 errno
#### bind()
int bind(int sockfd, const struct sockaddr *addr,
                socklen_t addrlen);
parma
- sockfd 
- addr 地址结构体 绑定socket的addr.sin_family应该和建立socket时候的domain一致.

struct sockaddr_in addr;
addr.sin_family=AF_INET;
addr.sin_port=htons(xxxx);
addr.sin_addr.s_addr=htonl(INADD_ANY);

- addrlen addr的大小
返回
- 成功:0
- -1 errno

#### listen()
设置**同时与服务器建立连接的上限数**(同时进行3次握手的客户端个数)
int listen(int sockfd, int backlog);
param
- sockfd
- backlog 上限最大不能超过128.linux内核决定.不论传多大,最大不超过这个值.
返回
- 成功:0
- -1 errno

**listen不是用来阻塞监听的!**

#### accept()
阻塞等待客户端建立连接,成功返回一个与客户端成功连接的socket fd
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
入参中有sockfd,实际上accept返回的socket是依靠入参sockfd监听,靠这个fd的ip和端口和客户端建立连接.
入参
- sockfd 
回参
- addr 成功与服务器建立连接的cli的地址结构
- addrlen 回参addr长度 传入的是addr的大小,返回的是实际客户端cli的地址结构大小
返回
- 成功:0
- -1 errno


socklen_t clit_addr_len = sizeof(addr);
accept(fd, addr, &clit_addr_leln);

#### connect()
客户端用于连接服务端的动作
int connect(int sockfd, const struct sockaddr *addr,
                   socklen_t addrlen);
入参
- sockfd 客户端自己的fd
- addr 服务端的地址结构
- addrlen addr的大小

如果不使用 bind函数绑定客户端地址,采用"隐式绑定"

### test code
测试  server把内容都改成大写
测试的时候可以用命令**nc**
nc ip port
这时候会连接对应的port
服务端代码


```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
#include <arpa/inet.h>
#include <ctype.h>

#define BUFSIZE 4096
void sys_err(const char* str){
    perror(str);
    exit(1);
}

int main(int argc, char* argv[])
{
    int lfd = 0;
    lfd = socket( AF_INET, SOCK_STREAM, 0 );
    if(lfd== -1){
        sys_err("socket err");
    }
    struct sockaddr_in t_addr;
    struct sockaddr_in * t_cli_addr;
    t_cli_addr = (struct sockaddr_in *)malloc(sizeof(struct sockaddr_in));

    socklen_t *t_cli_size;
    t_cli_size = (socklen_t * )malloc(sizeof(socklen_t));
    *t_cli_size = sizeof(struct sockaddr_in);

    t_addr.sin_family= AF_INET;
    t_addr.sin_port = htons(9999);
    t_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    int ret = bind(lfd, (struct sockaddr* )&t_addr, sizeof(t_addr));
    if(ret== -1){
        sys_err("bind err");
    }
    int iBackLog = 3;
    ret = listen(lfd, iBackLog);
    if(ret== -1){
        sys_err("listen err");
    }

    int cfd= accept(lfd, (struct sockaddr* ) t_cli_addr, t_cli_size);
    if(cfd == -1){
        sys_err("accept err");
    }

    char buf[BUFSIZE];
    char cCliIp[100];
    printf("cli ip=%s,port=%d\n",
           inet_ntop(AF_INET, &t_cli_addr->sin_addr.s_addr, cCliIp, sizeof(cCliIp)),
           ntohs(t_cli_addr->sin_port));

    while(1){
        ret = read(cfd, buf, sizeof(buf));
        if(ret== -1){
            sys_err("read err");
        }

        printf("===svr read:%s\n", buf);

        int i = 0;
        for(i = 0; i < ret; i++){
            buf[i]= toupper(buf[i]);
        }
        write(cfd,buf,ret);
        if(ret== -1){
            sys_err("write err");
        }

    }


    return 0;
}


```
使用nc测试
```
➜  socket ./s
===svr read:sfds

===svr read:shit

^C
```
nc的测试
```
➜  socket ./s
cli ip=127.0.0.1,port=48888
===svr read:23fesf

===svr read:sdfsdf

^C
```
注意:
t_cli_size = (socklen_t * )malloc(sizeof(socklen_t));
*t_cli_size = sizeof(struct sockaddr_in);
这里如果sizeof没有设置,不能正确拿回ip port


使用客户端写
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
#include <arpa/inet.h>
#include <ctype.h>

#define BUFSIZE 4096
void sys_err(const char* str){
    perror(str);
    exit(1);
}

int main(int argc, char* argv[])
{

    int cfd;
    cfd = socket(AF_INET, SOCK_STREAM, 0);
    if(cfd==-1){
        sys_err("socket err");

    }
    struct sockaddr t_addr;
    struct sockaddr_in t_svr;
    t_svr.sin_family = AF_INET;
    t_svr.sin_port = htons(9999);
    t_svr.sin_addr.s_addr = htonl(INADDR_ANY);
    socklen_t t_svr_size = sizeof(t_svr);

    socklen_t *t_cli_size;
    t_cli_size = (socklen_t * )malloc(sizeof(socklen_t));
    *t_cli_size = sizeof(struct sockaddr_in);

    t_addr.sa_family= AF_INET;
    /* t_addr.sin_port = htons(9999); t_addr.sin_addr.s_addr = htonl(INADDR_ANY); */
    int ret =connect(cfd, (struct sockaddr*)&t_svr, t_svr_size);
    if(ret == -1)
    {
        sys_err("socket err");

    }
    char b[4096];
    while(1){
        scanf("%s",b);

        write(cfd, b,sizeof(b));

        ret = read(cfd, b, sizeof(b));
        if(ret){
            printf("%s\n", b);
        }
    }

    return 0;
}

```

注意:
fwrite和fread不能用在socket编程,因为这两个函数需要文件结构指针,socket没有.


## 简单服务(使用多进程)
要点:
- 父进程listen fd,如果监听成功,fork子进程,子进程处理通信逻辑
- 考虑到wait的阻塞情况,使用信号SIGCHLD,如果子进程gg,父进程通过信号回收子进程(没有子进程回收的版本会存在僵尸进程,因为子进程没有被wait)

封装了异常逻辑的函数

```cpp
//wrap.h
#ifndef __WRAP_H_
#define __WRAP_H_

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
#include <arpa/inet.h>
#include <ctype.h>
#include <sys/signal.h>
#include <strings.h>
#include <sys/types.h>
#include <sys/wait.h>
int Socket(int domain, int type, int protocol);
int Listen(int sockfd, int backlog);
int Accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
ssize_t Read(int fd, void *buf, size_t count);
ssize_t Write(int fd, const void *buf, size_t count);
int Close(int fd);
ssize_t Writen(int fd, const void *buf, size_t count);
ssize_t Readn(int fd, void *buf, size_t count);
int Bind(int sockfd, const struct sockaddr *addr,
         socklen_t addrlen);
void sys_err(const char* str);
#endif
```

```cpp
#include "wrap.h"

void sys_err(const char* str){
    perror(str);
    exit(1);
}


int Socket(int domain, int type, int protocol){
    int iRet;
    iRet= socket( AF_INET, SOCK_STREAM, 0 );
    if(iRet == -1){
        sys_err("socket err");
    }
    return iRet;
}

int Listen(int sockfd, int backlog){

    int iRet = listen(sockfd, backlog);
    if(iRet== -1){
        sys_err("listen err");
    }
    return 0;
}

int Bind(int sockfd, const struct sockaddr *addr,
         socklen_t addrlen)
{
    int iRet = bind(sockfd, addr,  addrlen);
    if(iRet== -1){
        sys_err("bind err");
    }
    return 0;

}

int Accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen)
{
    int n;
again:
    if((n = accept(sockfd, addr, addrlen)) < 0){
        if((errno == ECONNABORTED) || (errno == EINTR)|| (errno == ERRCONNRESET)){
            goto again;
        }
        else{
            sys_err("accept err");
        }
    }
    return n;

}

ssize_t Write(int fd, const void *buf, size_t count){
    ssize_t n;
again:
    if((n = write(fd, buf,count)) == -1){
        if(errno == EINTR){
            goto again;
        }
        else
        {
            return -1;
        }
    }
    return n;
}

int Close(int fd){
    int n;
    if((n = close(fd)) == -1){
        sys_err("close err");
    }
    return n;
}

ssize_t Read(int fd, void *buf, size_t count){
    ssize_t n;
again:
    if((n = read(fd, buf,count)) == -1){
        if(errno == EINTR)
            goto again;
        else
            return -1;
    }
    return n;
}

ssize_t Readn(int fd, void *buf, size_t count){
    ssize_t nRead;
    size_t nLeft=count;
    char* ptr;
    ptr = (char*)buf;
    while(nLeft > 0){

        if((nRead = read(fd, buf,count)) == -1){
            if(errno == EINTR)
                nRead = 0;
            else
                return -1;
        }
        else if(nRead == 0){
            break;
        }

        nLeft = count - nRead;
        ptr+=nRead;
    }
    return count - nLeft;

}

ssize_t Writen(int fd, const void *buf, size_t count){
    ssize_t nWrite;
    size_t nLeft = count;
    char* ptr;
    ptr = (char* )buf;
    while(nLeft > 0){
        if((nWrite = write(fd, buf,count)) == -1){
            if(errno == EINTR){
                nWrite = 0;
            }
            else
            {
                return -1;
            }
        }
        else if(nWrite == 0){
            break;
        }
        nLeft = count - nWrite;
        ptr += nWrite;
    }
    return count - nWrite;
}
```


```cpp
#include "wrap.h"

void cat_sig(int iSig){
    if(iSig ==SIGCHLD){
        printf("catch SIGQUIT sig %d start =======\n", iSig);
        pid_t pChildId= 0;

        while((pChildId = wait(NULL)) > 0){
            printf("child id %d over\n",pChildId);
        }
        printf("catch SIGQUIT sig %d end ========\n", iSig);
    }
    if(iSig == SIGINT){

        printf("catch SIGINT sig %d start =======\n", iSig);

        printf("catch SIGINT sig %d end ========\n", iSig);
        exit(1);
    }
    else{

        printf("catch sig %d\n", iSig);
    }
    return;
}

int main(int argc, char* argv[])
{
    if(argc < 2){
        printf("usage: bin port\n");
        return 0;
    }


    int lfd, cfd;
    lfd = Socket(AF_INET, SOCK_STREAM, 0);


    struct sockaddr_in svr_addr;
    /* memset(&svr_addr, 0, sizeof(svr_addr)); */
    bzero(&svr_addr, sizeof(svr_addr));
    svr_addr.sin_family = AF_INET;
    int port = atoi(argv[1]);
    printf("====svr:port=%d======\n", port);
    svr_addr.sin_port = htons(port);
    svr_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    Bind(lfd,(struct sockaddr *)&svr_addr, sizeof(svr_addr) );

    Listen(lfd,128);

    struct sockaddr_in cli_addr;
    socklen_t cli_addr_size = sizeof(cli_addr);
    pid_t pid;
    while(1){
        cfd = Accept(lfd, (struct sockaddr *)&cli_addr, &cli_addr_size);

        pid = fork();
        if(pid < 0){
            sys_err("foerk err");
        }
        else if(pid == 0){
            break;
        }else {
            struct sigaction act, oldact;
            act.sa_handler = cat_sig;
            sigemptyset(&(act.sa_mask));
            act.sa_flags = 0;

            int ret = sigaction(SIGCHLD, &act, &oldact);
            if(ret == -1){
                sys_err("sigaction err");
            }
            close(cfd);
        }

    }

    if(pid == 0){
        while(1){
            char buf[4096];
            int ret, i;
            close(lfd);
            ret = Read(cfd, buf, sizeof(buf));
            if(ret == 0){
                close(cfd);//这样连接不是还是存在?

                exit(1);
            }
            for(i = 0; i < ret; i++){
                buf[i]=toupper(buf[i]);
            }
            write(cfd, buf, ret);
            write(STDOUT_FILENO, buf,ret);

        }
    }



    return 0;
}
```

注意:在centos7中外部访问可能被防火墙屏蔽.使用

firewall-cmd --zone=public --add-port=80/tcp(永久生效再加上 --permanent)
可以添加端口
ex:
sudo firewall-cmd --zone=public --add-port=9999/tcp --permanent

//TODO 进程间共享fd? 应该只是刚fork的时候一样.否则子进程关闭fd会影响全局.  待验证

## 简单服务(多线程)
```cpp
#include "wrap.h"

void tfn(void *arg){
    while(1){
        char buf[4096];
        int ret, i;
        int cfd = (int)arg;
        printf("in sub thread.cfd=%d\n",cfd);
        ret = Read(cfd, buf, sizeof(buf));
        if(ret == 0){
            Close(cfd);//这样连接不是还是存在?

            pthread_exit((void*)0);
        }
        for(i = 0; i < ret; i++){
            buf[i]=toupper(buf[i]);
        }
        Write(cfd, buf, ret);
        Write(STDOUT_FILENO, buf,ret);

    }
    printf("thread %lu gg\n", pthread_self());
    pthread_exit((void*)0);
}


int main(int argc, char* argv[])
{
    if(argc < 2){
        printf("usage: bin port\n");
        return 0;
    }


    int lfd, cfd;
    lfd = Socket(AF_INET, SOCK_STREAM, 0);


    struct sockaddr_in svr_addr;
    /* memset(&svr_addr, 0, sizeof(svr_addr)); */
    bzero(&svr_addr, sizeof(svr_addr));
    svr_addr.sin_family = AF_INET;
    int port = atoi(argv[1]);
    printf("====svr:port=%d======\n", port);
    svr_addr.sin_port = htons(port);
    svr_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    Bind(lfd,(struct sockaddr *)&svr_addr, sizeof(svr_addr) );

    Listen(lfd,128);

    struct sockaddr_in cli_addr;
    socklen_t cli_addr_size = sizeof(cli_addr);
    pthread_t tid;
    while(1){
        cfd = Accept(lfd, (struct sockaddr *)&cli_addr, &cli_addr_size);

        int ret = pthread_create(&tid, NULL, tfn, (void *) cfd);
        if(ret < 0){
            sys_err("err");
        }
        pthread_detach(tid);
        printf("main thread cfd is %d\n", cfd);
        //close(cfd);//线程:因为线程共享fd,这里主线程不能关闭fd

    }




    pthread_exit((void*)0);
}

```
### 注意:主线程不要关闭fd,因为fd共享.

### 注意:read = 0是关闭的意义:
如果fd就是文件,对端没有关闭(就是打开)的时候,是会阻塞等待读的.
如果返回时0,说明对端已经关闭.socket中就是对端写缓冲已经gg,意味着已经发生了close.(事实上对端shutdown(fd,SHUT_WR)的时候,就会发起了close请求).
本端发现对端不能读了,自然fd在read的时候返回0.

### 注意:read返回-1要进一步区分
errno是EAGAIN或者EWOULDBLOCK表示读是非阻塞读,可以继续再读
errno是EINTR是被中断,被系统中断.是应该再等下的
errno是**ECONNRESET**表示当前连接被重置,很可能是在accept的时候对端没有回包ack,内核会重置.
errno是其他情况才要gg

