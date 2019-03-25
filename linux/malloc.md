malloc有特性是:如果超过128k,会mmap分配内存

上述都是猜测.
现状是 128*1024 - 24 在堆上分配
128*1024 - 23 会在mmap上分配.
测试:
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>
#include <sys/mman.h>
#include <fcntl.h>

void sys_err(const char* str){
    perror(str);
    exit(1);
}

int var = 100;
int main(int argc, char* argv[])
{
    int *p = NULL;
    int len = 80;
    p = (int *) mmap(NULL, len, PROT_WRITE ,MAP_SHARED | MAP_ANON, -1,0);
    if(p == MAP_FAILED){
        sys_err("map fail");
    }
    void* pa = malloc(128*1024 - 24);
    void * pBrk = sbrk(0);
    printf("brk=%p, mmap=%p, malloc 599999=%p\n", pBrk, p, pa);




    return 0;
}
```

可以看到:
```
brk=0xa3a000, mmap=0x7fbc9282a000, malloc 599999=0x9f9010
```


```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>
#include <sys/mman.h>
#include <fcntl.h>

void sys_err(const char* str){
    perror(str);
    exit(1);
}

int var = 100;
int main(int argc, char* argv[])
{
    int *p = NULL;
    int len = 80;
    p = (int *) mmap(NULL, len, PROT_WRITE ,MAP_SHARED | MAP_ANON, -1,0);
    if(p == MAP_FAILED){
        sys_err("map fail");
    }
    void* pa = malloc(128*1024 - 23);
    void * pBrk = sbrk(0);
    printf("brk=%p, mmap=%p, malloc 599999=%p\n", pBrk, p, pa);




    return 0;
}
```
此时在mmap中分配
```
brk=0xcbc000, mmap=0x7f0f637e5000, malloc 599999=0x7f0f637ab010
```
猜测:
因为24的长度是因为malloc的时候如果内存空闲,需要用3个位置分别记录下一个指针,上一个指针以及当前长度.8*3
超过这个长度的话总体长度128*1024在堆上无法保证(因为头尾还要减去开销),所以超过这个长度需要mmap.
这里的128k指的是最终malloc分配的内存,不是用户感知的