# exec
直接可以让子进程调用某个程序,进程的用户空间代码和数据被新程序替换
不创建新进程,pid不变

执行exec()之后,函数**不会返回**.直接就执行了exec指定的**程序**(是程序,不是函数).
执行exec()之后的进程空间就是执行程序的进程空间.
**注意**原进程pcb中信息是通过三级映射将虚拟空间映射在内存空间,exec删除进程pcb信息,从磁盘把新代码调入内存,再形成pcb里的映射.

```ditaa
   父                         子
┌───────────┐            ┌───────────┐                
│           ├───fork()───▶           │                
│           │            │           │       ┌───────┐
│           │            │           ├exec()▶│ b.out │
│           │            │           │       └───────┘
│   a.out   │            │   a.out   │                
│           │            │           │                
│           │            │           │                
│           │            │           │                
│           │            │           │                
│           │            │           │                
└───────────┘            └───────────┘                
                  │                                   
            after exec()                              
                  │                                   
                  │                                   
                  │                                   
                  ▼                                   
 ┌───────────┐            ┌───────────┐               
 │           │            │           │               
 │           │            │           │               
 │           │            │           │               
 │           │            │           │               
 │   a.out   │            │   b.out   │               
 │           │            │           │               
 │           │            │           │               
 │           │            │           │               
 │           │            │           │               
 │           │            │           │               
 └───────────┘            └───────────┘               
 ```

exec函数族
- int execl(const char* path, const char* arg, ...);
- int execlp(const char* file, const char* arg,...);
- int execle(const char* path, const char* arg,...,char *const envp[]);
- int exevc(const char * path, char * const argv[]);
- int execvp(const char* file, char* const argv[]);
- int execve(const char* path, char* const argv[], char* const envp[]);

l list 命令行参数列表
p path 搜索file时候使用path环境变量
v vector 使用命令行参数数组
e environment 使用环境变量数组
execve才是系统调用,其他都是封装

注意点:
- arg参数是从argv[0]开始,所以这里的argv写的时候,例如调用ps 就先写ps;调用哪个二进制的话也要把路径写上
- 因为返回,但是exec调用出错会有errno记录,因此直接看调用后的errno即可知道调用是否成功


#### execlp
p:path 借助path环境变量
fork完之后调用系统程序多数使用execlp

```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main()
{
    pid_t pid = fork();
    if(pid == -1)
    {
        perror("fork err");
        exit(1);
    }else if(pid == 0){
         execlp("ls","ls","-l","-h", NULL);//这里的argv从argv[0]开始写.最后要加上哨兵NULL表示参数写完
         perror("exec err");//因为exec不会返回,如果执行出错才会到perror 因为error在exec失败才会设置
         exit(1);
    }else if(pid > 0){
       printf("i am father\n"); 
    }

    return 0;
}


```


#### execl 
执行普通二进制程序
```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

int main()
{
    pid_t pid = fork();
    if(pid == -1)
    {
        perror("fork err");
        exit(1);
    }else if(pid == 0){
         execl("./t","./t", NULL);//这里的argv从argv[0]开始写.最后要加上哨兵NULL表示参数写完
         //execl("/bin/ls","ls", "-l", NULL);//如果想用execl直接调用系统二进制,加上全路径即可
         perror("exec err");//因为exec不会返回,如果执行出错才会到perror 因为error在exec失败才会设置
         exit(1);
    }else if(pid > 0){
       printf("i am father\n"); 
    }

    return 0;
}


```

##### 写一个ps > 文件的函数
用到的操作:
1.dup2 把ps的结果重定向到文件
2.fork进程 execl ps

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <unistd.h>
#include <fcntl.h>//O_RDONLY所在
#include <errno.h>//errno所在

#define N 1
int main(int argc, char* argv[])
{
    int fd1=open(argv[1], O_RDWR|O_CREAT|O_TRUNC, 0664);
    if(fd1 < 0){
        perror("open argv1 err");
        exit(1);
    }
    int ret = dup2(fd1, STDOUT_FILENO);
    if(ret < 0){
        perror("dup fd err\n");
        exit(1);
    }
    pid_t pid = fork();
    if(pid < 0)
    {
        perror("fork err");
        exit(1);
    }
    else if(pid == 0)
    {
        execlp("ps", "ps", "aux",NULL);
        perror("execl err after fork");
        exit(1);
    }
    else{
        printf("i am father\n");
        wait();//父进程等待
    }




    close(fd1);
    return 0;
}


```

####  execvp
int execvp(const char* file, char* const argv[]);
传入argv的时候传的是数组
char* argv[] = {"ls", "-r", "-l",NULL};

