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