不使用多路复用:
- 阻塞
- 非阻塞忙轮询

使用多路IO复用:响应式,感知到fd变更

# select
```ditaa
┌─────────────────────────────────────┐
│                                     │
│                                     │
│        _______. _______  __         │
│       /       ||   ____||  |        │
│      |   (----`|  |__   |  |        │
│       \   \    |   __|  |  |        │
│   .----)   |   |  |____ |  `----.   │
│   |_______/    |_______||_______|   │
│                                     │
│    _______   ______ .___________.   │
│   |   ____| /      ||           |   │
│   |  |__   |  ,----'`---|  |----`   │
│   |   __|  |  |         |  |        │
│   |  |____ |  `----.    |  |        │
│   |_______| \______|    |__|        │
│                                     │
│                                     │
│                                     │
└─────────────────────────────────────┘
```

int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);
- nfds是最大的fd+1.select内部对传入fd做循环,nfds是循环的上限. 机制是对fd+1以内的挨个轮训
- readfds 传入传出参数.读事件
- writefds 传入传出参数.写事件
- exceptfds 传入传出参数.异常事件
- timeout阻塞时长. NULL:阻塞.0:不阻塞.

返回:返回所有有变化的fd总个数.
- 大于0:个数
- =0 当前没有fd变化
- <0异常

### fd_set操作
void FD_CLR(int fd, fd_set *set); fd从set清空.如果fd被close了,可以除移.
int  FD_ISSET(int fd, fd_set *set); fd是否在set中.判断回参集合是否有关心的fd
void FD_SET(int fd, fd_set *set); fd设置到set中.设置监听fd的时候用到.
void FD_ZERO(fd_set *set); set清0

### 使用select实现简单的svr
注意事项:
- select的时候如果监听到lfd,应该需要把cfd放入到allset中,后续继续select
- 如果仅仅是lfd改动,此处可以直接跳过其他cfd的操作,到下次再进行处理


```cpp
#include "wrap.h"
#include <unistd.h>

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
    int opt  = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    bzero(&svr_addr, sizeof(svr_addr));
    svr_addr.sin_family = AF_INET;
    int port = atoi(argv[1]);
    printf("====svr:port=%d======\n", port);
    svr_addr.sin_port = htons(port);
    svr_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    Bind(lfd,(struct sockaddr *)&svr_addr, sizeof(svr_addr) );
    Listen(lfd,128);


    fd_set rset, allset;
    int maxfd = lfd;
    int nready;
    FD_ZERO(&allset);
    FD_SET(lfd, &allset);
        int i = 0, n, j;
        char buf[4069];
    while(1){

        rset = allset;//allset后续会进行更新,添加cfd.这里rset需要调整
        nready = select(maxfd + 1, &rset, NULL, NULL, NULL);//select等待
        if(nready < 0){
            sys_err("select err");
        }

        //在lfd事件循环中
        if(FD_ISSET(lfd, &rset)){
            struct sockaddr_in cli_addr;
            socklen_t cli_addr_size = sizeof(cli_addr);
            printf("listen cli succ.now accept\n");

            cfd = Accept(lfd, (struct sockaddr *)&cli_addr, &cli_addr_size);
            printf("accept cli succ.fd = %d\n",cfd);

            FD_SET(cfd, &allset);//allset加入cfd
            if(maxfd < cfd){
                maxfd = cfd;
            }
            if(1 == nready){//如果只有nready 一个,说明只是来了一个监听事件.但是如果不是,说明还有cfd事件.所以要-1,后面cfd处理的时候需要
                continue;
            }
        }

        for(i= lfd + 1; i <= maxfd; i++){
            if(FD_ISSET(i, &rset)){
                n = Read(i,buf,sizeof(buf));
                if(n == 0){
                    printf("cfd %d gg\n",cfd); 
                    Close(i);
                    FD_CLR(i, &allset);
                }
                else if(n == -1){
                    sys_err("read err");
                }
                else {
                    printf("read buf = %d|buf=%s\n", n,buf);

                    for(j = 0; j < n; j++){
                        buf[j] = toupper(buf[j]);
                    }
                    Write(i,buf,n);
                    Write(STDOUT_FILENO, buf, n);
                }
            }
        }

    }
    Close(lfd);

    return 0;
}
```

上述逻辑总需要把所有的fd都轮训一次.事实上可能有这种情况:fd最大有一个,但是中间很多fd已经close了.这种实现效率比较低.
原因:select只有FD_ISSET用来确认是否在相应的fd集合中.这里使用方应该管理好目前活跃或者目标的fd.
//TODO 用链表或者vector优化.
目标是不用把所有的fd都遍FD_ISSET.
简单分析:因为目标的fd都应该得到遍历,这种情况应该链表比较合适.如果遍历到已经close的cfd,直接删除就可以.使用vector的删除涉及到挪动以及扩容缩容.
使用数组标记可以实现,但是每次都需要遍历max cfd个才能确保所有活跃的cfd被遍历完.直接使用链表性能应该更高.

#### 优缺点
- 缺点
    - 监听上限受fd限制,最大1024.(这个是select自己的监听上限)
    - 检测满足条件的fd,需要自己管理返回的fd_set.(但是性能和poll类似)
- 优点
    - 唯一一个可以跨平台完成fd监听 win linux unix 类unix 
    - 无法直接定位监听fd,都需要调用侧排查


# poll
```ditaa
┌─────────────────────────────────────────┐
│                                         │
│                                         │
│ .______     ______    __       __       │
│ |   _  \   /  __  \  |  |     |  |      │
│ |  |_)  | |  |  |  | |  |     |  |      │
│ |   ___/  |  |  |  | |  |     |  |      │
│ |  |      |  `--'  | |  `----.|  `----. │
│ | _|       \______/  |_______||_______| │
│                                         │
│                                         │
│                                         │
└─────────────────────────────────────────┘
```
include <poll.h>
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
- fds 是数组的地址.pollfd成员如下:
    - fd 监听的fd. **如果这个fd是负数,poll的时候会忽略.**
    - events 请求监听的事件 POLLIN POLLOUT POLLERR 读,写,异常
    - revents 结果监听到的事件.传入是给0,对应满足事件>0 POLLIN POLLOUT POLLERR 使用&操作
- nfds 监听数组的实际有效监听个数.传入fds这个数组的长度.
- timeout 超时时长. 毫秒
    - -1,阻塞等待
    - 大于0 超时时长
    - 0 非阻塞


返回
- 大于0 符合监听事件的fd总个数
- 0 没有
- 小于0 gg

### 注意点
- poll的结构体数组里的fd如果是-1,是可以被poll的,只是poll的时候不处理.

实例代码
```cpp
#include "wrap.h"
#include <unistd.h>
#include <poll.h>
#define OPEN_MAX 1024

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
    int opt  = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    bzero(&svr_addr, sizeof(svr_addr));
    svr_addr.sin_family = AF_INET;
    int port = atoi(argv[1]);
    printf("====svr:port=%d======\n", port);
    svr_addr.sin_port = htons(port);
    svr_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    Bind(lfd,(struct sockaddr *)&svr_addr, sizeof(svr_addr) );
    Listen(lfd,128);


    int maxfd = lfd;
    int nready;
    int i = 0, n, j;
    char buf[4069];
    struct pollfd pfd[OPEN_MAX];
    pfd[0].events = POLLIN;
    pfd[0].fd = lfd;
    for(i = 1; i < 1024;i++)
    {
        pfd[i].fd = -1;
    }

    int iMaxCliFdNum = 0;
            struct sockaddr_in cli_addr;
            socklen_t cli_addr_size = sizeof(cli_addr);
            char* str;
    while(1){
        //在lfd事件循环中
        nready = poll(pfd, iMaxCliFdNum + 1, -1);
        if(pfd[0].revents & POLLIN)
        {
            printf("listen cli succ.now accept\n");
            cfd = Accept(lfd, (struct sockaddr *)&cli_addr, &cli_addr_size);
            printf("accept cli succ.fd = %d ip=%s port=%d\n",
                   cfd, inet_ntop(AF_INET, &cli_addr.sin_addr, str, sizeof(str)), ntohs(cli_addr.sin_port));
            for (i =1; i < OPEN_MAX; i++){
                if(pfd[i].fd < 0){
                    pfd[i].fd = cfd;
                    break;
                }
            }

            if(i == OPEN_MAX){
                sys_err("too many cli\n");
            }

            pfd[i].events = POLLIN;
            if(i > iMaxCliFdNum){
                iMaxCliFdNum = i;
            }



            if(1 == nready){//如果只有nready 一个,说明只是来了一个监听事件.但是如果不是,说明还有cfd事件.所以要-1,后面cfd处理的时候需要
                continue;
            }
        }

        for(i=  1; i <=iMaxCliFdNum; i++){
            if((pfd[i].fd) > 0 && (pfd->events & POLLIN))
            {
                n = Read(pfd[i].fd,buf,sizeof(buf));
                if(n == 0){
                    printf("cfd %d gg\n",cfd);
                    pfd[i].fd=-1;
                }
                else if(n == -1){
                    sys_err("read err");
                }
                else {
                    printf("read buf = %d|buf=%s\n", n,buf);

                    for(j = 0; j < n; j++){
                        buf[j] = toupper(buf[j]);
                    }
                    Write(pfd[i].fd,buf,n);
                    Write(STDOUT_FILENO, buf, n);
                }

            }
        }

    }
    Close(lfd);

    return 0;
}
```
//TODO 发现使用poll的会存在情况:多次请求之后只得到一次的触发.

### poll优缺点
- 优点
    - 自带特性数组,根据特性监听
    - 拓展监听上限,打破1024上限(进程fd上限受内核限制.ulimit -n可以看到fd上限.可以在程序中使用setrlimit (RLIMIT_NOFILE, &fdLimit)调整)

- 缺点
    - 只能linux使用,跨平台gg
    - 无法直接定位监听fd,都需要调用侧排查


# epoll
```ditaa
┌──────────────────────────────────────────────────┐
│                                                  │
│                                                  │
│                                                  │
│                                                  │
│  _______ .______     ______    __       __       │
│ |   ____||   _  \   /  __  \  |  |     |  |      │
│ |  |__   |  |_)  | |  |  |  | |  |     |  |      │
│ |   __|  |   ___/  |  |  |  | |  |     |  |      │
│ |  |____ |  |      |  `--'  | |  `----.|  `----. │
│ |_______|| _|       \______/  |_______||_______| │
│                                                  │
│                                                  │
│                                                  │
│                                                  │
└──────────────────────────────────────────────────┘
```
### 机制
epoll_create的时候会建立**红黑树**,需要监听的fd放入节点中挂在红黑树上.根据关心的事件(读,写,异常)返回
返回的时候返回结构体数组,数组中只会有当前已经有变更的fd,不像select和poll那样还要对fd做再次筛选.
include <sys/epoll.h>
int epoll_create(int size)的时候size是这个红黑树的节点数.但是是内核不会因为这个上限后续不增加节点.如果超过这个size,内核可能会为之扩容.(内核会预先为epoll这个红黑树创建节点??//TODO 确认)
返回监听红黑树的树根


int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
对该监听红黑树所做的操作.增加节点/删除节点/修改节点  删除节点意味着取消监听.
- epfd eopll的红黑树fd
- op 操作类型 EPOLL_CTL_ADD  EPOLL_CTL_DEL EPOLL_CTL_MOD
- fd 监听的fd
- event eopll_event结构.保存包括fd,关心事件类型信息
**注意**
这里挂在树上的时候,把fd和epoll_event都写上了.实际上epoll_event.data的内容是**用户自己定义**的.后续实例代码写了fd,但是查看epoll_event内容
```cpp
           typedef union epoll_data {
               void        *ptr;
               int          fd;
               uint32_t     u32;
               uint64_t     u64;
           } epoll_data_t;

           struct epoll_event {
               uint32_t     events;      /* Epoll events */
               epoll_data_t data;        /* User data variable */
           };
```
所以往树上挂的时候**根据实际使用需求写入data数据**

```ditaa
                   ┌────┐                 
                   │epfd│                 
                   └────┘                 
                      │                   
            ┌─────────┴──────────┐        
            ▼                    ▼        
         ┌────┐               ┌────┐      
         │lfd │               │cfd1│      
         └───┬┴──────┐        └──┬─┴─────┐
            ││EPOLLIN│           │EPOLLIN│
   ┌────────┘├───────┤           ├───────┤
   │         │   3   │           │   4   │
   ▼         └───────┘           └───────┘
┌────┐                                    
│cfd2│                                    
└──┬─┴─────┐                              
   │EPOLLIN│                              
   ├───────┤                              
   │   5   │                              
   └───────┘                              
```


int epoll_wait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout);
- epfd 同上
- events 是epoll_event数组.这个数组是用于回参使用,获取epoll事件的数组.
- maxevnets events的上限大小,不是这个数组的大小.
- timeout 超时.



测试代码
```cpp
#include "wrap.h"
#include <unistd.h>
#include <sys/epoll.h>

#define OPEN_MAX 5000


int main(int argc, char* argv[])
{
    if(argc < 2){
        printf("usage: bin port\n");
        return 0;
    }


    int lfd, cfd, efd;
    lfd = Socket(AF_INET, SOCK_STREAM, 0);


    struct sockaddr_in svr_addr;
    /* memset(&svr_addr, 0, sizeof(svr_addr)); */
    int opt  = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    bzero(&svr_addr, sizeof(svr_addr));
    svr_addr.sin_family = AF_INET;
    int port = atoi(argv[1]);
    printf("====svr:port=%d======\n", port);
    svr_addr.sin_port = htons(port);
    svr_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    Bind(lfd,(struct sockaddr *)&svr_addr, sizeof(svr_addr) );
    Listen(lfd,128);


    struct epoll_event tEp, tEpRes[OPEN_MAX];
    efd = epoll_create(OPEN_MAX);//efd是一个fd,底层是红黑树
    if(efd == -1){
        sys_err("create epoll err");
    }

    tEp.events = EPOLLIN;
    tEp.data.fd = lfd;
    int res = epoll_ctl(efd, EPOLL_CTL_ADD, lfd, &tEp );//把lfd挂在efd上.
    if(res == -1){
        sys_err("epoll ctrl err");
    }

    int socketfd;
    int nready;
    int maxfd = lfd;
    char buf[4069];
    int i, j;
    struct sockaddr_in cli_addr;
    socklen_t cli_addr_size = sizeof(cli_addr);
    char* str;
    int n;
    while(1){
        nready = epoll_wait(efd, tEpRes, OPEN_MAX, -1);
        if(nready == -1){
            sys_err("epoll_wait err");
        }

        for(i = 0; i < nready; i++){
            if(!(tEpRes[i].events & EPOLLIN)){
                continue;
            }
            if(tEpRes[i].data.fd == lfd){

                printf("listen cli succ.now accept\n");
                cfd = Accept(lfd, (struct sockaddr *)&cli_addr, &cli_addr_size);
                printf("accept cli succ.fd = %d ip=%s port=%d\n",
                       cfd, inet_ntop(AF_INET, &cli_addr.sin_addr, str, sizeof(str)), ntohs(cli_addr.sin_port));
                tEp.events = EPOLLIN;
                tEp.data.fd = cfd;
                res = epoll_ctl(efd, EPOLL_CTL_ADD, cfd, &tEp);
                if(res == -1){
                    sys_err("epoll ctrl err");
                }
            }
            else{
                socketfd = tEpRes[i].data.fd;
                n = Read(tEpRes[i].data.fd,buf,sizeof(buf));
                if(n == 0){
                    printf("cfd %d gg\n",cfd);
                    Close(tEpRes[i].data.fd);
                    res = epoll_ctl(efd, EPOLL_CTL_DEL, socketfd, NULL);
                    if(res == -1){
                        sys_err("epoll del err");
                    }
                    printf("cfd %d gg\n", socketfd);
                }
                else if(n == -1){
                    sys_err("read err");
                }
                else {
                    printf("read fd=%d buf = %d|buf=%s\n",socketfd, n,buf);

                    for(j = 0; j < n; j++){
                        buf[j] = toupper(buf[j]);
                    }
                    Write(socketfd,buf,n);
                    Write(STDOUT_FILENO, buf, n);
                }

            }
        }
    }





    Close(lfd);

    return 0;
}
```

### epoll优缺点
能显著提高大量并发连接中只有少量获取情况下cpu的利用率
ex:当前有3个连接,fd是3 500 1000.select,poll的机制是在所有这些fd中进行轮询,每个都看一次是否活跃,然后返回列表.这样效率低下,浪费cpu
实际网络场景经常是有链接活跃少的情况.ex:打开网页,服务器连接已经打开,但是操作并没有很多.比如会浏览,然后再偶尔进行再次操作.





### 突破1024 fd上限
使用命令`cat /proc/sys/fs/file-max`当前计算机所能打开的最大fd个数.受硬件限制.
`ulimit -a`当前用户下的进程默认打开fd上限.默认是1024
如果需要,可以更改配置文件
`vim /etc/security/limits.conf`进行修改.
```
*               soft    core            10000
*               hard    rss             20000
```
soft可以通过命令进行修改,但是修改上限不能超过hard值.
修改后要**重新登录**
然后使用ulimit -n 2000进行修改,改到2000
ulimit -n调整后,不能继续往上调,只能往下调.如果需要上调**需要用户重新登录**

## epoll 事件模式
```ditaa
           ───LT────│       
           │        │       
           │        │       
           ET      ET       
           │        │       
───────────│        ───LT──▶
默认使用LT 水平触发 level trigger
ET edge trigger 边沿触发
```

实例
```cpp
#include "wrap.h"
#include <unistd.h>
#include <sys/epoll.h>

#include <sys/signal.h>
#define OPEN_MAX 5000

#define MAXLINE 10
int pfd[2];
pid_t pid;
void cat_sig(int iSig){
    if(iSig == SIGQUIT){
        printf("catch SIGQUIT sig %d start =======\n", iSig);
        printf("catch SIGQUIT sig %d end ========\n", iSig);
    }
    if(iSig == SIGINT){

        printf("catch SIGINT sig %d start =======\n", iSig);
        if(getpid() != pid)
        {
            char buf[4096];
            int len = Read(pfd[0], buf, sizeof(buf));
            printf("now in pipe = %s|len=%d\n", buf,len);
        }
        printf("catch SIGINT sig %d end ========\n", iSig);
        exit(0);
    }
    else{

        printf("catch sig %d\n", iSig);
    }
    return;
}

int main(int argc, char* argv[])
{
    int  i;
    char buf[MAXLINE];
    char ch = 'a';
    pipe(pfd);
    pid = fork();
    if(pid < 0){
        sys_err("fork err");
    }
    else if(pid == 0){
        Close(pfd[0]);
        while(1){
        struct sigaction act, oldact;

        act.sa_handler = cat_sig;
        sigemptyset(&(act.sa_mask));
        act.sa_flags = 0;

        int ret = sigaction(SIGINT, &act, &oldact);
        if(ret == -1){
            sys_err("sigaction err");
        }
            for(i = 0; i < MAXLINE/2; i++){
                buf[i] = ch;
            }
            buf[i-1] = '\n';
            ch++;
            for(;i < MAXLINE; i++){
                buf[i]=ch;
            }
            buf[i-1] = '\n';
            ch++;
            printf("before write buf=%s\n",buf);
            Write(pfd[1], buf, MAXLINE);
            printf("write buf=%s succ\n",buf);
            sleep(3);
        }
    }
    else if(pid > 0){
        struct sigaction act, oldact;

        act.sa_handler = cat_sig;
        sigemptyset(&(act.sa_mask));
        act.sa_flags = 0;

        int ret = sigaction(SIGINT, &act, &oldact);
        if(ret == -1){
            sys_err("sigaction err");
        }

        struct epoll_event event, resevent[10];

        int efd = epoll_create(10);
        event.data.fd = pfd[0];
        if(argc == 2){
            printf("in et stage\n");
            event.events = EPOLLIN|EPOLLET;
        }
        else{
            printf("in lt stage\n");
            event.events = EPOLLIN;
        }

        epoll_ctl(efd, EPOLL_CTL_ADD, pfd[0], &event);
        int res, len;

        while(1){
            res = epoll_wait(efd,resevent,10,-1);
            printf("epoll wait res %d\n",res);
            if(resevent[0].data.fd == pfd[0]){
                len = read(pfd[0], buf, MAXLINE/2);
                write(STDOUT_FILENO, buf, len);
            }
            printf("after epoll wait ====\n");
        }

        close(pfd[0]);
        close(efd);
    }
    return 0;
}
```
每次写入的是10个字符,只读到5个.
如果是et,读端**只有在pipe写的时候才会触发**,因此每次只读5个,pipe中总有剩余的字符没有被读取.
```
➜  socket ./et 0
in et stage
before write buf=aaaa
bbbb

write buf=aaaa
bbbb
 succ
epoll wait res 1
aaaa
after epoll wait ====
^Ccatch SIGINT sig 2 start =======
catch SIGINT sig 2 start =======
now in pipe = bbbb
|len=5
catch SIGINT sig 2 end ========
now in pipe = /|len=-1
catch SIGINT sig 2 end ========
```
可以看到退出的时候pipe里是有数据的.


如果是lt,读端**只要在pipe有数据的时候就触发**,因此每次读5个,但是在循环中还是会被触发.
```
➜  socket ./et
in lt stage
before write buf=aaaa
bbbb

write buf=aaaa
bbbb
 succ
epoll wait res 1
aaaa
after epoll wait ====
epoll wait res 1
bbbb
after epoll wait ====
^Ccatch SIGINT sig 2 start =======
now in pipe = /|len=-1
catch SIGINT sig 2 end ========
catch SIGINT sig 2 start =======
```
这样退出的时候没有数据.

小结:
ET 边沿触发  缓冲区剩余**未读尽**的数据**不会**导致**epoll_wait**返回.**新的事件满足才会触发**.
LT 水平触发  缓冲区剩余**未读尽**的数据**会**导致**epoll_wait**返回

测试过,对于这种情况: a写,b读.c读.b用epoll读.如果c读的时候不会导致b的ET触发.
注意:
**ET模式是高效模式,只支持非阻塞模式**.fcntl设置fd的非阻塞
使用ET的时候accept的逻辑应该如下
```cpp
int old_option = fcntl(listenfd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(listenfd, F_SETFL, new_option);

while ((connfd = accept(listenfd, (struct sockaddr*)& client_address,
                                        &client_addrlen)) > 0) {
                    // 在这里注册新的连接
                    ev.data.fd = connfd;
                    ev.events &= 0;
                    ev.events |= EPOLLIN | EPOLLET;
                    epoll_ctl(epollfd, EPOLL_CTL_DEL, connfd, &ev);
                   
                } 
```
- 优点
    - 高效,突破1024
- 缺点
    - 不能跨平台

LT模式:cli写500字节,svr读可能读不完,要全部度.剩下继续读.->LT
ET模式:cli写500字节,svr不需要全部读,读里面200字节就够->ET

ET非阻塞 -->忙轮询,读完数据.

### 非阻塞用ET
知乎找的:
设想这样的情况。
你使用了epoll，且你的fd是阻塞的。
如果epoll通知你fd可以写数据了。然后你啦啦啦的打算写100B数据，但是内核buf只有50B的可用
空间，这时候，你的进程就被阻塞了。。。
所以说，用了epoll不能保证你的进程在读写的时候不会阻塞

实验:
svr
```cpp
#include "wrap.h"
#include <unistd.h>
#include <sys/epoll.h>

#define OPEN_MAX 5000


int main(int argc, char* argv[])
{
    if(argc < 2){
        printf("usage: bin port\n");
        return 0;
    }


    int lfd, cfd, efd;
    lfd = Socket(AF_INET, SOCK_STREAM, 0);


    struct sockaddr_in svr_addr;
    /* memset(&svr_addr, 0, sizeof(svr_addr)); */
    int opt  = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    bzero(&svr_addr, sizeof(svr_addr));
    svr_addr.sin_family = AF_INET;
    int port = atoi(argv[1]);
    printf("====svr:port=%d======\n", port);
    svr_addr.sin_port = htons(port);
    svr_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    Bind(lfd,(struct sockaddr *)&svr_addr, sizeof(svr_addr) );
    Listen(lfd,128);


    struct epoll_event tEp, tEpRes[OPEN_MAX];
    efd = epoll_create(OPEN_MAX);//efd是一个fd,底层是红黑树
    if(efd == -1){
        sys_err("create epoll err");
    }

    tEp.events = EPOLLIN|EPOLLET;
    tEp.data.fd = lfd;
    int res = epoll_ctl(efd, EPOLL_CTL_ADD, lfd, &tEp );//把lfd挂在efd上.
    if(res == -1){
        sys_err("epoll ctrl err");
    }

    int socketfd;
    int nready;
    int maxfd = lfd;
    char buf[4069];
    int i, j;
    struct sockaddr_in cli_addr;
    socklen_t cli_addr_size = sizeof(cli_addr);
    char* str;
    int n;
    while(1){
        nready = epoll_wait(efd, tEpRes, OPEN_MAX, -1);
        if(nready == -1){
            sys_err("epoll_wait err");
        }

        for(i = 0; i < nready; i++){
            if(!(tEpRes[i].events & EPOLLIN)){
                continue;
            }
            if(tEpRes[i].data.fd == lfd){

                printf("listen cli succ.now accept\n");
                cfd = Accept(lfd, (struct sockaddr *)&cli_addr, &cli_addr_size);
                if(argc == 3){
                    int bufsize = atoi(argv[2]);
                    socklen_t optlen = sizeof(bufsize);
                    if((setsockopt(cfd, SOL_SOCKET, SO_RCVBUF,&bufsize,optlen))<0)
                    {
                        sys_err("set socket len err");
                    }
                    int nowbufsize = -1;
                    socklen_t nowoptlen = sizeof(bufsize);
                    if((getsockopt(cfd, SOL_SOCKET, SO_RCVBUF,&nowbufsize,&nowoptlen))<0)
                    {
                        sys_err("set socket len err");
                    }
                    printf("now sock leng=%d\n", nowbufsize);


                }
                printf("accept cli succ.fd = %d ip=%s port=%d\n",
                       cfd, inet_ntop(AF_INET, &cli_addr.sin_addr, str, sizeof(str)), ntohs(cli_addr.sin_port));
                tEp.events = EPOLLIN|EPOLLET;
                tEp.data.fd = cfd;
                res = epoll_ctl(efd, EPOLL_CTL_ADD, cfd, &tEp);
                if(res == -1){
                    sys_err("epoll ctrl err");
                }
            }
            else{
                socketfd = tEpRes[i].data.fd;
                /* n = Read(tEpRes[i].data.fd,buf,sizeof(buf)); */
                n = Read(tEpRes[i].data.fd,buf,5);
                if(n == 0){
                    printf("cfd %d gg\n",cfd);
                    Close(tEpRes[i].data.fd);
                    res = epoll_ctl(efd, EPOLL_CTL_DEL, socketfd, NULL);
                    if(res == -1){
                        sys_err("epoll del err");
                    }
                    printf("cfd %d gg\n", socketfd);
                }
                else if(n == -1){
                    sys_err("read err");
                }
                else {
                    printf("read fd=%d buf = %d|buf=%s\n",socketfd, n,buf);

                    for(j = 0; j < n; j++){
                        buf[j] = toupper(buf[j]);
                    }
                    Write(socketfd,buf,n);
                    Write(STDOUT_FILENO, buf, n);
                }

            }
        }
    }





    Close(lfd);

    return 0;
}
```
cli

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
#include <sys/signal.h>
#include "wrap.h"

#define BUFSIZE 4096
    int cfd;

void cat_sig(int iSig){
    if(iSig == SIGQUIT){
        printf("catch SIGQUIT sig %d start =======\n", iSig);
        sleep(4);
        printf("catch SIGQUIT sig %d end ========\n", iSig);
    }
    if(iSig == SIGINT){

        printf("catch SIGINT sig %d start =======\n", iSig);
        shutdown(cfd,2);
        close(cfd);

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
    struct sigaction act, oldact;
    act.sa_handler = cat_sig;
    sigemptyset(&(act.sa_mask));
    act.sa_flags = 0;

    int ret = sigaction(SIGINT, &act, &oldact);
    if(ret == -1){
        sys_err("sigaction err");
    }

    cfd = socket(AF_INET, SOCK_STREAM, 0);
    if(cfd==-1){
        sys_err("socket err");
    }
    int opt  = 1;
    setsockopt(cfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    struct sockaddr_in t_addr;
    struct sockaddr_in t_svr;
    t_svr.sin_family = AF_INET;
    t_svr.sin_port = htons(9999);
    t_svr.sin_addr.s_addr = htonl(INADDR_ANY);
    socklen_t t_svr_size = sizeof(t_svr);

    socklen_t *t_cli_size;
    t_cli_size = (socklen_t * )malloc(sizeof(socklen_t));
    *t_cli_size = sizeof(struct sockaddr_in);

    t_addr.sin_family= AF_INET;
    if(argc >= 2)
    {
        int port = atoi(argv[1]);
        bzero(&t_addr, sizeof(t_addr));
        t_addr.sin_family = AF_INET;
        printf("====svr:port=%d======\n", port);
        t_addr.sin_port = htons(port);
        t_addr.sin_addr.s_addr = htonl(INADDR_ANY);
        Bind(cfd,(struct sockaddr *)&t_addr, sizeof(t_addr) );

    }
    /* t_addr.sin_port = htons(9999); t_addr.sin_addr.s_addr = htonl(INADDR_ANY); */
    ret =connect(cfd, (struct sockaddr*)&t_svr, t_svr_size);
    if(ret == -1)
    {
        sys_err("socket err");

    }
    char b[4096];
        int i  = 0;
        int maxbuf=0;
        if(argc == 3){
            maxbuf = atoi(argv[2]);
        }
        for(i = 0;i < maxbuf; i ++)
        {
            b[i]='a';
        }

    while(1){

        write(cfd, b,maxbuf);
        printf("write succ. now read\n");
        ret = read(cfd, b, sizeof(b));
        if(ret){
            printf("%s\n", b);
        }
        sleep(1);
    }

    return 0;
}
```
可以看到:cli是最后不能发送到的(需要制定每次发较大的内容),从[tcpdump](../tools/tcpdump.md)看到:最后cli不停发送包,但是svr没有回包.因为svr还没处理完数据.cli最后一次填满之后,svr侧由于et后续没有触发到read动作.因此cli的cfd写阻塞.

所以**ET要和忙轮询**一起处

### epoll反应堆模型
epoll ET
非阻塞
void *ptr

留意epoll_wait:
```cpp
int epoll_wait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout);
```
epoll_event定义:
```cpp
           typedef union epoll_data {
               void    *ptr;
               int      fd;
               uint32_t u32;
               uint64_t u64;
           } epoll_data_t;

           struct epoll_event {
               uint32_t     events;    /* Epoll events */
               epoll_data_t data;      /* User data variable */
           };
```
epoll_wait的返回判断data.fd.和ptr用同一块内存.
epoll_wait有相应的时候自动回调->反应堆模式

#### 普通流程vs反应堆流程
普通流程:
- socket 
- bind 
- listen 
- epoll_create epoll_ctl(EPOLL_CTL_ADD)
- while(1)
    - eopll_wait
    - fd有事件
    - 判断监听事件数组
        - lfd满足
            - Accept
            - epoll_create epoll_ctl(EPOLL_CTL_ADD, cfd)
        - cfd满足
            - read
                - 读gg:epoll_ctl(EPOLL_CTL_DEL, cfd) 不监听
            - logic
            - write

反应堆模型
- socket 
- bind 
- listen 
- epoll_create epoll_ctl(EPOLL_CTL_ADD)
- while(1)
    - eopll_wait
    - fd有事件
    - 判断监听事件数组
        - lfd满足
            - Accept
            - epoll_ctl(EPOLL_CTL_ADD, cfd)
        - cfd读满足
            - read
                - 读gg:epoll_ctl(EPOLL_CTL_DEL, cfd) 不监听
            - epoll_ctl(EPOLL_CTL_DEL, cfd)
            - epoll_ctl(EPOLL_CTL_ADD, cfd.EPOLLOUT)
        - cfd满足写
            - write
            - epoll_ctl(EPOLL_CTL_DEL, cfd)
            - epoll_ctl(EPOLL_CTL_ADD, cfd.EPOLLIN)

为什么使用反应堆:
网络环境复杂,对端可能半关闭,或者对端可能滑动窗口很小,阻塞.
监听写事件然后再写.可读的情况下再写,有两个好处
- 不需要写阻塞.如果对端没有及时处理数据,socket本端可能写阻塞.
- 提高cpu效率.阻塞会大大浪费系统资源.

反应堆例子
```cpp
#include "wrap.h"
#include <unistd.h>
#include <sys/epoll.h>
#include <fcntl.h>
#include "react.h"

#define OPEN_MAX 5000

#define BUFLEN 4096
#define MAX_EVENTS 5000
#define SVR_PORT 9999

struct myevent_s{
    int fd;
    int events;
    void *arg;
    void (*call_back)(int fd, int events, void* arg);
    int status;//是否在红黑树上
    char buf[BUFLEN];
    int len;
    long last_active;//加入红黑树时间 过期之后从红黑树拿掉
};

int g_efd;
struct myevent_s g_events[MAX_EVENTS + 1];//for lfd
struct epoll_event events[MAX_EVENTS + 1];

//events表示epoll关心的EPOLLIN EPOLLOUT
void eventadd(int efd, int events, struct myevent_s* myEv){
    struct epoll_event epv = {0,{0}};
    int op = 0;
    epv.data.ptr = myEv;
    epv.events = myEv->events = events;

    if(myEv->status == 0){
        op= EPOLL_CTL_ADD;
        myEv->status = 1;
    }

    if(epoll_ctl(efd, op, myEv->fd, &epv) < 0){
        printf("event add fail [fd=%d],events[%d]\n", myEv->fd, events);
    }
    else{
        printf("myEvnet add succ [fd=%d], op=%d, events[%0X]\n",myEv->fd,op, events);
    }
    return;
}


void eventset(struct myevent_s *ev, int fd, void(*call_back)(int, int, void*), void *arg){
    ev->fd = fd;
    ev->call_back = call_back;
    ev->events = 0;
    ev->arg = arg;
    ev->status = 0;
    if(ev->len == 0){
        bzero(ev->buf, sizeof(ev->buf));
    }
    /* bzero(ev->buf, sizeof(ev->buf)); */
    /* ev->len = 0; */
    ev->last_active = time(NULL);
    return;
}
void eventdel(int efd, struct myevent_s *myEv){
    struct epoll_event epv = {0,{0}};
    if(myEv->status != 1){
        return;
    }
    myEv->status = 0;

    epv.data.ptr = NULL;
    epoll_ctl(efd, EPOLL_CTL_DEL, myEv->fd , &epv);
    printf("fd %d is removed in epoll list\n",myEv->fd);
    return;
}

void senddata(int fd, int events, void *arg){
    struct myevent_s *myEv = (struct myevent_s*) arg;
    int len;
    printf("[line:%d] before send data.fd=%d, buf=%s, sizeof(buf)=%d\n",
           __LINE__,fd, myEv->buf, myEv->len);
    len = send(fd, myEv->buf, myEv->len ,0);
    eventdel(g_efd, myEv);
    if(len > 0){
        printf("send fd=%d len=%lu, buf=%s",myEv->fd ,sizeof(myEv->buf ), myEv->buf );
        eventset(myEv, fd, rcvddata,myEv);
        eventadd(g_efd, EPOLLIN, myEv);
    }
    else{
        close(myEv->fd );
        printf("send fd=%d err. err=%s\n", fd,strerror(errno));
    }
    return;
}

void rcvddata(int fd, int events, void *arg){
    struct myevent_s *myEv = (struct myevent_s *)arg;
    int len;
    printf("ready to read. cfd = %d\n",fd);
    len = recv(fd, myEv->buf, sizeof(myEv->buf),0);

    eventdel(g_efd, myEv);

    if(len > 0){
        myEv->len = len;
        myEv->buf[len] = '\0';
        printf("[line:%d] fd=%d read data size=%d:%s\n",__LINE__, myEv->len, fd, myEv->buf);

        eventset(myEv, fd, senddata, myEv);
        eventadd(g_efd, EPOLLOUT, myEv);
    }
    else if(len == 0){
        close(myEv->fd);
        printf("fd:%d pos:%ld closed\n",fd, myEv - g_events);
    }
    else{
        close(myEv->fd);
        printf("recv fd:%d errno:%d  err:%s\n",fd, errno, strerror(errno));
    }

}

void acceptconn(int lfd, int events, void *arg){
    struct sockaddr_in cin;
    socklen_t len =  sizeof(cin);
    int cfd, i;
    if((cfd= accept(lfd, (struct sockaddr*)&cin, &len)) == -1){
        if(errno != EAGAIN && errno != EINTR){

        }
        sys_err("accept err");
    }

    do{
        for(i = 0; i < MAX_EVENTS; i++){
            if(g_events[i].status == 0){
                break;
            }
        }

        if(i == MAX_EVENTS){
            printf("%s:max conn limit[%d]\n",__func__, MAX_EVENTS);
            break;
        }

        int flag = 0;
        if((flag = fcntl(cfd, F_SETFL, O_NONBLOCK)) < 0){
            printf("%s: fcntl nonblocking fail,%s\n",__func__, strerror(errno));
        }

        printf("event list %d added.fd=%d\n",i,cfd);
        eventset(&g_events[i], cfd, rcvddata, &g_events[i]);
        eventadd(g_efd, EPOLLIN, &g_events[i]);
    }while(0);
    printf("new connect [%s:%d][time:%ld],pos[%d]\n",
           inet_ntoa(cin.sin_addr), ntohs(cin.sin_port), g_events[i].last_active,i);
    return;
}




void initSocketListen(int efd, unsigned short port){

    int lfd = Socket(AF_INET, SOCK_STREAM, 0);
    fcntl(lfd, F_SETFL, O_NONBLOCK);


    int opt  = 1;
    setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in svr_addr;
    /* memset(&svr_addr, 0, sizeof(svr_addr)); */
    bzero(&svr_addr, sizeof(svr_addr));//初始化socket

    svr_addr.sin_family = AF_INET;
    svr_addr.sin_port = htons(port);//根据port 转为网络字节序
    /* svr_addr.sin_addr.s_addr = htonl(INADDR_ANY);//使用本机地址 */
    svr_addr.sin_addr.s_addr = htonl(INADDR_ANY);//使用本机地址

    Bind(lfd,(struct sockaddr *)&svr_addr, sizeof(svr_addr) );
    Listen(lfd,128);

    eventset(&g_events[MAX_EVENTS], lfd, acceptconn, &g_events[MAX_EVENTS]);
    eventadd(efd, EPOLLIN, &g_events[MAX_EVENTS]);

    printf("===lfd added on epoll.svr:port=%d======\n", port);
    return;
}


int main(int argc, char* argv[])
{
    unsigned short port = SVR_PORT;
    if(argc == 2){
        port = atoi(argv[1]);
    }

    g_efd = epoll_create(MAX_EVENTS + 1);
    if(g_efd <=0){
        sys_err("epoll_create err");
    }

    initSocketListen(g_efd,port);

    int checkpos = 0, i;
    while(1){
        long lNow = time(NULL);
        for(i = 0; i < 100; i++, checkpos ++){
            if(checkpos == MAX_EVENTS){
                checkpos = 0;
            }
            if(g_events[checkpos].status != 1){
                continue;//事件列表不在红黑树上
            }
            long lDuration = lNow - g_events[checkpos].last_active;
            if(lDuration >= 60){
                Close(g_events[checkpos].fd);
                printf("===fd %d gg====\n", g_events[checkpos].fd);
                eventdel(g_efd, &g_events[checkpos]);
            }
        }

        int nfd = epoll_wait(g_efd, events, MAX_EVENTS+1,1000);
        if(nfd < 0){
            sys_err("epoll wait err");
        }

        for(i = 0; i < nfd;i++){
            struct myevent_s *ev = (struct myevent_s*)events[i].data.ptr;
            if((events[i].events & EPOLLIN) && (ev->events & EPOLLIN)){
                ev->call_back(ev->fd, events[i].events, ev->arg);

            }
            if((events[i].events & EPOLLOUT) && (ev->events & EPOLLOUT)){
                ev->call_back(ev->fd, events[i].events, ev->arg);
            }
        }


    }

    return 0;
}
```


