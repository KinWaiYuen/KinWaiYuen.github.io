# fork

### fork
函数原型:pid_t pid = fork();
- 父进程拿到的pid >0 表示子进程pid
- 子进程返回0,表示自己是子进程
- 返回-1:gg


fork之后,子进程从fork位置向下执行
```ditaa
┌──────────────┐                          ┌──────────────┐         
│ 父进程        │                          │  子进程       │         
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
├──────────────┤────────▶fork>0:父进程     ├──────────────┤────────▶ fork=0:子进程.往下执行
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
├──────────────┤                          ├──────────────┤         
└──────────────┘                          └──────────────┘         
```

```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main()
{
    printf("before fork-1---\n");
    printf("before fork-2---\n");
    printf("before fork-3---\n");
    printf("before fork-4---\n");
    printf("before fork-5---\n");

    pid_t pid = fork();
    if(pid == -1){
        perror("fork err");
        exit(1);
    }
    else if(pid == 0){
        printf("child created\n");
        printf("=======end of child file\n");
    }
    else if(pid > 0){
        printf("parent proc my child is %d\n",pid);
        printf("=======end of father file\n");
    }


    return 0;
}

```
此处子进程和父进程的先后问题,纠结不大:
- 子进程先执行:因为cow机制,父进程还修改内存的话碎片多
- 父进程先执行:因为cpu已经让父进程拿到了,这个时候子进程先执行的话父进程还要有挂起的内核开销

纠纷点不用在意.内核开始的时候是子进程先执行,后来是父进程先执行.

### getpid getppid
获取pid; parent pid
getpid getppid

#### 并发fork
```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main()
{
    printf("before fork-1---\n");
    printf("before fork-2---\n");
    printf("before fork-3---\n");
    printf("before fork-4---\n");
    printf("before fork-5---\n");
    int pid_now = getpid();
    int i = 0;
    pid_t pid = 0;
    for(i = 0;i < 5; i++)
    {
        pid = fork();
        if(pid == 0) break;//子进程退出循环
        printf("father make a child.child pid=%d|father pid=%d\n",pid, getpid());
    }
    if(i == 5){
        printf("i am father\n");
    }
    else{
        printf("i am %dth child\n",i);
    }

    if(getppid() == pid_now){
        printf("=======end of child pid=%d ppid=%d\n",getpid(), getppid());
    }
    else {
        printf("=======end of father pid=%d ppid=%d\n", getpid(),getppid());
    }


    return 0;
}


```
因为fork之后没有wait,子进程有可能成为孤儿进程
```
➜  linux ./child_fork
before fork-1---
before fork-2---
before fork-3---
before fork-4---
before fork-5---
father make a child.child pid=9868|father pid=9867
i am 0th child
=======end of child pid=9868 ppid=9867
father make a child.child pid=9869|father pid=9867
father make a child.child pid=9870|father pid=9867
i am 1th child
=======end of child pid=9869 ppid=9867
father make a child.child pid=9871|father pid=9867
i am 2th child
father make a child.child pid=9872|father pid=9867
i am father
=======end of father pid=9867 ppid=4612
=======end of child pid=9870 ppid=9867
i am 4th child
=======end of father pid=9872 ppid=1
i am 3th child
=======end of father pid=9871 ppid=1
```
上面的ppid=1的就是孤儿进程,因为father先结束了,被init进程接管

而且也是没有顺序的.各个进程抢cpu.
加上wait之后能正常返回
```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main()
{
    printf("before fork-1---\n");
    printf("before fork-2---\n");
    printf("before fork-3---\n");
    printf("before fork-4---\n");
    printf("before fork-5---\n");
    int pid_now = getpid();
    int i = 0;
    pid_t pid = 0;
    for(i = 0;i < 5; i++)
    {
        pid = fork();
        if(pid == 0) break;//子进程退出循环
        printf("father make a child.child pid=%d|father pid=%d\n",pid, getpid());
        wait();
    }
    if(i == 5){
        printf("i am father\n");
    }
    else{
        printf("i am %dth child\n",i);
    }

    if(getppid() == pid_now){
        printf("=======end of child pid=%d ppid=%d\n",getpid(), getppid());
    }
    else {
        printf("=======end of father pid=%d ppid=%d\n", getpid(),getppid());
    }


    return 0;
}

```

返回

```
➜  linux ./child_fork
before fork-1---
before fork-2---
before fork-3---
before fork-4---
before fork-5---
father make a child.child pid=21363|father pid=21362
i am 0th child
=======end of child pid=21363 ppid=21362
father make a child.child pid=21364|father pid=21362
i am 1th child
=======end of child pid=21364 ppid=21362
i am 2th child
father make a child.child pid=21365|father pid=21362
=======end of child pid=21365 ppid=21362
father make a child.child pid=21366|father pid=21362
i am 3th child
=======end of child pid=21366 ppid=21362
i am 4th child
father make a child.child pid=21367|father pid=21362
=======end of child pid=21367 ppid=21362
i am father
=======end of father pid=21362 ppid=4612
```

但是这个时候不是并行的进程,而是串行,等到了子进程ok后再进行后续操作

### 进程共享
刚fork后,父子进程相同的点
- 全局变量
- .data
- test
- 堆
- 栈
- 环境变量
- 用户id
- 宿主目录
- 进程工作目录
- 信号处理方式

几乎3G的虚拟空间完全一样
不同的点:
- 进程id
- fork返回值
- 父进程id
- 进程运行时间
- 闹钟(定时器)
- 未决信号集

进程共享:
- fd
- mmap映射区

使用cow 写时复制的特性,只有在写的时候才会另外写,不需要全部复制.
//TODO cow的缺点

#### gdb跟踪父子进程

在gdb 二进制文件后,run到需要的断点.
- set follow-fork-mode child 后续fork后进入子进程
- set follow-fork-mode parent 后续fork后进入父进程
