dup,用于重定向
cat a > b 
把写到屏幕的内容写入到b.

dup
int dup(int oldfd) 返回新的fd

int dup2(int oldfd, int newfd) 此处注意:
- 正常情况下,newfd会先关闭,然后再把oldfd的dentry让newfd指过去,然后返回
- 如果oldfd非法,newfd不会被关闭
- 如果oldfd和newfd指向的denntry一样,dup2不做其他操作,返回newfd

拿到的fd和oldfd是一样的,操作什么的是相同的.
具体是pcb中fd表的ptr指向了同样的dentry
```ditaa
┌─────────┐          ┌───────────────────┐
│fd list1 │          │  open file table  │
│┌───┬───┐│          │                   │
││fd │ptr││          │ ┌────┬────┬─────┐ │
│├───┼───┤│          │ │diff│stat│inode│ │
││fd0│ptr├┼────┐     │ ├────┼────┼─────┤ │
│├───┼───┤│    └┬────┼─▶    │    │     │ │
││fd1│ptr││     │    │ ├────┼────┼─────┤ │
│├───┼───┤│     │    │ │    │    │     │ │
││fd3│ptr├dup(fd0)   │ ├────┼────┼─────┤ │
│├───┼───┤│          │ │    │    │     │ │
││...│...││          │ ├────┼────┼─────┤ │
│└───┴───┘│          │ │    │    │     │ │
│         │          │ ├────┼────┼─────┤ │
└─────────┘          │ │    │    │     │ │
                     │ ├────┼────┼─────┤ │
                     │ │    │    │     │ │
                     │ └────┴────┴─────┘ │
                     └───────────────────┘
```

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>

int main(int argc, char* argv[])
{
    int fd1 = open(argv[1], O_RDWR);
    int fd2 = open(argv[2], O_RDWR);
    int fdret = dup2(fd1, fd2);
    printf("fdret=%d\n",fdret);
    write (fd2, "123123", 6);
    return 0;
}


```

虽然fdret=4(符合预期,0 1 2占用,3是fd1),但是fd1最后是有写入的
cat a > b 意思就是把这时候的stdout_fileno要重定向给b这个fd,所以就是dup2(fd,STDOUT_FILENO);
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>

int main(int argc, char* argv[])
{
    int fd1 = open(argv[1], O_RDWR);
    int fdret = dup2(fd1, STDOUT_FILENO);
    printf("fdret=%d\n",fdret);
    printf("----------testdup2-------cat-------\n");

    return 0;
}


```
 这里可以最后的printf函数内容都会在fd1的文件中

 fcntl也可以做类似dup的操作,flag要设置,最后参数表示希望拿到>=多少的fd
 ```cpp
 #include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>

int main(int argc, char* argv[])
{
    int fd1 = open(argv[1], O_RDWR);
    printf("fd1=%d\n",fd1);
    int fd2 = fcntl(fd1, F_DUPFD,0);//最后入参0这个fd已经被占用,返回的是当前可用最小的fd
    printf("fd2=%d\n",fd2);
    int fd3 = fcntl(fd1, F_DUPFD,7);//最后入参7这个fd未占用,返回>=7的最小描述符.此时7能
    printf("fd3=%d\n",fd3);
    int iret = write(fd2,"fd2\n",4);
    printf("iret=%d",iret);
    if(iret < 0)
    {
        perror("write err");
        exit(1);
    }
    iret = write(fd3,"fd3\n",4);
    printf("iret=%d",iret);
    if(iret < 0)
    {
        perror("write err");
        exit(1);
    }

    return 0;
}


```
使用不同的fd写入,能看到文件顺序写入了对应的字符.