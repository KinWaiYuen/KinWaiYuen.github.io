ipc,interprocess communication,进程间通信

## ipc机制
进程间通信本质是在内核中建立缓冲区(一般大小4096),让不同进程在内核空间进行写/读

```ditaa

+-----------------------------------------+
|                   kernel                |
|             +------------+              |
|      +----->|    缓冲区   |-------+      |
|      |      +------------+       |      |
|      |                           |      |
|      |                           |      |
+------+------+-------------+------+------+
|      |      |             |      |      |
|      |      |             |      |      |
|      |      |             |      |      |
|      |      |             |      |      |
|      |      |             |      |      |
|      |      |             |      v      |
|   +----+    |             |   +----+    |
|   |    |    |             |   |    |    |
|   +----+    |             |   +----+    |
| process 1   |             |   process 2 |
|             |             |             |
|             |             |             |
|             |             |             |
+-------------+             +-------------+

```

ipc很多
- 文件
- 管道
- 信号
- 共享内存
- 消息队列
- socket
- 命名管道 没有血缘关系的也能用
- ...

现在常用的
- fifo(命名管道)最简单 只能血缘关系的,父子,兄弟,子孙等 没血缘关系的不能
- 信号 开销小 携带数据量有限,内存/cpu占用小,但是快
- 共享内存 无血缘关系 
- 本地socket 最稳定 实现复杂度最高

## pipe
需要有血缘关系.调用pipe 或者 mkfifo 
- 本质是伪文件
- 由两个fd引用,一个表示读,一个表示写 
- 规定数据写端流入,读端流出

原理:内核使用环状队列机制,借助内核缓冲区(4k)实现 
局限:
- 不能进程自己写自己读
- 数据不可反复读取,读走数据就不存在 相当于队列的出队
- 半双工通信,只能单方向流动
- 只能在公共祖先的进程之间使用pipe



 通信类型
- 单双工 单向发送信号  固定接收方和发送方 ex:遥控器
- 双向半双工 两端指定了方向后只能一个方向流动 发送方和接收方可以指定 但是指定只有不能变 ex:对讲机 pipe
- 双向全双工 可以两端进出 ex:电话 socket

int pipe(int pipefd[2])
- 创建并打开管道  pipefd[0]读 pipefd[1]写
- 成功:0
- 失败:-1 errno

父子进程读写
```ditaa
                            ┌──────────────────────────┐      
         ┌─────────────────▶│   r      pipe       w    │◀─┐   
         │                  └──────────────────────────┘  │   
         │                                                │   
  ┌─────────────┐                                         │   
  │ proc father │─────────────────────────────────────────┘   
  └─────────────┘                                             
                                                              
                                │                             
                              fork                            
                         for shared fds                       
                                │                             
                                │                             
                                │                             
                                ▼                             
                                                              
 ┌─────────────┐                                              
 │  proc son   │────────────────────────────────────────┐     
 └─────────────┘                                        │     
        │                                               │     
        │                                               │     
        │                 ┌──────────────────────────┐  │     
       ┌┴────────────────▶│   r      pipe       w    │◀─┤     
       │                  └──────────────────────────┘  │     
       │                                                │     
┌─────────────┐                                         │     
│ proc father │─────────────────────────────────────────┘     
└─────────────┘                                               
                               │                              
                               │                              
                               │                              
                        father cut read                       
                         son cut write                        
                               │                              
                               │                              
                               │                              
                               │                              
                               ▼                              
      ┌─────────────┐                                         
      │  proc son   │                                         
      └─────────────┘                                         
             │                                                
             │                                                
             │                 ┌──────────────────────────┐   
             └────────────────▶│   r      pipe       w    │◀─┐
                               └──────────────────────────┘  │
                                                             │
     ┌─────────────┐                                         │
     │ proc father │─────────────────────────────────────────┘
     └─────────────┘                                          
```

实现
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>

void sys_err(const char* str){
    perror(str);
    exit(1);
}

int main()
{
    int ret;
    int fd[2];
    ret = pipe(fd);
    if(ret == -1){
        sys_err("pipe err");
    }
    pid_t pid = fork();
    if(pid > 0){
        close(fd[0]);
        write(fd[1],"hello pipe",sizeof("hello pipe"));
        close(fd[1]);
    }else if(pid == 0){
        close(fd[1]);
        char buf[1024];
        int size = read(fd[0], buf,sizeof(buf));
        write(STDOUT_FILENO, buf, size);
        close(fd[0]);
    }
    else{
        sys_err("fork err");
    }
    return 0;
}

```

#### 管道读写行为
管道可能会不够用,内核会扩容
读管道
- 有数据,返回读到字节数
- 没数据:
> - 写端全部关闭,返回0 相当于读到文件尾部
> - 写端没全部关闭,阻塞等待,让出cpu(挂起)
 
|写端|管道有数据|管道没数据|
|---|---|---|
|全部关闭|读数据,返回字节数(不管,读了再说)|返回0(没数据也没人写,不管)|
|没有全部关闭|读数据,返回字节数(不管,读了再说)|阻塞(管道没数据,但是有进程可能写)|
总结:有得读就读,没得读就看下是不是关了写,写都关了就不读(都没人写,读毛),写还没关就阻塞(可能别人还写啊)

写管道
- 读端全部关闭,进程异常(收到SIGPIPE信号,进程不终止)
- 读端没有全部关闭
> - 管道满:阻塞(少见)
> - 管道未满:write写入,返回实际写入字节数

|读端|管道未写满|管道写满|
|---|---|---|
|未全部关闭|写(还能写就写)|阻塞(等读的消耗了才能写,环状队列就这么多)|
|全部关闭|进程异常(SIGPIPE,都没有进程读还写?)|进程异常(SIGPIPE,都没有进程读还写?)|
总结:有读接口没关就写,就看下怎么写.管道有空间就写,没空间就阻塞.读接口都关了,直接拿到SIGPIPE(都没人读你还写??)

如果写端不写,阻塞后关闭
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>

void sys_err(const char* str){
    perror(str);
    exit(1);
}

int main()
{
    int ret;
    int fd[2];
    ret = pipe(fd);
    if(ret == -1){
        sys_err("pipe err");
    }
    pid_t pid = fork();
    if(pid > 0){
        close(fd[0]);
        sleep(2);
        /* write(fd[1],"hello pipe\n",sizeof("hello pipe\n")); */
        close(fd[1]);
    }else if(pid == 0){
        close(fd[1]);
        char buf[1024];
        int size = read(fd[0], buf,sizeof(buf));
        printf("child read pipe size=%d\n",size);
        write(STDOUT_FILENO, buf, size);
        close(fd[0]);
    }
    else{
        sys_err("fork err");
    }
    return 0;
}
```
可以看到结果
```
➜  linux ./pipe
child read pipe size=0
```
过了3s只有返回,说明子进程阻塞等待,发现fd已经关闭,返回0

使用pipe进行ls|wc -l
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>

void sys_err(const char* str){
    perror(str);
    exit(1);
}

int main()
{
    int pipefd[2];
    int ret = pipe(pipefd);
    if(0 > ret){
        sys_err("pipe err");
    }
    pid_t pChild = fork();
    if(pChild < 0){
        sys_err("fork err");
    }
    else if(pChild == 0){//child
        close(pipefd[1]);
        ret = dup2(pipefd[0], STDIN_FILENO);
        if(ret < 0){
            sys_err("son fup2 err");
        }
        execlp("wc", "wc", "-l", NULL);

        sys_err("son wc err");
    }
    else{//father
        close(pipefd[0]);
        ret = dup2(pipefd[1], STDOUT_FILENO);
        if(ret < 0){
            sys_err("father fup2 err");
        }
        execlp("ls","ls", NULL);
        sys_err("father ps err");
    }
    return 0;
}

```
使用兄弟进程做ls wc
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>
#include <sys/wait.h>

void sys_err(const char* str){
    perror(str);
    exit(1);
}

int main()
{
    int pipefd[2];
    int ret = pipe(pipefd);
    if(0 > ret){
        sys_err("pipe err");
    }
    int i = 0;
    for(i  = 0; i < 2; i++)
    {
        pid_t pChild = fork();
        if(pChild < 0){
            sys_err("fork err");
        }
        if(pChild == 0){//child
            if(i == 1){
                printf("in little bro\n");

                close(pipefd[1]);
                ret = dup2(pipefd[0], STDIN_FILENO);
                if(ret < 0){
                    sys_err("son fup2 err");
                }
                execlp("wc", "wc", "-l", NULL);

                sys_err("son wc err");
            }
            else if(i == 0)
            {

                printf("in big bro\n");
                close(pipefd[0]);
                ret = dup2(pipefd[1], STDOUT_FILENO);
                if(ret < 0){
                    sys_err("father fup2 err");
                }
                execlp("ls","ls", NULL);
                sys_err("father ps err");

            }
        }
        else{
            continue;
        }

    }
    close(pipefd[0]);
    close(pipefd[1]);

    while((ret = wait(NULL)) < 0){}

    return 0;
}
```

允许pipe有多个写端,多个读端,但不能保证读和写的顺序.但是这个场景不能多用,可能乱序
(当然允许,fork的时候就有多个读端多个写端)

管道缓冲区大小
使用ulimit -a看当前系统管道文件以及内核缓冲区大小
stack只有8192k 所以很多时候只能malloc


## FIFO
因为pipe只能有血缘关系才能使用,fifo出现解决没有血缘能做pipe能力的事情
叫命名管道.pipe是非命名管道

虽然进程用户空间独立,但是内核区共享.共享区创建缓存,进程之间进行共享.参考[ipc机制](##ipc机制)
命令行  mkfifo 
函数 int mkfifo(const char* pathname, mode_t mode);

fifo和文件的用法类似,指定路径下读取/写入管道达到进程之间的通信目的
测试:
```cpp
//写
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>
#include <fcntl.h>

void sys_err(const char* str){
    perror(str);
    exit(1);
}

int main(int argc, char* argv[])
{
    int fd, i;
    char buf[4096];
    if(argc < 2){
        printf("input filename\n");
        return -1;
    }
    fd = open(argv[1], O_WRONLY);
    if(fd < 0){
        perror("open");
    }
    i = 0;
    while(1){
        sprintf(buf, "hello i = %d\n",i++);
        write(fd, buf,strlen(buf));
        sleep(1);
    }
    close(fd);

    return 0;
}

//读
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>
#include <fcntl.h>

void sys_err(const char* str){
    perror(str);
    exit(1);
}

int main(int argc, char* argv[])
{
    int fd, i ,len;
    char buf[4096];
    if(argc < 2){
        printf("input filename\n");
        return -1;
    }
    fd = open(argv[1], O_RDONLY);
    if(fd < 0){
        perror("open");
    }
    i = 0;
    while(1){
        len = read(fd,buf,sizeof(buf));
        write(STDOUT_FILENO, buf,len);
        sleep(3);
    }
    close(fd);

    return 0;
}
```
一读多写:写顺序不能保证 全部出队
多读一些:取出顺序不能保证,但是进程1读了的进程2读不到

## mmap
可以直接使用文件,没有血缘的进程进行通信
ex:a写入,b lseek到文件头然后读

内存映射直接把磁盘文件映射到内存中,修改内存直接修改磁盘文件.

创建共享内存映射
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
- addr 指定映射区首地址 通常传NULL,让系统自动分配
- length 共享内存映射区大小
- prot protection 表示映射内存的保护属性 属性位&组合
> - PROT_EXEC 页可能执行
> - PROT_READ 页可能读
> - PROT_WRITE 页可能写
> - PROT_NONE 页不能访问
