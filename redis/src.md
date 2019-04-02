# SDS
字符串数据结构.  
通过结构体sdshdr sds handler 进行管理
```cpp
struct sdshdr {
    
    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 数据空间
    char buf[];
};
```
### sdslen
查看当前sds长度.  
因为每次操作的时候都会更新sdshdr.len,所以这里只是简单获取sdshdr的值.  
但是为了直接访问sds的内容,对外的sds指针是里面buf的指针.所以要先减去sdshdr大小,拿到sdshdr的头地址,然后访问.  
复杂度:O(1)
### sdsnewlen
根据给定的字符串长度,和内容,初始化字符串
注意:申请的内存大小是sdshdr和buf的大小,操作使用sdshdr操作,**返回的是buf的指针**.  
因此返回的内容可以直接当char*使用.  
### sdsempty
创建并返回一个空的sds  
直接是sdsnewlen("",0)
### sdsnew
给定字符串初始化sds,如果字符串有内容,用了strlen,复杂度:O(n)  
直接使用sdsnewlen创建
### sdsfree
释放sds,free O(n) //TODO为什么是n复杂度
### sdsclear
不释放sds内存,直接把buf第一个设置为\0,free=len,len=0.惰性删除.  
O(1)
### sdsMakeRoomFor
函数定义 sds sdsMakeRoomFor(sds s, size_t addlen) ,目标是对 sds 中 buf 的长度进行扩展，确保在函数执行之后buf 至少会有 addlen + 1 长度的空余空间.（额外的 1 字节是为 \0 准备的）

- s的free已经足够addlen大,直接返回s  
- s的free不够大,根据s的len加上入参addlen拿到新的长度newlen.  
    - newlen小于1024*1024,newlen *=2,预留长度防止后续继续扩张  
    - newlen已经>=1024*1024,加上一个 1024 *1024大小,防止后续扩张  
根据这个大小在入参s的sh地址zrealloc出sds头和buffer大小的空间  
注意:这里会**预先分配更多的空间**

### sdsRemoveFreeSpace
对sds的所有free空间会搜,使用zrealloc  
free设置为0.  
复杂度 O(N)
### sdsAllocSize
返回sds分配的字节数.直接返回sh len free +1的和.  
O(N)
### sdsgrowzero
让sds对设定的长度后面用0填充,用来扩充长度  
使用sdsMakeRoomFor扩充,memset塞0  
如果长度比原来的sds.len小,直接返回  
### sdscatlen
sdscatlen(sds s, const void *t, size_t len)  
把长度是len的t放在的后方.  
先对s使用sdsMakeRoomFor,扩展长度.(如果原来s足够大,不会扩展)  
把t通过memcpy放到sds的字符后方.(使用sdslen),在最后加上\0  
len和free进行调整.  
O(N)
### sdscat
sdscat(sds s, const char *t)  
把t放在s后.使用sdscatlen  
### sdscpylen
sdscpylen(sds s, const char *t, size_t len)  
把t放在s,buf的开始地址开始放,放len这么多.
如果s不够大,sdsMakeRoomFor  
然后memcpy
O(N)
### sdscpy
使用sdscpylen
### sdsll2str
ll转char*    
先把ll数据摆正  
逆序放入数组中,再进行一次调整,顺序回来.  
好处是不需要对负号进行再次调整.  
好处是不用每次都对指针进行定位.顺序的话指针需要多次移动,或者使用额外的空间存下来.  

```cpp
int sdsll2str(char *s, long long value) {
    char *p, aux;
    unsigned long long v;
    size_t l;

    /* Generate the string representation, this method produces
     * an reversed string. */
    v = (value < 0) ? -value : value;
    p = s;
    do {
        *p++ = '0'+(v%10);
        v /= 10;
    } while(v);
    if (value < 0) *p++ = '-';

    /* Compute length and add null term. */
    l = p-s;
    *p = '\0';

    /* Reverse the string. */
    p--;
    while(s < p) {
        aux = *s;
        *s = *p;
        *p = aux;
        s++;
        p--;
    }
    return l;
}
```
###  sdsull2str
没有负号转换的sdsll2str
### sdsfromlonglong
sdsll2str和sdsnewlen,返回sds

### sdscatvprintf
sds sdscatvprintf(sds s, const char *fmt, va_list ap)  
打印函数,使用静态数组提高效率,使用vsnprintf打印到指定buff,然后sdscat到s

### sdscatprintf
使用sdscatvprintf把字符串打印到s

### sdscatfmt
类似sdscatprintf,但是比较快,因为不使用sprintf,而是直接对sds的buff操作.
直接sdsMakeRoomFor之后memcpy,如果是数值型先sdsll2str等处理.

### sdstrim
sds sdstrim(sds s, const char *cset)   
修剪  
```cpp
* Example:
 *
 * s = sdsnew("AA...AA.a.aa.aHelloWorld     :::");
 * s = sdstrim(s,"A. :");
 * printf("%s\n", s);
 *
 * Output will be just "Hello World".

 ```
 核心
 如果当前指针的字符在cset集合中,跳过
 ```cpp
     // 修剪, T = O(N^2)
    while(sp <= end && strchr(cset, *sp)) sp++;
    while(ep > start && strchr(cset, *ep)) ep--;
```
如果当前字符不在起始位置,memmove.
然后更新属性
```cpp
    sh->free = sh->free+(sh->len-len);
    sh->len = len;
```
这里不会对内存做处理

### sdsrange
对buff内容做修改,使用memmove  
最后更新属性len free,不对内存做处理

### sdstolower
使用tolower
O(N)

### sdstoupper
使用toupper
O(N)

### sdscmp
使用memcmp

### sdssplitlen
sds *sdssplitlen(const char *s, int len, const char *sep, int seplen, int *count)  
返回数组
memcmp用来匹配断裂符号
使用sds数组指向结果,sdsnewlen新建结果存放
### sdsfreesplitres
释放sds数组,sdsfree每个sds,zfree数组
### sdscatrepr
sdscatrepr(sds s, const char *p, size_t len)  
在sds后追加长度为len的p字符串,如果p开始是特殊符号会在p结尾加上同样的特殊符号

### is_hex_digit
是否16进制.0-9,A-F,a-f

### hex_digit_to_int
16进制转10进制

### sdssplitargs
文本翻译成REPL格式
返回sds数组

### sdsmapchars
对固定长度的from 和to 字符替换



### sdsjoin
根据main的入参argc argv依次把字字符串连接

# adlist 双向链表
双端链表结构
```cpp
typedef struct list {

    // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);

    // 链表所包含的节点数量
    unsigned long len;

} list;
```
链表节点结构
```cpp
typedef struct listNode {

    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value;

} listNode;
```

迭代器结构
```cpp
typedef struct listIter {

    // 当前迭代到的节点
    listNode *next;

    // 迭代的方向
    int direction;

} listIter;

```

### list *listCreate(void)
zmalloc分配list内存创建链表.此时没有listNode  

### void listRelease(list *list)
遍历list的listNode zfree释放,最后zfree list

### list *listAddNodeHead(list *list, void *value)
zmalloc一个listNode空间,把value放入链表中.(链表中的value是个指针,需要注意)  
新节点放在头,list.len++

### list *listAddNodeTail(list *list, void *value)
类似上述,只是放在尾

### list *listInsertNode(list *list, listNode *old_node, void *value, int after) 
根据after看放在old_node的前面还是后面.  
zmalloc一个listNOde  
看情况改头指针  
list.len++

### void listDelNode(list *list, listNode *node)
删除一个节点.链表操作完成之后zfree node.  
list.len--  

### listIter *listGetIterator(list *list, int direction)
获取一个迭代器  
zmalloc申请listIter空间  
如果方向是AL_START_HEAD(0),从头开始,listIter.next是list.head  
如果方向不是,从尾开始,listIter.next是list.tail  
listIter.direction记录方向

### listReleaseIterator
zfree

### void listRewind(list *list, listIter *li) 
迭代器初始化,指向头 往尾

### void listRewindTail(list *list, listIter *li)
迭代器初始化,指向尾 往头

### listNode *listNext(listIter *iter)
返回迭代器的next指向的listNode,同时迭代器++(next=self.next)

### list *listDup(list *orig)
复制一个链表  
使用iterator,每个链表遍历,申请list和listNode   
如果listNode.dup有设置,对value进行复制.  
否则不复制,新旧list指向同一个value

### 


