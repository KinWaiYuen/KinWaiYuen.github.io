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

### listNode *listSearchKey(list *list, void *key)
使用iterator,如果有定义match方法,就使用match对比ite的value和key.没有就直接对比listnode.value是否等于key  
返回结果前free ite

### listNode *listIndex(list *list, long index) 
如果index负数,从尾部开始;否则从头开始  
找第index个listnode

### void listRotate(list *list)
表尾节点放到表头

# DICT 字典
哈希表结构定义
```cpp
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    // hashing的时候是对应正在hash的值
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

    // 目前正在运行的安全迭代器的数量
    int iterators; /* number of iterators currently running */

} dict;
```
其中dictType用于定义哈希节点的具体函数  
```cpp
typedef struct dictType {

    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);

    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);

    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);

    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);

    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);

} dictType;
```
dict.ht[2]  
里面有哈希表数组,用于resize  
dictht定义  
```cpp
typedef struct dictht {
    
    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;
    
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;

} dictht;
```
size是哈希的桶数.sizemask用来计算index  

哈希节点
```cpp
typedef struct dictEntry {
    
    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;

} dictEntry;
```
看到里面是有指向next的,就是说如果冲突的话使用链地址法,冲突的往下一个添加.

dict迭代器  
用于迭代访问字典的每一项.
```cpp
typedef struct dictIterator {
        
    // 被迭代的字典
    dict *d;

    // table ：正在被迭代的哈希表号码，值可以是 0 或 1 。
    // index ：迭代器当前所指向的哈希表索引位置。
    // safe ：标识这个迭代器是否安全
    int table, index, safe;

    // entry ：当前迭代到的节点的指针
    // nextEntry ：当前迭代节点的下一个节点
    //             因为在安全迭代器运作时， entry 所指向的节点可能会被修改，
    //             所以需要一个额外的指针来保存下一节点的位置，
    //             从而防止指针丢失
    dictEntry *entry, *nextEntry;

    long long fingerprint; /* unsafe iterator fingerprint for misuse detection */
} dictIterator;
```

### dict *dictCreate(dictType *type,void *privDataPtr)
zmalloc出dict,对dict初始化_dictInit
### int _dictInit(dict *d, dictType *type,void *privDataPtr)
对dict各个元素初始化,其中privdata设置成privDataPtr,rehashidx设置成-1  
使用dictReset初始化ht  
_dictReset对ht的元素初始化,设置为0

### int dictResize(dict *d)
如果dict_can_resize是假,或者dictIsRehashing为真,不能resize.  
#### dict_can_resize
是全局变量,要么所有的哈希表都能resize,要么都不能resize.通过static全局控制.
#### dictIsRehashing
看rehashidx是否-1.如果-1表示当前没有在rehashing.不是的话表示正在rehash到第几个.

获取ht[0]的used,如果小于4,resize的时候至少是4.  
调用dictExpand 调整大小

### int dictExpand(dict *d, unsigned long size)
实际分配的空间会是size的两倍,但是size要小于  0x7fffffffffffffffL.  
如果d正在哈希,或者d的ht[0].used比size要大,报错  
根据计算的实际分配空间个数realsize,zcalloc(realsize * sizeof(dictEntry*)).把dictEntry使用数组存起来,就是哈希表.  
如果ht[0].table为空,这次初始化给ht[0].  
否则给ht[1],并且rehashidx=0,表示当前正在rehash.用于后续rehash
```ditaa
┌──────────────┐            ┌──────────────┐
│  dictEntry   │───────────▶│  dictEntry   │
├──────────────┤            └──────────────┘
│  dictEntry   │                            
├──────────────┤                            
│  dictEntry   │                            
├──────────────┤                            
│  dictEntry   │                            
├──────────────┤                            
│  dictEntry   │                            
├──────────────┤                            
│  dictEntry   │                            
├──────────────┤                            
│     ...      │                            
├──────────────┤                            
│  dictEntry   │                            
└──────────────┘                            
```


### int dictRehash(dict *d, int n) 
对d进行n步的rehash  
如果ht[0].used已经为0,就是ht[0]已经被处理干净.
- zfree(ht[0].table)
- d->ht[0] = d->ht[1]
- 重置ht[1]
- d->rehashidx = -1 表示rehash完成
- 返回0

在ht[0]根据d.rehashidx找非空的dictEntry,如果下一个idx是空,d.rehashidx++,记录当前找到的非空dictEntry在ht[0]中的位置.  
找到对应的ht[0][rehashid]入口之后  
- 计算在ht[1]中的哈希值
- 放入ht[1]对应位置
- ht[0].used--, ht[1].used++
- 如果当前ht[0][rehashid]还有其他元素,重复上述步骤.否则跳出往下
- ht[0][rehashid]置空 当前数组over
- d.rehashidx++ 表示rehash继续进发

### long long timeInMilliseconds(void)
获取当前毫秒时间戳

### int dictRehashMilliseconds(dict *d, int ms) 
以毫秒为单位,步长100,使用dictRehash进行哈希.如果超时就退出,返回已经哈希的次数.  
使用timeInMilliseconds控制时长


### static void _dictRehashStep(dict *d)
如果字典没有迭代器,对字典单步reahash.使用dictRehash(d,1)




### static int _dictExpandIfNeeded(dict *d)
根据需要初始化哈希表或者对字典哈希表进行扩展  
- 如果已经在rehashing,返回ok
- 如果ht[0].size是0,返回dictExpand(d, DICT_HT_INITIAL_SIZE),给ht[0]分配空间
- 如果ht[0].used >= ht[0].size,且当前全局变量允许扩展或者负载因子大于5,扩展成2倍dictExpand(d, d->ht[0].used*2)

注意:
- 如果ht[0]size是0,默认分配空间是4,实际分配是8
- 如果ht[0].used已经大于ht[0].size,也就是当前这个字典负载因子>1,需要扩size.如果全局变量dict_can_resize允许,就马上扩;但是如果全局变量dict_can_resize但是负载因子>5,基于字典可用性降低,会强制扩.除了这里其他地方都依据dict_can_resize来扩.


### dictCompareKeys(d, key1, key2) 
使用keyCompare比较key1,key2

### static int _dictKeyIndex(dict *d, const void *key)
返回可以将 key 插入到哈希表的索引位置  
如果key已经在d中,返回-1  
- 先执行_dictExpandIfNeeded,先根据需求进行扩容.
- 使用dictHashKey(使用h中的哈希方法)对key转化为数值.使用hashFunction计算key的值
- 从ht[0]开始看,计算哈希的值idx,看ht[i].table[idx]是否存在.
		- 如果存在,使用dictCompareKeys,看key是否和哈希表里的key一样.一样返回-1,不一样就next,
		- 如果next到空,说明找到了对应的位置(要么第一个,要么在链表尾部)
		- 如果这时候没有hashing,退出.否则说明ht[0]没有并没有key,但是现在rehashing,继续对ht[1]查找idx,以ht[1]为准.因为后续插入在ht[1]中.**这里应该有优化空间,直接根据rehashing状态判断是否查ht[1]即可**



### dictEntry *dictAddRaw(dict *d, void *key)
尝试将key放入d中,如果可以放,直接新建一个空的dictEntry,并返回.
- 如果当前d在rehashing,进行_dictRehashStep
- 计算key在d中的索引值,如果key已经存在,gg
- 如果正在rehashing,写入ht[1].否则写入ht[0]
- zmalloc分配dictEntry空间,加入对应的ht index中
- ht.used++
- dictSetKey(d, entry, key)设置新节点的键
- 返回dictEntry


### dictSetKey(d, entry, key)
如果有自定义keyDup,使用d中的keyDup进行key设定.  
否则直接把key赋值给entry.key

### dictSetVal(d, entry, _val_) 
有定义d的valDup就使用定义函数,没有的话直接val赋值给entry.val

### int dictAdd(dict *d, void *key, void *val)
往字典添加一个kv  
先使用dictAddRaw,添加给d一个空的dictentry,然后再用dictSetVal把value赋值过去.

### int dictReplace(dict *d, void *key, void *val)
给定的kv添加到字典.如果字典已经有,就替换value
- 先直接dictAdd,如果成功直接返回
- dictAdd失败,dictFind查找d中的key 
		- 先保存找到的entry(是保存dictEntry的内容)
		- 对原来的entry进行dictSetVal,把新的val到原来的entry
		- 把第一步中的entry内容进行dictFreeVal,释放旧的value.

### dictEntry *dictReplaceRaw(dict *d, void *key) 
如果节点存在,直接返回.   
如果不存在,直接dictAddRaw一个空的entry,返回  

### static int dictGenericDelete(dict *d, const void *key, int nofree)
查找删除key的节点.nofree表示是否释放entry  
**如果正在rehashing,会进行_dictRehashStep**
查找范围是**ht[0]到ht[1]**.如果没有rehashing,直接查ht[0],否则还会继续查ht[1]
如果设置了nofree是0,会dictFreeKey和dictFreeVal
最后zfree整个entry,对应的th[i].used--

### int dictDelete(dict *ht, const void *key) 
dictGenericDelete nofree=0

### int dictDeleteNoFree(dict *ht, const void *key)
nofree = 1

### int _dictClear(dict *d, dictht *ht, void(callback)(void *)) 
遍历整个哈希表,对每个entry的key value都释放,以及entry,ht,最后_dictReset ht  
最后目的是初始化了ht 


### void dictRelease(dict *d)
对d的两个ht进行_dictClear



### dictEntry *dictFind(dict *d, const void *key)
如果没有rehashing,就在ht[0]进行idx查找;否则在ht[1]也会查找.  
计算key,求出索引值idx  
注意:idx计算  
```cpp
idx = h & d->ht[table].sizemask
```
这里是使用位&,并不是求余数,是因为sizemask+1总是2^n,所以sizemask的二进制是000...000111...111,所以使用位&就能满足idx的获取.

### void *dictFetchValue(dict *d, const void *key)
dictFind + dictGetVal

### long long dictFingerprint(dict *d)
根据dict的信息(ht只有指针信息)计算指纹信息,应该用于验证


### dictIterator *dictGetIterator(dict *d)
zmalloc dictIterator, 初始化.这里:
- safe=0,
- table是0,默认0的ht.
- entry是NULL,
- index是-1

### dictIterator *dictGetSafeIterator(dict *d)
dictGetIterator + 设置safe=1


### dictEntry *dictNext(dictIterator *iter)
使用迭代器查找  
如果是安全迭代器,d的iterators++,表明当前有多少个安全迭代器在运行.否则就记录下d的指纹  
- 如果iter.index已经>=ht.size,说明已经到了当前ht的尾,要看当前是否rehashing.如果是,跳到ht[1],继续查找;否则报错  
- 如果当前entry位空,表示当前entry是在哈希列表中的某个空节点,或者是某个节点的尾节点.此时index++  
- 如果当前entry不是空,下一个节点就是nextEntry,可能是冲突导致,也可能当前哈希列表只有一个节点.

### void dictReleaseIterator(dictIterator *iter)
释放迭代器.  
如果之前迭代器已经在某个哈希表中使用过:
- 安全的迭代器释放时候d的iterator--
- 不安全的迭代器需要确认指纹是否一致,不一致的话exit(1)

zfree迭代器

### dictEntry *dictGetRandomKey(dict *d)
返回随机节点  
如果size是0,返空  
如果rehashing,**_dictRehashStep(d)**  
返回非空节点  
先找到一个非空的ht[i][idx],再在这个链表中找一个非空的entry返回

### int dictGetRandomKeys(dict *d, dictEntry **des, int count) 
获取随机节点数组

### static unsigned long rev(unsigned long v) 
翻转bits
```cpp
static unsigned long rev(unsigned long v) {
    unsigned long s = 8 * sizeof(v); // bit size; must be power of 2
    unsigned long mask = ~0;
    while ((s >>= 1) > 0) {
        mask ^= (mask << s);
        v = ((v >> s) & mask) | ((v << s) & ~mask);
    }
    return v;
}
```

### unsigned long dictScan(dict *d,unsigned long v,dictScanFunction *fn,void *privdata)
扫描d,fn是扫描到之后的处理动作.   
scan的时候回吧ht[i].table[j]上所有的dictEntry都会执行一次fn(也就是所有冲突的都会处理一次)
使用游标的方式,每次传入一个位置,查找完会返回下一个位置.  
游标的返回比较巧妙.

#### 字典scan顺序
如果当前ht只有一个(不是rehashing),fn处理当前的table[j]后返回下一个游标  
如果当前ht不止一个(正在rehashing):
- 先确保第一个处理的是size小的ht,否则互换
- size小的ht先遍历当前table[j]
- 大size的ht分别遍历j以及比小ht的mask高一位的位置置1的table[j]

此处j的变化
```cpp
        do {
            /* Emit entries at cursor */
            // 指向桶，并迭代桶中的所有节点
            de = t1->table[v & m1];
            while (de) {
                fn(privdata, de);
                de = de->next;
            }

            /* Increment bits not covered by the smaller mask */
            //这里是v在m0的前一位+1
            //ex:v 11010110 m0 1111
            //结果 11100110
            v = (((v | m0) + 1) & ~m0) | (v & m0);

            /* Continue while bits covered by mask difference is non-zero */
        } while (v & (m0 ^ m1));//比m0高出一位的.事实上就是m0 m1两个都走一次.上面的
```
关键部分
```cpp
v = (((v | m0) + 1) & ~m0) | (v & m0);
```
((v | m0) + 1)表示v在m0的前一个位置+1,然后超过m0的位置不变,和尾部拼接.  
(v & (m0 ^ m1))正常情况下v不应该大于2*mask,所以m0^m1表示m1最高位是1,其他都是0.经过两个循环之后v的这个位会恢复.所以退出.  

**游标变化**
```cpp
   v |= ~m0;//留尾部,前面都是1

    /* Increment the reverse cursor */
    v = rev(v);//二进制倒转
    v++;//自增
    v = rev(v);//二进制再返回
    //上述的处理完成,比如传入是0 m0是00001111
    //v先变成11110000,然后翻转 00001111 自增后  00010000再翻转00001000
    //所以下次遍历的是00001000,
    //再下次是00000100
    //再下次是00001100
    //如果当时的ht表更改了,扩容或者缩容,因为是2的倍数,
    //例如扩容,下一次遍历的位置是00011100,之前扩容搜索的位置可以确保不再被搜索

    return v;
```
举例:v开始是00001000,经过过程  11111000 00011111 00100000 00000100,下次遍历的位置是00000100
因为在遍历的时候有可能m0会变化.这里使用游标的好处:  
下面举例:原来mask是00001111.  
- 扩容时,例如上一次遍历位置在00001100,扩容后mask是00011111,经过过程后是00011100,下次只要直接访问00011100,再下次访问的idx是00000010(这和在没有扩容的00001100访问的下一个一样).因为在00001100前面的都已经遍历过,而这些遍历过的dictEntry扩容之后的下标是000x<原来下标>,这些之前都已经遍历过,所以不需要遍历.
- 缩容是,例如上次遍历位置是00001000,缩容后mask是00000111,经过上述过程转换后是00000100.因为缩容前的0000x<下标后缀>的下标缩容后变成00000<下标后缀>.这种情况下下次访问00000100实际上是访问00001100和00000100.00000100之前的(00000000,00001000)都已经被访问过.

因此使用这种方式进行遍历对扩容/缩容不会造成影响,不怕访问漏.  
但是可能多访问:  
扩容情况,原来mask00001111,访问了00000000后,mask变成00011111.此时下个遍历的是00010000.
但是在扩容前的00000000是包括了扩容后的00000000和00010000,所以这里的00010000会被重复访问.

```ditaa
         ┌────┐                  ┌────┐                     ┌────┐                       ┌────┐         
         │ 00 │                  │ 10 │                     │ 01 │                       │ 11 │         
         └────┘                  └────┘                     └────┘                       └────┘         
            │                       │                          │                            │           
      ┌─────┴────┐             ┌────┴─────┐             ┌──────┴────┐                ┌──────┴─────┐     
      │          │             │          │             │           │                │            │     
      ▼          ▼             ▼          ▼             ▼           ▼                ▼            ▼     
   ┌────┐     ┌────┐        ┌────┐     ┌────┐        ┌────┐      ┌────┐           ┌────┐       ┌────┐   
   │000 │     │100 │        │010 │     │110 │        │001 │      │101 │           │011 │       │111 │   
   └────┘     └────┘        └────┘     └────┘        └────┘      └────┘           └────┘       └────┘   
      ▲          ▲             ▲          ▲             ▲           ▲                ▲            ▲     
   ┌──┴──┐    ┌──┴──┐       ┌──┴──┐    ┌──┴──┐       ┌──┴──┐     ┌──┴──┐          ┌──┴──┐      ┌──┴──┐  
   │     │    │     │       │     │    │     │       │     │     │     │          │     │      │     │  
┌────┐┌────┬────┐┌────┐  ┌────┐┌────┬────┐┌────┐  ┌────┐┌────┐┌────┐┌────┐     ┌────┐┌────┐ ┌────┐┌────┐
│0000││1000│0100││1100│  │0010││1010│0110││1110│  │0001││1001││0101││1101│     │0011││1011│ │0111││1111│
└────┘└────┴────┘└────┘  └────┘└────┴────┘└────┘  └────┘└────┘└────┘└────┘     └────┘└────┘ └────┘└────┘
```


### dict有_dictRehashStep的操作
**重点注意**
所谓的渐进式rehash.增删改查都会有触发到
- dictAddRaw
- dictGenericDelete
- dictFind
- dictGetRandomKey
- dictAdd
- dictReplace
- dictReplaceRaw

### 迭代器和scan区别

|特点|迭代器|scan|
|---|---|---|
|单位|entry|table i|
|遍历顺序|table++,nextEntry|table编号后缀二进制倒叙|
|能不能保证完全遍历|不能.比如缩容|能|

# ziplist 
跳表结构  
```cpp
typedef struct zskiplist {

    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;

} zskiplist;
```
头尾节点,数量,最大层数

跳表节点
```cpp
typedef struct zskiplistNode {

    // 成员对象
    robj *obj;

    // 分值
    double score;

    // 后退指针
    struct zskiplistNode *backward;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 跨度
        unsigned int span;

    } level[];

} zskiplistNode;
```
成员,分值,这个节点的后退指.层包括层的前进指针以及这个前进指针对应的跨度. 
注意:层是结构体数组.本质是一个指向这个结构体的指针.  
头结点不记录数据,但是有每一层的层高,span,方便查找.

## zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) 

跳表插入.根据分数插入,返回新节点的地址.  
ZSKIPLIST_MAXLEVEL是跳表级数的最大上限,32.
插入之前先找到插入的路径.  
- 新建unsigned int rank[ZSKIPLIST_MAXLEVEL],用来记录跨度
- zskiplistNode * update[ZSKIPLIST_MAXLEVEL], * x,用来记录当前更新到哪个节点(就是搜索路经)
- 从头节点开始往下搜,头结点会记录最大的level.而rank[i]从zsl.level-1开始往下搜.最顶开始初始化是0,后续的初始化是上一个,就是rank[i+1]
		- 如果当前节点的level[i]有forward,而且需要插入的score比节点的score要大,或者有forward且插入的obj比节点的obj要大,rank[i] += x->level[i].span;x = x->level[i].forward;
        - 上述流程继续找,直到找到比要插入节点的要大的
        - 此时终止在刚好比自己小的节点上.这里rank[i]记录了从头节点到现在移动的跨度.记录下当时的x.update[i] = x
经过上面的跳转,可以理解成:从最高的level的节点开始找,找到刚好比自己大的,记录下这个跳表节点.如果这个节点已经没有forward,跳出来找下一级.  
这时候拿到了rank[],表示搜索的时候在当前的level跳转了多少距离到下一个符合条件的跳表节点.  
update[]表示的是每次每一级应该在哪个节点插入.
```cpp
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    redisAssert(!isnan(score));

    // 在各个层查找节点的插入位置
    // T_wrost = O(N^2), T_avg = O(N log N)
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {

        /* store rank that is crossed to reach the insert position */
        // 如果 i 不是 zsl->level-1 层
        // 那么 i 层的起始 rank 值为 i+1 层的 rank 值
        // 各个层的 rank 值一层层累积
        // 最终 rank[0] 的值加一就是新节点的前置节点的排位
        // rank[0] 会在后面成为计算 span 值和 rank 值的基础
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];

        // 沿着前进指针遍历跳跃表
        // T_wrost = O(N^2), T_avg = O(N log N)
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                // 比对分值
                (x->level[i].forward->score == score &&
                // 比对成员， T = O(N)
                compareStringObjects(x->level[i].forward->obj,obj) < 0))
                ) {

            // 记录沿途跨越了多少个节点
            rank[i] += x->level[i].span;

            // 移动至下一指针
            x = x->level[i].forward;
        }
        // 记录将要和新节点相连接的节点
        update[i] = x;
    }
```
经过上面的跳转,可以拿到一条**搜索路径**.
![](../imgs/redis/skiplist_search.png)
概括来说就是从上到下开始,每次找到当前level适合插入的节点,然后降级,从上次的节点开始继续向下找.同时记录路径.  

下一步是通过连续枚举随机数,这个随机数用来确定当前这个跳表节点的level
```cpp
int zslRandomLevel(void) {
    int level = 1;

    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;

    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```
ZSKIPLIST_P是0.25,意味着在大概率里高一级的个数是低一级的1/4.  
节点的级数是通过概率来确定的,也就是会有极小的情况会拿到一个很高的level(当枚举足够大的时候才出现,是合理的.因为这时候已经有足够多的小level节点)  

如果高于当前zsl的level,高于level的需要进行初始化
```cpp
    if (level > zsl->level) {

        // 初始化未使用层
        // T = O(1)
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }

        // 更新表中节点最大层数
        zsl->level = level;
    }
```
可以理解是:update[]这里高于level的都相当于从头节点开始找,这个level的span(就是到即将插入的节点)就是zsl的length(因为这个节点一定会放到最后面)

有了level和搜索路径,可以创建节点,然后把节点插入到跳表中.  
因为之前的update[i]就是刚好比即将插入节点要小的节点,所以节点的level[i]就是update[i]的level[i]的forward,就是刚好比自己小的节点的后驱  
update[i]的forward就是x(因为x插在后面)  
x的跨度update[i]->level[i].span - (rank[0] - rank[i]),因为已经比update[i]往后了(rank[0] - rank[i])这里rank[0]可以理解就是插入的排序.  
update[i].span也需要更改,因为已经不是之前的span,而是比之前span更往前.rank[i]就是当前update[i]的位置,所以直接是(rank[0] - rank[i]) + 1
```cpp
    for (i = 0; i < level; i++) {
        
        // 设置新节点的 forward 指针
        x->level[i].forward = update[i]->level[i].forward;
        
        // 将沿途记录的各个节点的 forward 指针指向新节点
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here */
        // 计算新节点跨越的节点数量
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);

        // 更新新节点插入之后，沿途节点的 span 值
        // 其中的 +1 计算的是新节点
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }
```
如果插入的节点level比原来的要高,span++(因为之前还没插入,现在节点插入了)    
最后设置x的backward指针,就是update[0],以及zsl的长度++  

插入前
```ditaa
┌─────────────┐                                                                                                                                                           
│     zsl     │          ┌──────────────┐                                                                                                                                 
├─────────────┤          │     ...      │                                                                                                                                 
│   header    │          ├──────────────┤                                                                                                                                 
│             │─────────▶│      L4      │                                                                                                                                 
├─────────────┤          ├──────────────┤                                                                                                                                 
│    tail     │          │      L3      │                                                                                                                                 
│             │          ├──────────────┤                                                                                                                                 
├─────────────┤          │      L2      │                                                                                                                                 
│    level    │          ├──────────────┤                                                                      ┌──────────────┐                                           
│             │          │      L1      │──────────────────────────────────────────────2──────────────────────▶│      L1      │                                           
├─────────────┤          ├──────────────┤                 ┌──────────────┐            ┌──────────────┐         ├──────────────┤          ┌──────────────┐                 
│   length    │          │      L0      │─────1──────────▶│      L0      │────1──────▶│      L0      │───1────▶│      L0      │───1─────▶│      L0      │────────────────▶
│             │          └──────────────┘                 ├──────────────┤            ├──────────────┤         ├──────────────┤          ├──────────────┤                 
├─────────────┤                                     ┌─────│   backward   │            │   backward   │         │   backward   │          │   backward   │                 
│             │                                     │     ├──────────────┤            ├──────────────┤         ├──────────────┤          ├──────────────┤                 
│             │                                     │     │    score     │            │    score     │         │    score     │          │    score     │                 
│             │                                ◀────┘     ├──────────────┤            ├──────────────┤         ├──────────────┤          ├──────────────┤                 
│             │                                           │     obj      │            │     obj      │         │     obj      │          │     obj      │                 
│             │                                           └──────────────┘            └──────────────┘         └──────────────┘          └──────────────┘                 
└─────────────┘                                                                                                                                                                                                                                                                                                                   
```

找到路径
```ditaa
┌─────────────┐                                                                                                                                                                                    
│     zsl     │          ┌──────────────┐                                                                                                                                                          
├─────────────┤          │     ...      │                                                                                                                                                          
│   header    │          ├──────────────┤                                                                                                                                                          
│             │─────────▶│      L4      │                                                                                                                                                          
├─────────────┤          ├──────────────┤                                                                                                                                                          
│    tail     │          │      L3      │                                                                                                                                                          
│             │          ├──────────────┤                                                                                                                                                          
├─────────────┤          │      L2      │                                                                                                                                                          
│    level    │          ├──────────────┤                                                                      ┌──────────────┐                                                                    
│             │          │      L1      │──────────────────────────────────────────────2──────────────────────▶│      L1      │───────────────▶                                                    
├─────────────┤          ├──────────────┤                 ┌──────────────┐            ┌──────────────┐         ├──────────────┤               ┌──────────────┐                                     
│   length    │          │      L0      │─────1──────────▶│      L0      │────1──────▶│      L0      │───1────▶│      L0      │─────1────────▶│      L0      │────────────────────────────▶        
│             │          └──────────────┘                 ├──────────────┤            ├──────────────┤         ├──────────────┤               ├──────────────┤                                     
├─────────────┤                                     ┌─────│   backward   │            │   backward   │         │   backward   │               │   backward   │                                     
│             │                                     │     ├──────────────┤            ├──────────────┤         ├──────────────┤               ├──────────────┤                                     
│             │                                     │     │    score     │            │    score     │         │    score     │               │    score     │                                     
│             │                                ◀────┘     ├──────────────┤            ├──────────────┤         ├──────────────┤               ├──────────────┤                                     
│             │                                           │     obj      │            │     obj      │         │     obj      │               │     obj      │                                     
│             │                                           └──────────────┘            └──────────────┘         └──────────────┘               └──────────────┘                                     
└─────────────┘                                                                                                        ▲                              ▲                                            
                                                                                                                       │                              │                            ┌──────────────┐
                                                                                                                       │                              │                            │   new node   │
                                                                                                                       │                              │                            ├──────────────┤
                                                                                                                       │                              │                            │      L0      │
                                                                                                                       │                              │                            ├──────────────┤
                                                                                                                       │                              │                            │   backward   │
                                                                                                                       │                              │                            ├──────────────┤
                                                                                                                       │                              │                            │    score     │
                                                                                                                       │                              │                            ├──────────────┤
                                                                                                                       │         ┌────────┐           │         ┌────────┐         │     obj      │
                                                                                                                       │         │ update │           │         │  rank  │         └──────────────┘
                                                                                                                       │         ├────────┤           │         ├────────┤                         
                                                                                                                       └─────────│   1    │           │         │   3    │                         
                                                                                                                                 ├────────┤           │         ├────────┤                         
                                                                                                                                 │   0    │───────────┘         │   4    │                         
                                                                                                                                 └────────┘                     └────────┘                         
                                                                                                                                
```

插入后
```ditaa
┌─────────────┐                                                                                                                                                                                          
│     zsl     │          ┌──────────────┐                                                                                                                                                                
├─────────────┤          │     ...      │                                                                                                                                                                
│   header    │          ├──────────────┤                                                                                                                                                                
│             │─────────▶│      L4      │                                                                                                                                                                
├─────────────┤          ├──────────────┤                                                                                                                                                                
│    tail     │          │      L3      │                                                                                                                                                                
│             │          ├──────────────┤                                                                                                                                                                
├─────────────┤          │      L2      │                                                                                                                                                                
│    level    │          ├──────────────┤                                                                      ┌──────────────┐                                               ┌──────────────┐           
│             │          │      L1      │──────────────────────────────────────────────2──────────────────────▶│      L1      │───────────────▶                               │   new node   │           
├─────────────┤          ├──────────────┤                 ┌──────────────┐            ┌──────────────┐         ├──────────────┤               ┌──────────────┐                ├──────────────┤           
│   length    │          │      L0      │─────1──────────▶│      L0      │────1──────▶│      L0      │───1────▶│      L0      │─────1────────▶│      L0      │───────1───────▶│      L0      │──────────▶
│             │          └──────────────┘                 ├──────────────┤            ├──────────────┤         ├──────────────┤               ├──────────────┤                ├──────────────┤           
├─────────────┤                                     ┌─────│   backward   │◀───────────│   backward   │◀────────│   backward   │◀──────────────│   backward   │◀───────────────│   backward   │           
│             │                                     │     ├──────────────┤            ├──────────────┤         ├──────────────┤               ├──────────────┤                ├──────────────┤           
│             │                                     │     │    score     │            │    score     │         │    score     │               │    score     │                │    score     │           
│             │                                ◀────┘     ├──────────────┤            ├──────────────┤         ├──────────────┤               ├──────────────┤                ├──────────────┤           
│             │                                           │     obj      │            │     obj      │         │     obj      │               │     obj      │                │     obj      │           
│             │                                           └──────────────┘            └──────────────┘         └──────────────┘               └──────────────┘                └──────────────┘           
└─────────────┘                                                                                                        ▲                              ▲                                                  
                                                                                                                       │                              │                                                  
                                                                                                                       │                              │                                                  
                                                                                                                       │                              │                                                  
                                                                                                                       │                              │                                                  
                                                                                                                       │                              │                                                  
                                                                                                                       │                              │                                                  
                                                                                                                       │                              │                                                  
                                                                                                                       │                              │                                                  
                                                                                                                       │                              │                                                  
                                                                                                                       │         ┌────────┐           │         ┌────────┐                               
                                                                                                                       │         │ update │           │         │  rank  │                               
                                                                                                                       │         ├────────┤           │         ├────────┤                               
                                                                                                                       └─────────│   1    │           │         │   3    │                               
                                                                                                                                 ├────────┤           │         ├────────┤                               
                                                                                                                                 │   0    │───────────┘         │   4    │                               
                                                                                                                                 └────────┘                     └────────┘                               


```

### void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update)
和插入类似,使用update数组记录搜索路径,对每一个刚好比x小的节点都释放掉x(指针不指向x,而是指向x.forward.而且span+=x.level[i].span - 1,补全了和删除后的forward的距离)

其次:level[0].forward更改,forward的backward更改,以及zsl的length --

如果这个节点是最高的节点,zsl的level调整.  
如果这个节点是尾部,zsl.tail调整.

# intset
```cpp
typedef struct intset {
    
    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;

```
搜索:二分查找(因为底层有序)

### 插入
intset在插入的时候会判断新插入的元素判断是否升级格式,如果是,需要升级insert格式.  
每次都需要insert resize,因为inset长度需要改变.
需要调整的位置使用把整块内存往后移动  

如果需要调整空间(要么插入的数比所有的数都大,要么是负数)
- 申请足够的空间 zrealloc
- _intsetSet让每个元素放到适合的地方(避开新插入的元素)
- 插入新元素

如果当前的编码是适合的,直接插入
- 找出插入元素的位置,使用二分法
- memmove符合位置后面的数据
- 元素放入合适的位置

### 删除
找出对应删除的位置,后续数组memmove到要删除的位置(覆盖原来数据),然后zrealloc(size-1)


# ziplist 压缩列表
结构
```
<zlbytes><zltail><zllen><entry><entry><zlend>
```
- <zlbytes> 是一个无符号整数，保存着 ziplist 使用的内存数量。为了不需要遍历整个ziplist而记录.
- <zltail> 保存着到达列表中最后一个节点的偏移量。 这样使得pop不需要遍历都可以实现.O(1)
- <zllen> 保存着列表中的节点数量。 但是当zllen > 2**16-2 要遍历才知道实际多少节点.
- <zlend> 的长度为 1 字节，值为 255 ，标识列表的末尾
- <entry> 格式是<header><body> <header>记录两个信息:
    - 前置节点的长度,用于向前遍历.
        - 如果前置节点长度 < 254,使用一个字节保存(2^8).
        - 否则使用5个字节保存:第一个字节是254表示是一个5字节长度的长度值.后面4个字节保存实际长度.
    - 当前节点的类型和长度.不同类型会有不同的格式
        - 字符串类型,如果长度足够小,信息直接放在头中
        - 整形都放在body中
        - 如果是11111111,表示是zpilist的结尾.

```
 * 1) 如果节点保存的是字符串值，
 *    那么这部分 header 的头 2 个位将保存编码字符串长度所使用的类型，
 *    而之后跟着的内容则是字符串的实际长度。
 *
 * |00pppppp| - 1 byte
 *      String value with length less than or equal to 63 bytes (6 bits).
 *      字符串的长度小于或等于 63 字节。
 * |01pppppp|qqqqqqqq| - 2 bytes
 *      String value with length less than or equal to 16383 bytes (14 bits).
 *      字符串的长度小于或等于 16383 字节。
 * |10______|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt| - 5 bytes
 *      String value with length greater than or equal to 16384 bytes.
 *      字符串的长度大于或等于 16384 字节。
 *
 * 2) 如果节点保存的是整数值，
 *    那么这部分 header 的头 2 位都将被设置为 1 ，
 *    而之后跟着的 2 位则用于标识节点所保存的整数的类型。
 *
 * |11000000| - 1 byte
 *      Integer encoded as int16_t (2 bytes).
 *      节点的值为 int16_t 类型的整数，长度为 2 字节。
 * |11010000| - 1 byte
 *      Integer encoded as int32_t (4 bytes).
 *      节点的值为 int32_t 类型的整数，长度为 4 字节。
 * |11100000| - 1 byte
 *      Integer encoded as int64_t (8 bytes).
 *      节点的值为 int64_t 类型的整数，长度为 8 字节。
 * |11110000| - 1 byte
 *      Integer encoded as 24 bit signed (3 bytes).
 *      节点的值为 24 位（3 字节）长的整数。
 * |11111110| - 1 byte
 *      Integer encoded as 8 bit signed (1 byte).
 *      节点的值为 8 位（1 字节）长的整数。
 * |1111xxxx| - (with xxxx between 0000 and 1101) immediate 4 bit integer.
 *      Unsigned integer from 0 to 12. The encoded value is actually from
 *      1 to 13 because 0000 and 1111 can not be used, so 1 should be
 *      subtracted from the encoded 4 bit value to obtain the right value.
 *      节点的值为介于 0 至 12 之间的无符号整数。
 *      因为 0000 和 1111 都不能使用，所以位的实际值将是 1 至 13 。
 *      程序在取得这 4 个位的值之后，还需要减去 1 ，才能计算出正确的值。
 *      比如说，如果位的值为 0001 = 1 ，那么程序返回的值将是 1 - 1 = 0 。
 * |11111111| - End of ziplist.
 *      ziplist 的结尾标识
 *
 * All the integers are represented in little endian byte order.
 *
 * 所有整数都表示为小端字节序。
 ```

 ziplist格式
 ```cpp
 /* 
空白 ziplist 示例图

area        |<---- ziplist header ---->|<-- end -->|

size          4 bytes   4 bytes 2 bytes  1 byte
            +---------+--------+-------+-----------+
component   | zlbytes | zltail | zllen | zlend     |
            |         |        |       |           |
value       |  1011   |  1010  |   0   | 1111 1111 |
            +---------+--------+-------+-----------+
                                       ^
                                       |
                               ZIPLIST_ENTRY_HEAD
                                       &
address                        ZIPLIST_ENTRY_TAIL
                                       &
                               ZIPLIST_ENTRY_END

非空 ziplist 示例图

area        |<---- ziplist header ---->|<----------- entries ------------->|<-end->|

size          4 bytes  4 bytes  2 bytes    ?        ?        ?        ?     1 byte
            +---------+--------+-------+--------+--------+--------+--------+-------+
component   | zlbytes | zltail | zllen | entry1 | entry2 |  ...   | entryN | zlend |
            +---------+--------+-------+--------+--------+--------+--------+-------+
                                       ^                          ^        ^
address                                |                          |        |
                                ZIPLIST_ENTRY_HEAD                |   ZIPLIST_ENTRY_END
                                                                  |
                                                        ZIPLIST_ENTRY_TAIL
*/
```
zltail是从ztail开始到zlend之前这段内存偏移的大小. 空白图例中的ztail+zllen  
zlbytes是从zlbytes开始到zlend最后这段内存的偏移. 空白图例中的zlbytes+zltail+zllen+zlend

zlentry结构
```cpp
/*
 * 保存 ziplist 节点信息的结构
 */
typedef struct zlentry {

    // prevrawlen ：前置节点的长度
    // prevrawlensize ：编码 prevrawlen 所需的字节大小
    unsigned int prevrawlensize, prevrawlen;

    // len ：当前节点值的长度
    // lensize ：编码 len 所需的字节大小
    unsigned int lensize, len;

    // 当前节点 header 的大小
    // 等于 prevrawlensize + lensize
    unsigned int headersize;

    // 当前节点值所使用的编码类型
    unsigned char encoding;

    // 指向当前节点的指针
    unsigned char *p;

} zlentry;
```

### __ziplistCascadeUpdate
因为ziplist是连续的,如果元素更改的时候需要调整大小,需要zrelloc,后续元素也需要往后挪动.  
同理,缩小的时候也会有这样的问题.但是为了避免反复出现扩展/缩小的情景,缩小的时候内存不调整,任由prevlen更长.
每次发现下一个节点需要挪动的时候,zrealloc以及memmove  
如果发现有节点不需要扩展,后续的都不需要扩展,就不需要挪动.
最极端的情况是如果都是253字节长度,第一个entry扩展之后,因为下一个的prevlen需要扩展成5bytes,导致自身长度增加,引发后续的所有entry都需要扩展.  
但是如果到某一个的prevlen不需要扩展,他的下一个也不需要更改.这就是为什么发现有节点不需要扩展,后续都不需要.


# 对象
对象结构
```cpp
typedef struct redisObject {

    // 类型
    unsigned type:4;

    // 编码
    unsigned encoding:4;

    // 对象最后一次被访问的时间
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */

    // 引用计数
    int refcount;

    // 指向实际值的指针
    void *ptr;

} robj;
```
其中lru是REDIS_LRU_BITS,24.  
对象被创建的时候  
```cpp
robj *createObject(int type, void *ptr) {

    robj *o = zmalloc(sizeof(*o));

    o->type = type; //类型
    o->encoding = REDIS_ENCODING_RAW;  //编码规则
    o->ptr = ptr; //数据
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution). */
    o->lru = LRU_CLOCK();
    return o;
}
```
初始的lru是LRU_CLOCK
```cpp
#define LRU_CLOCK() ((1000/server.hz <= REDIS_LRU_CLOCK_RESOLUTION) ? server.lruclock : getLRUClock())
```
1000/server.hz 是server每秒被调用的次数  默认10次,最小1 最大500
REDIS_LRU_CLOCK_RESOLUTION 是1000,    这里应该理解成**多少毫秒作为一个采样周期**.  
server.lruclock后续被调整  

如果当前1000/hz <= 1000,也就是当前系统没有被调用,使用getLRUClock;否则使用服务的lruclock.
```cpp
unsigned int getLRUClock(void) {
    return (mstime()/REDIS_LRU_CLOCK_RESOLUTION) & REDIS_LRU_CLOCK_MAX;
}
```
```cpp
#define REDIS_LRU_CLOCK_MAX ((1<<REDIS_LRU_BITS)-1)
```
所以LRU的上限是2^24 - 1,getLRUClock拿到的就是秒数时间戳. 2^24 -1 大约是388天.所以一个LRU周期是388天  
综上:如果服务器最近一秒被访问超过一次,也就是最近server的时钟是可信的,因为在最近的采样周期内server被访问过.1000表示的是1000个ms.这里的意思是当前的服务器采样周期是否比REDIS_LRU_CLOCK_RESOLUTION,也就是默认的采样周期要小.如果采样周期比默认的都小,说明这时候服务的采样是可信的,精度更高,直接使用server的lrulclock.否则计算一次,拿到一个秒级别的时间戳(需要&2<<24 -1)

除非REDIS_LRU_CLOCK_RESOLUTION设置比较小,采样周期比较小,否则都会拿到server.lruclock  

### 字符串类型
REDIS_ENCODING_EMBSTR  
使用连续的空间,redisObject,sds以及sds的buf空间连续.  
当长度<=39 使用EMBSTR类型,否则使用RawStringObject(buf内存和sds不连续)  
#### 39作为分水岭
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
```cpp
typedef struct redisObject {

    // 类型
    unsigned type:4;

    // 编码
    unsigned encoding:4;

    // 对象最后一次被访问的时间
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */

    // 引用计数
    int refcount;

    // 指向实际值的指针
    void *ptr;

} robj;
```
robj:8+8+8+4+8 =36
sds:4+4+? = 8+?   
还要再最后留一个字符给\0,39的意义是**刚好整块内存空间是64,满足jmalloc的一个小内存分配**,字符对齐且满足日常小字符串使用.  
```cpp
we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc
```

#### robj *createStringObjectFromLongLong(long long value)
传入的整数值保存为字符串对象  
- 如果小于10000,会引用预先开辟好的共享内存空间(常用小数字,字符等),减少内存碎片
- 如果value大小在0x7fffffffffffffffL内,直接转为void*,就是用ptr存放数据,编码类型REDIS_ENCODING_INT.  
- 如果value大小超过0x7fffffffffffffffL,新建一个REDIS_STRING对象,里面的sds的buff放的是value转化为字符串之后的buffer.
```cpp
robj *createStringObjectFromLongLong(long long value) {

    robj *o;

    // value 的大小符合 REDIS 共享整数的范围
    // 那么返回一个共享对象
    if (value >= 0 && value < REDIS_SHARED_INTEGERS) {
        incrRefCount(shared.integers[value]);
        o = shared.integers[value];

    // 不符合共享范围，创建一个新的整数对象
    } else {
        // 值可以用 long 类型保存，
        // 创建一个 REDIS_ENCODING_INT 编码的字符串对象
        if (value >= LONG_MIN && value <= LONG_MAX) {
            o = createObject(REDIS_STRING, NULL);
            o->encoding = REDIS_ENCODING_INT;
            o->ptr = (void*)((long)value);

        // 值不能用 long 类型保存（long long 类型），将值转换为字符串，
        // 并创建一个 REDIS_ENCODING_RAW 的字符串对象来保存值
        } else {
            o = createObject(REDIS_STRING,sdsfromlonglong(value));
        }
    }

    return o;
}
```

#### robj *createStringObjectFromLongDouble(long double value) 
使用17位小数,初始化之后对字符串最后的0删除.如果最后的是小数点,直接删除.  
然后createStringObject

#### robj *dupStringObject(robj *o) 
新建一个对象,包括字符串内容.  
根据对象的类型:REDIS_ENCODING_RAW,REDIS_ENCODING_EMBSTR,REDIS_ENCODING_INT分别对应复制.  
新对象refcount=1,就是当做新对象对待.

#### 新建对象类型

|函数|对象类型|对象编码|
|---|---|---|
|createListObject|REDIS_LIST|REDIS_ENCODING_LINKEDLIST|
|createZiplistObject|REDIS_LIST|REDIS_ENCODING_ZIPLIST|
|createHashObject|REDIS_HASH|REDIS_ENCODING_ZIPLIST|
|createZsetObject|REDIS_ZSET|REDIS_ENCODING_SKIPLIST|
|createZsetZiplistObject|REDIS_ZSET|REDIS_ENCODING_ZIPLIST|

#### 对象编码类型
- list类
    - REDIS_ENCODING_LINKEDLIST 释放需要挨个节点释放
    - REDIS_ENCODING_ZIPLIST 直接zfree
- set类
    - REDIS_ENCODING_HT 需要挨个ht的table释放
    - REDIS_ENCODING_INTSET  直接zfree
- zset类
    - REDIS_ENCODING_SKIPLIST  挨个跳表节点释放
    - REDIS_ENCODING_ZIPLIST  直接释放
- 哈希类
    - REDIS_ENCODING_HT  挨个ht释放
    - REDIS_ENCODING_ZIPLIST  zfree

#### decrRefCount
对象引用减少.减少到0时候释放内存.  
类似inode  

#### robj *tryObjectEncoding(robj *o)
尝试对字符串对象进行编码,节约内存.  
只在字符串的编码为 RAW 或者 EMBSTR 时尝试进行编码  
REDIS_ENCODING_EMBSTR尝试转换为REDIS_ENCODING_INT  
REDIS_ENCODING_RAW尝试转换为REDIS_ENCODING_EMBSTR,失败的话sdsRemoveFreeSpace(zrealloc)

#### unsigned long long estimateObjectIdleTime(robj *o)
```cpp
unsigned long long estimateObjectIdleTime(robj *o) {
    unsigned long long lruclock = LRU_CLOCK();
    if (lruclock >= o->lru) {
        return (lruclock - o->lru) * REDIS_LRU_CLOCK_RESOLUTION;
    } else {
        return (lruclock + (REDIS_LRU_CLOCK_MAX - o->lru)) *
                    REDIS_LRU_CLOCK_RESOLUTION;
    }
}
```
LRU_CLOCK可以看成是秒(ts&2^24 - 1)  
如果当前的时间戳比对象的lru大,直接返回毫秒格式的  
如果当前对象的lru比当前的时间戳大,说明可能已经过了一个轮回,lru置0.所以实际的空转时间是max-o.lru+lurclock  
```ditaa
┌─────────────────────────────────────────────────────────────┐       
│                            o.lru                            │       
├─────────────────────────────────────────────────────────────┴──────┐
│                              LRU_MAX                               │
├───────────┬────────────────────────────────────────────────────────┘
│LRU_CLOCK()│                                                         
└───────────┘                                                         
```

### zset
```cpp
/*
 * 有序集合
 */
typedef struct zset {

    // 字典，键为成员，值为分值
    // 用于支持 O(1) 复杂度的按成员取分值操作
    dict *dict;

    // 跳跃表，按分值排序成员
    // 用于支持平均复杂度为 O(log N) 的按分值定位成员操作
    // 以及范围操作
    zskiplist *zsl;

} zset;
```

zset中包括跳表和dict
如果zset同时满足下面条件:  
- 元素数量<128
- 每个元素长度<64 b

zset可以退化成ziplist.注意,**此时仅有ziplist,没有dict**  
#### 编码转换
```cpp
        // 有序集合在 ziplist 中的排列：
        //
        // | member-1 | score-1 | member-2 | score-2 | ... |
        //
```
ziplist->skiplist(+dict):每次取出member和score,zsInsert给skiplist,dictAdd给dict
```cpp
    if (zobj->encoding == REDIS_ENCODING_ZIPLIST) {
        unsigned char *zl = zobj->ptr;
        unsigned char *eptr, *sptr;
        unsigned char *vstr;
        unsigned int vlen;
        long long vlong;

        if (encoding != REDIS_ENCODING_SKIPLIST)
            redisPanic("Unknown target encoding");

        // 创建有序集合结构
        zs = zmalloc(sizeof(*zs));
        // 字典
        zs->dict = dictCreate(&zsetDictType,NULL);
        // 跳跃表
        zs->zsl = zslCreate();

        // 有序集合在 ziplist 中的排列：
        //
        // | member-1 | score-1 | member-2 | score-2 | ... |
        //
        // 指向 ziplist 中的首个节点（保存着元素成员）
        eptr = ziplistIndex(zl,0);
        redisAssertWithInfo(NULL,zobj,eptr != NULL);
        // 指向 ziplist 中的第二个节点（保存着元素分值）
        sptr = ziplistNext(zl,eptr);
        redisAssertWithInfo(NULL,zobj,sptr != NULL);

        // 遍历所有 ziplist 节点，并将元素的成员和分值添加到有序集合中
        while (eptr != NULL) {
            
            // 取出分值
            score = zzlGetScore(sptr);

            // 取出成员
            redisAssertWithInfo(NULL,zobj,ziplistGet(eptr,&vstr,&vlen,&vlong));
            if (vstr == NULL)
                ele = createStringObjectFromLongLong(vlong);
            else
                ele = createStringObject((char*)vstr,vlen);

            /* Has incremented refcount since it was just created. */
            // 将成员和分值分别关联到跳跃表和字典中
            node = zslInsert(zs->zsl,score,ele);
            redisAssertWithInfo(NULL,zobj,dictAdd(zs->dict,ele,&node->score) == DICT_OK);
            incrRefCount(ele); /* Added to dictionary. */

            // 移动指针，指向下个元素
            zzlNext(zl,&eptr,&sptr);//每次eptr和sptr都跳两次
        }

        // 释放原来的 ziplist
        zfree(zobj->ptr);

        // 更新对象的值，以及编码方式
        zobj->ptr = zs;
        zobj->encoding = REDIS_ENCODING_SKIPLIST;

    // 从 SKIPLIST 转换为 ZIPLIST 编码
    } 
```
注意点:
- ele对象是**新建**的,所以跳表和字典的ele是共享的,不会有重复的ele对象
- ele需要新建是因为后续ziplist会释放掉,原来的数据都会gg
- **字典的key是ele的指针,value就是score**
- 跳表为了统计快,字典为了查找快


skiplist->ziplist:释放dict,遍历skiplist,依次按照member,score写入到skiplist.  
```cpp
else if (zobj->encoding == REDIS_ENCODING_SKIPLIST) {

        // 新的 ziplist
        unsigned char *zl = ziplistNew();

        if (encoding != REDIS_ENCODING_ZIPLIST)
            redisPanic("Unknown target encoding");

        /* Approach similar to zslFree(), since we want to free the skiplist at
         * the same time as creating the ziplist. */
        // 指向跳跃表
        zs = zobj->ptr;

        // 先释放字典，因为只需要跳跃表就可以遍历整个有序集合了
        dictRelease(zs->dict);

        // 指向跳跃表首个节点
        node = zs->zsl->header->level[0].forward;

        // 释放跳跃表表头
        zfree(zs->zsl->header);
        zfree(zs->zsl);

        // 遍历跳跃表，取出里面的元素，并将它们添加到 ziplist
        while (node) {

            // 取出解码后的值对象
            ele = getDecodedObject(node->obj);

            // 添加元素到 ziplist
            zl = zzlInsertAt(zl,NULL,ele,node->score);
            decrRefCount(ele);

            // 沿着跳跃表的第 0 层前进
            next = node->level[0].forward;
            zslFreeNode(node);
            node = next;
        }

        // 释放跳跃表
        zfree(zs);

        // 更新对象的值，以及对象的编码方式
        zobj->ptr = zl;
        zobj->encoding = REDIS_ENCODING_ZIPLIST;
    } 
```
注意:
- 因为字典释放的时候根据(d)->type->valDestructor方法析构val(具体节点),是否val被释放要看zset创建
- 根据顺序member score加入ziplist
- 最后ele不是直接删除,而是decrRefCount.引用为0自动删除.防止其他对象有对ele引用报错

#### createZsetObject
创建skiplist的zset
```cpp
robj *createZsetObject(void) {

    zset *zs = zmalloc(sizeof(*zs));

    robj *o;

    zs->dict = dictCreate(&zsetDictType,NULL);
    zs->zsl = zslCreate();

    o = createObject(REDIS_ZSET,zs);

    o->encoding = REDIS_ENCODING_SKIPLIST;

    return o;
}
```
注意:object是REDIS_ZSET类型,编码是REDIS_ENCODING_SKIPLIST

#### createZsetZiplistObject
```cpp
robj *createZsetZiplistObject(void) {

    unsigned char *zl = ziplistNew();

    robj *o = createObject(REDIS_ZSET,zl);

    o->encoding = REDIS_ENCODING_ZIPLIST;

    return o;
}
```
注意:本质就是一个ziplist,只是对象的type是REDIS_ZSET

## robj *lookupKey(redisDb *db, robj *key)
redis在db中找key对应的指针   底层使用dict存储.
```cpp
robj *lookupKey(redisDb *db, robj *key) {

    // 查找键空间
    dictEntry *de = dictFind(db->dict,key->ptr);

    // 节点存在
    if (de) {
        

        // 取出值
        robj *val = dictGetVal(de);

        /* Update the access time for the ageing algorithm.
         * Don't do it if we have a saving child, as this will trigger
         * a copy on write madness. */
        // 更新时间信息（只在不存在子进程时执行，防止破坏 copy-on-write 机制）
        if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
            val->lru = LRU_CLOCK();

        // 返回值
        return val;
    } else {

        // 节点不存在

        return NULL;
    }
}
```
注意:只有在没有子进程的时候才会更新对象的lru.更新lru会改变内存,有子进程的时候会避免更改


#### zaddGenericCommand
如果server的dict(db的管理key模块,使用dict)没有key,新建:  
- 如果配置server.zset_max_ziplist_entries是0,或者server.zset_max_ziplist_value(默认64)小于第一个key的长度,使用createZsetObject
- 否则使用createZsetZiplistObject
- 新建对象添加到c->db 使用dbAdd

对象如果存在,检查type  
- ziplist类型
    - 如果之前没存在,添加
    - 如果之前存在,score加上,然后删除,添加(更新流程)
- skiplist类型
    - 之前没存在,添加
    - 之前存在,skiplist删除后重新添加(更新流程),dict直接对value更新

最后清理临时变量

# db
db结构体
```cpp
typedef struct redisDb {

    // 数据库键空间，保存着数据库中的所有键值对
    dict *dict;                 /* The keyspace for this DB */

    // 键的过期时间，字典的键为键，字典的值为过期事件 UNIX 时间戳
    dict *expires;              /* Timeout of keys with a timeout set */

    // 正处于阻塞状态的键
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP) */

    // 可以解除阻塞的键
    dict *ready_keys;           /* Blocked keys that received a PUSH */

    // 正在被 WATCH 命令监视的键
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */

    struct evictionPoolEntry *eviction_pool;    /* Eviction pool of keys */

    // 数据库号码
    int id;                     /* Database ID */

    // 数据库的键的平均 TTL ，统计信息
    long long avg_ttl;          /* Average TTL, just for stats */

} redisDb;
```
dcit:key是kv的key,v是对象指针  
expires 记录key 的过期时间,value是unix时间戳  
blocking_keys 记录阻塞状态的key  BLPOP使用
ready_keys 记录可以解除阻塞的key PUSH使用
watched_keys  记录正在waich的key,value是对应的client链表.  MULTI/EXEC使用  


### robj *lookupKey(redisDb *db, robj *key)
在db.dict中查找并返回.
如果存在,当前有子进程,不更新lru(否则子进程需要重新拷贝一次内存,意义不大)

### robj *lookupKeyRead(redisDb *db, robj *key)
```cpp
robj *lookupKeyRead(redisDb *db, robj *key) {
    robj *val;

    // 检查 key 释放已经过期
    expireIfNeeded(db,key);

    // 从数据库中取出键的值
    val = lookupKey(db,key);

    // 更新命中/不命中信息
    if (val == NULL)
        server.stat_keyspace_misses++;
    else
        server.stat_keyspace_hits++;

    // 返回值
    return val;
}
```
先看是否过期,如果过期,根据情况会释放掉过期的数据.  
然后从dict中找value的指针.根据找到的情况更新命中/不命中信息.  
这里server是全局信息.  


### int expireIfNeeded(redisDb *db, robj *key) 
看是否过期,如果key过期会删掉.
```cpp
int expireIfNeeded(redisDb *db, robj *key) {

    // 取出键的过期时间
    mstime_t when = getExpire(db,key);
    mstime_t now;

    // 没有过期时间
    if (when < 0) return 0; /* No expire for this key */

    /* Don't expire anything while loading. It will be done later. */
    // 如果服务器正在进行载入，那么不进行任何过期检查
    if (server.loading) return 0;

    /* If we are in the context of a Lua script, we claim that time is
     * blocked to when the Lua script started. This way a key can expire
     * only the first time it is accessed and not in the middle of the
     * script execution, making propagation to slaves / AOF consistent.
     * See issue #1525 on Github for more information. */
    now = server.lua_caller ? server.lua_time_start : mstime();

    /* If we are running in the context of a slave, return ASAP:
     * the slave key expiration is controlled by the master that will
     * send us synthesized DEL operations for expired keys.
     *
     * Still we try to return the right information to the caller, 
     * that is, 0 if we think the key should be still valid, 1 if
     * we think the key is expired at this time. */
    // 当服务器运行在 replication 模式时
    // 附属节点并不主动删除 key
    // 它只返回一个逻辑上正确的返回值
    // 真正的删除操作要等待主节点发来删除命令时才执行
    // 从而保证数据的同步

    if (server.masterhost != NULL) return now > when;

    // 运行到这里，表示键带有过期时间，并且服务器为主节点

    /* Return when this key has not expired */
    // 如果未过期，返回 0
    if (now <= when) return 0;

    /* Delete the key */ 

    //能到这里的逻辑已经是master
    server.stat_expiredkeys++;

    // 向 AOF 文件和附属节点传播过期信息
 
    propagateExpire(db,key);

    // 发送事件通知
    notifyKeyspaceEvent(REDIS_NOTIFY_EXPIRED,
        "expired",key,db->id);

    // 将过期键从数据库中删除
    return dbDelete(db,key);
}
```
when:当前key的过期时间,如果-1,没有设置    
now:当前时间  
如果是slave,只返回结果,不删除.因为删除流程从master发起  
如果正常没过期,返回0  
主机删除需要的动作:
- server过期key个数++
- AOF文件写入删除信息
- 发送事件给client:已经过期
- 从db.dict删除key

### robj *lookupKeyWrite(redisDb *db, robj *key)
expireIfNeeded + lookupKey  
不会统计命中信息

### robj *lookupKeyReadOrReply(redisClient *c, robj *key, robj *reply)
lookupKeyRead  
如果不存在,addReply给客户端发送reply信息 与客户端交互使用

### robj *lookupKeyWriteOrReply(redisClient *c, robj *key, robj *reply)
lookupKeyWrite
如果不存在,addReply给客户端发送reply信息 与客户端交互使用

### void dbAdd(redisDb *db, robj *key, robj *val)
dictAdd  
如果开启了集群模式,slotToKeyAdd  

### void slotToKeyAdd(robj *key)
计算key的hash值,(使用低key crc16后&0x3FFF),往server.cluster->slots_to_keys这个跳表写入,score是key的hash,value是key  
使用跳表的好处是:跳表的score是根据key计算的,所以根据score已经区分了位置,所以想知道这个key是否在某个slot中比较方便.(key是根据字典序在跳表中排列)  
最后添加对key的引用
```cpp
void slotToKeyAdd(robj *key) {

    // 计算出键所属的槽
    unsigned int hashslot = keyHashSlot(key->ptr,sdslen(key->ptr));

    // 将槽 slot 作为分值，键作为成员，添加到 slots_to_keys 跳跃表里面
    zslInsert(server.cluster->slots_to_keys,hashslot,key);
    incrRefCount(key);
}
```

### void dbOverwrite(redisDb *db, robj *key, robj *val) 
目的是把key的value更改,也就是把key对应的对象进行更改.
db.dict找key  
如果不存在,gg  
存在,dictReplace


### void setKey(redisDb *db, robj *key, robj *val)
是一种不会过期的set key.不存在就加上,存在的话就直接更改.  
需要告知watch的client有更改.使用signalModifiedKey,实际上调用的就是touchWatchedKey
```cpp
    // 添加或覆写数据库中的键值对
    if (lookupKeyWrite(db,key) == NULL) {
        dbAdd(db,key,val);
    } else {
        dbOverwrite(db,key,val);
    }

    incrRefCount(val);

    // 移除键的过期时间
    removeExpire(db,key);

    // 发送键修改通知
    signalModifiedKey(db,key);
```

### void touchWatchedKey(redisDb *db, robj *key)
如果某个键被客户端监视,这些cli执行exec时事务失败.  
获取db->watched_keys,本质上是一个dict,key是kv的key,value是client列表.根据key是否在被watch添加到这个字典中.  
如果key在db->watched_keys中存在,clients中的所有flags都打上REDIS_DIRTY_CAS,表示watch的时候已经被修改.后续exec的时候会失败.  

### int dbDelete(redisDb *db, robj *key)
dictDelete(db->expires,key->ptr)  
dictDelete(db->dict,key->ptr)  
如果是集群模式slotToKeyDel(key)  
不存在返回0,存在正常删除返回1  

### void slotToKeyDel(robj *key)
zslDelete(server.cluster->slots_to_keys,hashslot,key)  
把当前的db中slots_to_keys对应的hashslot的key删除.

### void slotToKeyFlush(void)
清空节点中所有槽保存的所有键  
先对server.cluster->slots_to_keys进行zslFree  
再进行zslCreate()  

### unsigned int getKeysInSlot(unsigned int hashslot, robj **keys, unsigned int count) 
记录入参是hashslot,以及count表示数组大小. 
回参keys,robj数组,表示属于hashslot中count个的robj 
把这些数组都放进去hashslot中,并且返回加入之后这个slot的key个数.

### unsigned int delKeysInSlot(unsigned int hashslot) 
删除hashslot的key,返回删除的个数

### unsigned int countKeysInSlot(unsigned int hashslot)
计算hashslot的key个数.  
使用rank头尾相减得到

### long long emptyDb(void(callback)(void*))
清空db所有数据  
对每个db[]下的dict以及expires进行清空(dictEmpty)  
如果server.cluster_enabled,slotToKeyFlush  
返回删除key个数

### int selectDb(redisClient *c, int id)
c.db=server.db[id]

### void signalFlushedDb(int dbid)
db清空的时候会触发,调用touchWatchedKeysOnFlush

### void touchWatchedKeysOnFlush(int dbid)
当某个db清空的时候,如果cli的watch key的db是传入的dbid,需要打标REDIS_DIRTY_CAS  
server的cli是链表形式存放,client的watch_keys也使用链表形式存放

### void flushdbCommand(redisClient *c)
清空客户端所有的数据库  
signalFlushedDb(c->db->id),对这个db的所有关联cli发通知  
清空cli.db.dict cli.d..expires  
如果开启集群模式slotToKeyFlush  
给cli返回成功信息

### void flushallCommand(redisClient *c) 
清空server所有db  
signalFlushedDb(-1)   
如果有正在保存的新的rdb,取消保存操作(kill进程,unlink文件)  
如果当前server还有saving points,rdbSave之后把原来的server.dirty属性在执行rdbSave后恢复到server(因为rdbSave的时候会调整dirty属性)


### void delCommand(redisClient *c)
c会有需要删除的键  使用命令DEL k1 k2 ...  
先删除过期的key  
然后dbDelete(c->db,c->argv[j]),成功的话db->watched_keys写入dirty(其他watch这个key的client会感知)  
notifyKeyspaceEvent,把删除事件通知到对应的db  

### void notifyKeyspaceEvent(int type, char *event, robj *key, int dbid) 
- event:c字符串,表示事件名称
- key:事件相关的key
- dbid:key所在的dbid
- type:发送通知类型

如果配置server.notify_keyspace_events不发送对应的type,直接返回  
如果配置了发送REDIS_NOTIFY_KEYSPACE  
chan格式: __keyspace@<db>__:<key>  发送<event>

如果配置了发送REDIS_NOTIFY_KEYEVENT
chan格式:__keyevente@<db>__:<event> 发送<key>

使用pubsubPublishMessage发送消息

### int pubsubPublishMessage(robj *channel, robj *message)
如果channel在server的订阅频道中存在(使用dict存储),这个channel下挂的cli链表发送消息  
如果server有pubsub_patterns(客户端订阅的所有模式的名字,list,内容是cli和模式结构体),如果模式匹配(正则表达式),给对应的cli发送消息


```cpp

  int pubsubPublishMessage(robj *channel, robj *message) {
      int receivers = 0;
      dictEntry *de;
      listNode *ln;
      listIter li;

      /* Send to clients listening for that channel */
      // 取出包含所有订阅频道 channel 的客户端的链表
      // 并将消息发送给它们
      de = dictFind(server.pubsub_channels,channel);
      if (de) {
          list *list = dictGetVal(de);
          listNode *ln;
          listIter li;

          // 遍历客户端链表，将 message 发送给它们
          listRewind(list,&li);
          while ((ln = listNext(&li)) != NULL) {
              redisClient *c = ln->value;

              // 回复客户端。
              // 示例：
              // 1) "message"
              // 2) "xxx"
              // 3) "hello"
              addReply(c,shared.mbulkhdr[3]);
              // "message" 字符串
              addReply(c,shared.messagebulk);
              // 消息的来源频道
              addReplyBulk(c,channel);
              // 消息内容
              addReplyBulk(c,message);

              // 接收客户端计数
              receivers++;
          }
      }

      /* Send to clients listening to matching channels */
      // 将消息也发送给那些和频道匹配的模式
      if (listLength(server.pubsub_patterns)) {

          // 遍历模式链表
          listRewind(server.pubsub_patterns,&li);
          channel = getDecodedObject(channel);
          while ((ln = listNext(&li)) != NULL) {

              // 取出 pubsubPattern
              pubsubPattern *pat = ln->value;

              // 如果 channel 和 pattern 匹配
              // 就给所有订阅该 pattern 的客户端发送消息
              if (stringmatchlen((char*)pat->pattern->ptr,
                                  sdslen(pat->pattern->ptr),
                                  (char*)channel->ptr,
                                  sdslen(channel->ptr),0)) {

                  // 回复客户端
                  // 示例：
                  // 1) "pmessage"
                  // 2) "*"
                  // 3) "xxx"
                  // 4) "hello"
                  addReply(pat->client,shared.mbulkhdr[4]);
                  addReply(pat->client,shared.pmessagebulk);
                  addReplyBulk(pat->client,pat->pattern);
                  addReplyBulk(pat->client,channel);
                  addReplyBulk(pat->client,message);

                  // 对接收消息的客户端进行计数
                  receivers++;
              }
          }

          decrRefCount(channel);
      }

      // 返回计数
      return receivers;
  }
```

### void existsCommand(redisClient *c)
先检查key是否过期  
给c返回是否存在

### void selectCommand(redisClient *c)
这里的select是切换db,换到对应dbid
1.看c的dbid是否合法,不合法gg  
2.看当前是否集群模式,如果是,不允许select  
3.切换db,失败gg  
4.给c返回  

### void keysCommand(redisClient *c)
返回keys,对c->db->dict遍历,addReplyBulk返回给c

### void scanGenericCommand(redisClient *c, robj *o, unsigned long cursor)
SCAN,HSCAN,SSCAN实现函数  
解析命令,根据对象encoding进行扫描,dict使用dictscan,跳表使用ite  
整合数据后返回

### void renameGenericCommand(redisClient *c, int nx)
更改key名字.RENAME a b
- 名字一样,报错
- 取出a,如果不存在,gg
- 查看b的key是否存在,存在分情况,RENAMENX直接返回,RENAME就删除b
- 把a在dict中的o给b

### void moveCommand(redisClient *c) 
从c.db移动到c->argv[2]指定的db 类似rename,只是在不同的dbidx中转移

### void propagateExpire(redisDb *db, robj *key)
将某个key的过期事件传播给AOF文件以及附属节点  
如果server.aof_state != REDIS_AOF_OFF,AOF中添加上删除key命令  
使用replicationFeedSlaves对各个server.slaves传播删除key命令

### void expireGenericCommand(redisClient *c, long long basetime, int unit)  
过期,主机且不在loading状态:notifyKeyspaceEvent 删除
其他:可能过期,notifyKeyspaceEvent expire

## redisDb
```cpp
typedef struct redisDb {

    // 数据库键空间，保存着数据库中的所有键值对
    dict *dict;                 /* The keyspace for this DB */

    // 键的过期时间，字典的键为键，字典的值为过期事件 UNIX 时间戳
    dict *expires;              /* Timeout of keys with a timeout set */

    // 正处于阻塞状态的键
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP) */

    // 可以解除阻塞的键
    dict *ready_keys;           /* Blocked keys that received a PUSH */

    // 正在被 WATCH 命令监视的键
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */

    struct evictionPoolEntry *eviction_pool;    /* Eviction pool of keys */

    // 数据库号码
    int id;                     /* Database ID */

    // 数据库的键的平均 TTL ，统计信息
    long long avg_ttl;          /* Average TTL, just for stats */

} redisDb;
```

### void activeExpireCycle(int type) 
定期删除  
```cpp
void activeExpireCycle(int type) {
    /* This function has some global state in order to continue the work
     * incrementally across calls. */
    // 静态变量，用来累积函数连续执行时的数据
    static unsigned int current_db = 0; /* Last DB tested. */
    static int timelimit_exit = 0;      /* Time limit hit in previous call? */
    static long long last_fast_cycle = 0; /* When last fast cycle ran. */

    unsigned int j, iteration = 0;
    // 默认每次处理的数据库数量
    unsigned int dbs_per_call = REDIS_DBCRON_DBS_PER_CALL;
    // 函数开始的时间
    long long start = ustime(), timelimit;

    // 快速模式
    if (type == ACTIVE_EXPIRE_CYCLE_FAST) {
        /* Don't start a fast cycle if the previous cycle did not exited
         * for time limt. Also don't repeat a fast cycle for the same period
         * as the fast cycle total duration itself. */
        // 如果上次函数没有触发 timelimit_exit ，那么不执行处理
        if (!timelimit_exit) return;
        // 如果距离上次执行未够一定时间，那么不执行处理
        if (start < last_fast_cycle + ACTIVE_EXPIRE_CYCLE_FAST_DURATION*2) return;
        // 运行到这里，说明执行快速处理，记录当前时间
        last_fast_cycle = start;
    }

    /* We usually should test REDIS_DBCRON_DBS_PER_CALL per iteration, with
     * two exceptions:
     *
     * 一般情况下，函数只处理 REDIS_DBCRON_DBS_PER_CALL 个数据库，
     * 除非：
     *
     * 1) Don't test more DBs than we have.
     *    当前数据库的数量小于 REDIS_DBCRON_DBS_PER_CALL
     * 2) If last time we hit the time limit, we want to scan all DBs
     * in this iteration, as there is work to do in some DB and we don't want
     * expired keys to use memory for too much time. 
     *     如果上次处理遇到了时间上限，那么这次需要对所有数据库进行扫描，
     *     这可以避免过多的过期键占用空间
     */
    if (dbs_per_call > server.dbnum || timelimit_exit)
        dbs_per_call = server.dbnum;

    /* We can use at max ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC percentage of CPU time
     * per iteration. Since this function gets called with a frequency of
     * server.hz times per second, the following is the max amount of
     * microseconds we can spend in this function. */
    // 函数处理的微秒时间上限
    // ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC 默认为 25 ，也即是 25 % 的 CPU 时间
    timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;
    //hz是每秒的次数.1000000/hz就是看us/time,表示这次处理的时候最多是多少个微秒.  
    timelimit_exit = 0;
    if (timelimit <= 0) timelimit = 1;

    // 如果是运行在快速模式之下
    // 那么最多只能运行 FAST_DURATION 微秒 
    // 默认值为 1000 （微秒）
    if (type == ACTIVE_EXPIRE_CYCLE_FAST)
        timelimit = ACTIVE_EXPIRE_CYCLE_FAST_DURATION; /* in microseconds. */

    // 遍历数据库
    for (j = 0; j < dbs_per_call; j++) {
        int expired;
        // 指向要处理的数据库
        redisDb *db = server.db+(current_db % server.dbnum);

        /* Increment the DB now so we are sure if we run out of time
         * in the current DB we'll restart from the next. This allows to
         * distribute the time evenly across DBs. */
        // 为 DB 计数器加一，如果进入 do 循环之后因为超时而跳出
        // 那么下次会直接从下个 DB 开始处理
        current_db++;

        /* Continue to expire if at the end of the cycle more than 25%
         * of the keys were expired. */
        do {
            unsigned long num, slots;
            long long now, ttl_sum;
            int ttl_samples;

            /* If there is nothing to expire try next DB ASAP. */
            // 获取数据库中带过期时间的键的数量
            // 如果该数量为 0 ，直接跳过这个数据库
            if ((num = dictSize(db->expires)) == 0) {
                db->avg_ttl = 0;
                break;
            }
            // 获取数据库中键值对的数量
            slots = dictSlots(db->expires);
            // 当前时间
            now = mstime();

            /* When there are less than 1% filled slots getting random
             * keys is expensive, so stop here waiting for better times...
             * The dictionary will be resized asap. */
            // 这个数据库的使用率低于 1% ，扫描起来太费力了（大部分都会 MISS）
            // 跳过，等待字典收缩程序运行
            if (num && slots > DICT_HT_INITIAL_SIZE &&
                (num*100/slots < 1)) break;

            /* The main collection cycle. Sample random keys among keys
             * with an expire set, checking for expired ones. 
             *
             * 样本计数器
             */
            // 已处理过期键计数器
            expired = 0;
            // 键的总 TTL 计数器
            ttl_sum = 0;
            // 总共处理的键计数器
            ttl_samples = 0;

            // 每次最多只能检查 LOOKUPS_PER_LOOP 个键
            if (num > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP)
                num = ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP;

            // 开始遍历数据库
            while (num--) {
                dictEntry *de;
                long long ttl;

                // 从 expires 中随机取出一个带过期时间的键
                if ((de = dictGetRandomKey(db->expires)) == NULL) break;
                // 计算 TTL
                ttl = dictGetSignedIntegerVal(de)-now;
                // 如果键已经过期，那么删除它，并将 expired 计数器增一
                if (activeExpireCycleTryExpire(db,de,now)) expired++;
                if (ttl < 0) ttl = 0;
                // 累积键的 TTL
                ttl_sum += ttl;
                // 累积处理键的个数
                ttl_samples++;
            }

            /* Update the average TTL stats for this database. */
            // 为这个数据库更新平均 TTL 统计数据
            if (ttl_samples) {
                // 计算当前平均值
                long long avg_ttl = ttl_sum/ttl_samples;
                
                // 如果这是第一次设置数据库平均 TTL ，那么进行初始化
                if (db->avg_ttl == 0) db->avg_ttl = avg_ttl;
                /* Smooth the value averaging with the previous one. */
                // 取数据库的上次平均 TTL 和今次平均 TTL 的平均值
                db->avg_ttl = (db->avg_ttl+avg_ttl)/2;
            }

            /* We can't block forever here even if there are many keys to
             * expire. So after a given amount of milliseconds return to the
             * caller waiting for the other active expire cycle. */
            // 我们不能用太长时间处理过期键，
            // 所以这个函数执行一定时间之后就要返回

            // 更新遍历次数
            iteration++;

            // 每遍历 16 次执行一次
            if ((iteration & 0xf) == 0 && /* check once every 16 iterations. */
                (ustime()-start) > timelimit)
            {
                // 如果遍历次数正好是 16 的倍数
                // 并且遍历的时间超过了 timelimit
                // 那么断开 timelimit_exit
                timelimit_exit = 1;
            }

            // 已经超时了，返回
            if (timelimit_exit) return;

            /* We don't repeat the cycle if there are less than 25% of keys
             * found expired in the current DB. */
            // 如果已删除的过期键占当前总数据库带过期时间的键数量的 25 %
            // 那么不再遍历
        } while (expired > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4);
    }
}
```

注意:  
如果过期超过5个,继续循环.因为最后一次可能会少于5个,就是没有expire可以删  
如果扫了16次还没有超过时间限制,继续扫.需要**超过时间限制并且扫了16的倍数**才完成  
hz是每秒的次数.1000000/hz就是看us/time,表示这次处理的时候最多是多少个微秒.  

# redis


## server

### 过期操作

- 1.如果是从机,只返回结果不进行操作

	- 主机进行操作后同步del命令

- 2.如果是主机

	- 1.向从机传播过期节点,使用del key

	- 2.notifyKeyspaceEvent,发送过期事件通知

	- 3.从db删除

## redisDb

### dict

- 存放当前所有keys

- 类型:dict

- key:key

- value:对象

### expires

- 存放过期时间

- 类型:dict

- key:key

- value:unix时间戳

### blocking_keys

- 处于阻塞状态的keys

- 类型:dict

- keys:key

### ready_keys

- 可以解除阻塞的keys

- 类型:dict

- key:key

### watched_keys

- 正在被watch监视的keys

- 类型:dict

- key:key

### eviction_pool

### id

- 类型:int

- dbidx

### avg_ttl

- key平均ttl,用于统计

## 过期删除

### 惰性删除策略

- 实现:expireIfNeeded

- 调用场景

	- lookupKeyRead,查找

	- lookupKeyWrite,查找

	- dbRandomKey,随机返回key

	- delCommand,删除

	- existsCommand,查看是否存在

	- keysCommand,遍历db的keys

	- scanGenericCommand,scan调用

- 步骤

	- 1.从db.expires查找key的过期时间,没有直接返回

	- 2.如果server.loading,当前载入rdb,直接返回不检查

	- 3.如果是从机,直接返回不删除.从机等待主机同步删除命令后删除.

	- 4.数据没过期,直接返回

	- 5.数据过期

		- 5.1.向AOF文件追加del命令,传播给从节点

		- 5.2.notifyKeyspaceEvent对key和[db.id](http://db.id)通知键空间,这个key现在过期

		- 5.3.从db.dict删除key

### 定期删除策略

- 实现activeExpireCycle

- 步骤

	- 基于server.hz的25%求出这次处理应该占用多少耗时

		- timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;

	- ACTIVE_EXPIRE_CYCLE_FAST

		- 上次函数不触发timelimit_exit,跳过

		- 如果上次执行到现在不超过2s,跳过

		- 如果超过2s,记录本次执行时间,继续往下

	- 遍历db

		- 如果当前删除的个数>5,继续循环

			- 如果db.expires的占用率小于1%,跳过,等rehash

			- 每次最多删20个key,随机删

			- 计算ttl,过期时间和当前时间的时间差用于统计

			- 每16次循环看一下耗时是不是超过了步骤1计算的耗时.如果是,退出

		- 如果删除个数小于5个,可能因为后面已经删完

- 调用场景

	- beforeSleep

		- 事件触发之前执行

		- 类型:ACTIVE_EXPIRE_CYCLE_FAST

	- databasesCron

		- server周期触发

		- 类型:ACTIVE_EXPIRE_CYCLE_SLOW

## 定期策略

### databasesCron

- 主机且配置了server.active_expire_enabled

	- 删除过期key

	- activeExpireCycle(ACTIVE_EXPIRE_CYCLE_SLOW)

- 当前没有bgsave且没有aofrewrite,遍历每个db

	- 步骤

		- 看dict,expires是否需要缩容,尝试缩容

			- tryResizeHashTables

		- 如果配置了server.activerehashing

			- incrementallyRehash

				- 处理dict和expires

				- 如果正在rehashing,每次步长100rehash,超过1ms退出

	- 如果server长期没有执行命令,dict很难完成rehash.正常情况下应该让databasecron推动db.dict和db.expires的rehash

## rdb

### rdb日志文件格式

- REDIS

- db_version

- databases

	- SELECTDB

	- db_number

	- key_value

		- EXPIRETIME_MS

		- ms

		- TYPE

		- key

		- value

			- INT编码

				- encoding

				- intenger

			- 字符串编码(无压缩)

				- len

				- string

			- 字符串(压缩)

				- REDIS_RDB_ENC_LZF

				- compressed_len

				- origin_len

				- compressed_string

			- REDIS_RDB_TYPE_LIST

				- list_length

				- item1

					- length

					- value

				- item2

					- length

					- value

				- ...

			- REDIS_RDB_TYPE_SET

				- set_size

				- elem1

					- length

					- value

				- elem2

					- length

					- value

				- ...

			- REDIS_RDB_TYPE_HASH

				- hash_size

				- kv1

					- key_length

					- key

					- value_length

					- value

				- kv2

					- key_length

					- key

					- value_length

					- value

				- ...

			- REDIS_RDB_TYPE_ZSET

				- sorted_set_size

				- member1

					- length

					- value

				- score1

					- length

					- value

				- member2

					- length

					- value

				- score2

					- length

					- value

				- ...

			- REDIS_RDB_TYPE_SET_INSET

				- 转为字符串后写入

			- ZIPLIST

				- 转为字符串后写入

- EOF

- check_sum

### bgsave

- rdbSaveBackground

	- 父进程

		- 记录当前bgsave相关参数

			- 记录fork消耗时间

			- 记录bgsave开始时间

			- 记录bgsave的子进程id rdb_child_pid

		- updateDictResizePolicy

			- 不允许rehash

		- pid = wait3(&statloc,WNOHANG,NULL) (serverCron中调用)

			- 非阻塞等待

		- backgroundSaveDoneHandler(serverCron中调用)

			- 成功

				- server.dirty = server.dirty - server.dirty_before_bgsave

				- 更新server.lastsace

				- 更新server.lastbgsave_status

			- 出错

				- server.lastbgsave_status = REDIS_ERR

			- 被中断

				- 删除临时文件

				- server.lastbgsave_status = REDIS_ERR

			- 统一流程

				- 更新server信息

					- server.rdb_child_pid = -1;

					- server.rdb_save_time_last = time(NULL)-server.rdb_save_time_start;

					- server.rdb_save_time_start = -1;

				- updateSlavesWaitingBgsave

					- 遍历所有slave

						- Slave复制状态是start的改为end,之前的复制终止

						- Slave复制状态是end的说明等待bgsave,进行复制

							- slave->repldbfd = open(server.rdb_filename,O_RDONLY),打开rdb文件

							- 更新偏移量,slave状态

								- slave->repldboff = 0;

								- slave->repldbsize = buf.st_size;

									- rdb文件大小

								- slave->replstate = REDIS_REPL_SEND_BULK;

							- 添加fd事件

								- aeCreateFileEvent(server.el, slave->fd, AE_WRITABLE, sendBulkToSlave, slave)

									- 添加fd到对应的监听接口,例如epoll,kqueue等

								- sendBulkToSlave发送给slave

									- 读rdb 16*1024大小

									- slave的fd写入,每次写入16*1024

									- 更新slave->repldboff

									- 如果slave->repldboff==slave->repldbsize,表示已经更新完成

										- 关闭rbd fd slave->repldbfd

										- slave->repldbfd = -1

										- aeDeleteFileEvent 删除之前的写事件

										- slave->replstate = REDIS_REPL_ONLINE 更新状态,不是更新中

										- aeCreateFileEvent 添加事件 fd:slave.fd, 方法:sendReplyToClient

											- 期间cli都是发送rdb文件,没有发送回复.这里把内容都发送过去

											- 回复放在链表中,给slave的fd写入回复直到写完

											- 循环写入,但是如果写入大小>16k,且当前内存够用.等下次fd触发再写入

												- 防止给slave发送过大的回复独占server

											- 删除写事件

	- 子进程

		- 关闭网络fd

		- rdbSave

			- 初始化io

				- r.io.file.fp

					- Fp后续用于写入

				- r.io.file.buffered

				- r.io.file.autosync

			- 写入rdb版本号

			- 遍历db,遍历db.dict

				- 每个对象遍历,根据日志文件格式存储

				- 实际调用那个函数写入//TODO

			- 写入EOF

			- 写入checksum

			- fflush

				- 清空用户区缓存

			- fsync

				- 阻塞,写入磁盘

			- fclose

			- 写入日志

			- 清理server.dirty

			- 更新server.lastsave

			- 更新server.lastbgsave_status=REDIS_OK

		- 日志记录zmalloc_get_private_dirty,因为cow导致的脏页数

		- exitFromChild,使用exit退出

### 加载

- rdbLoad

	- 加载rdb文件

	- 步骤

		- 读rdb

		- 检查版本号,不支持的gg

		- 开始的db选择,select dbid

		- 解析每个kv,根据编码情况还原o

			- 读入过期时间

				- 主机对过期key直接不写入,跳过

				- 从机对过期key写入

			- rdbLoadObject还原o

				- rdbLoadObject

					- 根据不同的格式进行加载,遍历item写入对应类型的robj

					- 如果是zset且个数少于server.zset_max_ziplist_value,zset使用ziplist

					- 如果是dict且个数少于server.hash_max_ziplist_entries,使用ziplist

			- dictAdd到server.dict

			- 版本>=5,验证crc

## aof

### aof文件格式

- *参数个数\r\n

- $参数长度\r\n

- 参数\r\n

### 加载aof

- loadAppendOnlyFile

	- server.loading

	- 读取aof

	- 创建FakeClient

	- 解析aof,使用fakeclient执行aof的命令

### Aof刷入磁盘

- flushAppendOnlyFile

	- 用于把server.aof_buf刷入到磁盘

	- 步骤

		- server.aof_buf为空,返回

		- server.aof_fsync == AOF_FSYNC_EVERYSEC

			- 确定是否有子线程在写入(因为只有这种模式是子线程写入)

			- 如果是AOF_FSYNC_EVERYSEC且不是强制写入

				- 如果已经有子线程在写入

					- server.aof_flush_postponed_start为0,记录推迟时间,跳出

					- server.aof_flush_postponed_start和现在时间相差2s内,跳出

				- 记录aof同步delay,继续操作

			- server.aof_flush_postponed_start设置为0,清空

		- 往server.aof_fd写入aof_buf

			- write结果不是buf长度

				- 返回-1,server.aof_last_write_errno = errno

				- 返回不是-1但不是buf长度,就是当前内核buff不足写入

					- ftruncate aof_fd,看下次能否写入

					- server.aof_last_write_errno = ENOSPC

				- server.aof_fsync == AOF_FSYNC_ALWAYS

					- 是

						- 打log

						- exit(1),子进程退出

					- 否

						- server.aof_last_write_status = REDIS_ERR

						- 如果write返回>0

							- server.aof_current_size += nwritten

							- 将aof_buffer把写入的剪掉

			- write成功

				- server.aof_last_write_status = REDIS_OK

				- 更新server.aof_current_size 

				- aof_buf<4000,clear.否则free

				- 如果aof_no_fsync_on_rewrite,而且当前有子进程执行aof或者rdb,直接跳出,不执行fsync

				- 判断aof_fsync

					- AOF_FSYNC_ALWAYS

						- 是:执行fsync

					- AOF_FSYNC_EVERYSEC 且距离上次超过1s

						- 当前没有同步:aof_background_fsync,另外线程处理

							- Init的时候会新建几个线程,用于处理aof等后台同步动作

					- 其他

						- 内核选择时机进行刷盘

### 写入到aof

- feedAppendOnlyFile

	- 根据cmd.proc生成对应cmd精灵

	- server.aof_buf使用sdscatlen追加

### 重写aof

- aof重写缓存(执行rewrite的时候写入缓存)

	- 作用:在gbrewriteaof的时候记录修改db的命令

		- 使用链表因为不可能一次拿出一个很大的空间.使用链表可以多次申请

	- aofRewriteBufferAppend

		- 如果当前有缓冲块,能写入多少写入多少

			- 更新block的used,free

			- 记录剩下的len

		- 如果len存在,说明当前没有block,或者block已经用完

			- 申请block添加到server.aof_rewrite_buf_blocks链表尾部

			- 每10个block打印日志,如果100倍数个block标记warning

			- 写入缓冲块,同上述步骤

	- server.aof_rewrite_buf_blocks

		- 链表

		- value是aofrwblock

			- Used,缓冲块已使用字节数

			- free,缓冲块可用字节数

			- buf,10M大小

- rewriteAppendOnlyFileBackground

	- 父进程

		- 已经在重写,REDIS_ERR

		- fork

		- fork gg,REDIS_ERR

		- 记录AOF重写信息

			- server.aof_rewrite_scheduled = 0

			- server.aof_rewrite_time_start = time(NULL)

			- server.aof_child_pid = childpid

		- 字典不允许rehash

		- server.aof_selected_db = -1

			- 强制让feedAppendOnlyFile执行调用SELECT,选择正确db

		- server.repl_scriptcache_fifo,清空脚本复制缓存

		- wait3(serverCron调用)

		- backgroundRewriteDoneHandler

			- 打开临时文件(子进程写的文件)

			- aofRewriteBufferWrite(因为rewrite的时候aof会写入到rewrite缓存中,此时需要把缓存内容写入到文件)

				- server.aof_rewrite_buf_blocks被遍历

				- 写入到临时文件中

		- 把新的aof fd设置到server_aof_fd

		- 关闭旧的aof_fd

		- server.aof_selected_db = -1

			- 强制触发SELECT db

		- 清空server.aof_buf

	- 子进程

		- 关闭网络连接 fd

		- rewriteAppendOnlyFile

			- 遍历db.dict

			- 如果已经过期,跳过

			- 根据不同类型写入aof

				- list:RPUSH

				- set:SADD

				- zset:ZADD

				- hash:HMSET

			- fflush

			- fsync

			- fclose

			- rename

			- log

		- 根据是否重写成功返回

## 事件驱动

### server.aeEventLoop

- maxfd

- setsize

- timeEventNextId

- lastTime

- aeFileEvent *events

	- 是数组,直接通过events[fd]获取

	- Int mask

	- aeFileProc *rfileProc

	- aeFileProc *wfileProc

	- Void *clientData

- aeFiredEvent *fired

- aeTimeEvent *timeEventHead

- stop

- apidata

- aeBeforeSleepProc *beforesleep

### 会aeDeleteFileEvent(从epoll拿走)的场景

- freeClient

- sendReplyToClient

- sendBulkToSlave

- updateSlavesWaitingBgsave

- replicationAbortSyncTransfer

- readSyncBulkPayload

- redisAeDelWrite

- aeDeleteFileEvent

- replicationCacheMaster

- undoConnectWithMaster

- syncWithMaster

### initServer

- 初始化server参数

- fd相关

	- aeCreateEventLoop

		- 初始化server.el

			- Fd个数:maxclients+128 (32预留 + 96)

		- aeApiCreate,初始化epoll

			- 新建epoll 1024个节点

			- 根据fd个数创建事件

	- listenToPort

		- 通过AI_PASSIVE设置socket的flag

		- 绑定ipv4和ipv6

		- Fd写到server.ipfd数组.因为ipv4和ipv6都支持,都需要监听

	- 新建unix domain

	- 对ipfd进行aeCreateFileEvent,使用accpetTcpHandler处理socketfd响应

		- Fd对应的aeFileEvent的rfileProc设置成acceptTcpHandler

			- anetTcpAccept accept客户端的请求

				- accept

				- 根据cli ip port返回

			- acceptCommonHandler 为客户端创建redisClient

				- createClient

					- 新建redisClient

					- Fd不是-1,普通客户端

						- 栓塞制非阻塞

						- 如果配置是长连接,设置长连接

							- setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &val, sizeof(val)

						- 禁用nagle算法

							- setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &val, sizeof(val)

						- 绑定为AE_READABLE事件,响应函数是readQueryFromClient

							- 读buffer,执行命令

					- 初始化redisClient各种参数

					- Fd不是-1,添加到server.clients末尾

					- 初始化客户端事务(multi)状态

				- 如果listLength(server.clients) > server.maxclients,向cli发送错误信息,并释放连接,更新拒绝连接数

				- 正常情况下更新连接数,对cli.flags设置并返回

		- epoll_ctl给epfd添加对应的fd,epoll_event.data.fd设置成监听fd,类型根据写/读对应EPOLLOUT/EPOLLIN

			- 对ee.data先对u64=0,后fd=入参fd.可以理解为对union先进行一次初始化,防止valgrind告警

	- 对sofd(unix domain)进行事件添加

- 时间事件相关

	- 初始化timeEvent相关

	- aeCreateTimeEvent

		- 添加serverCron

			- 更新lruClock

			- 如果server.shutdown_asap,准备关闭

				- (sigterm的时候信号处理器会把server.shutdown_asap设置成1)

			- 日志记录当前dict expires情况

			- 如果不是哨兵模式

				- 记录cli连接信息

			- clientsCron

				- 关闭超时客户端,释放客户端多余缓冲区

			- databasesCron

				- 主机主动清理过期key,activeExpireCycle

				- 没有gbsave以及没有aof重写

					- 尝试对dict和expires rehash

					- 允许rehash, rehash expires,dict

			- 当前没有bgsave和aof重写进程,如果有aofrewrite等待

				- rewriteAppendOnlyFileBackground

			- 如果有bgsave或者aof重写进程

				- 是

					- 非阻塞wait

					- 执行对应的回收handler

				- 否

					- 如果上次同步超过5s,server.dirty比changes要多

						- rdbSaveBackground

					- 如果aof文件体积已经比rewrite的要大server.aof_rewrite_perc

						- rewriteAppendOnlyFileBackground

						- 因此aof只要配置,都会有rewrite出现,整个db的都会被重写

			- 根据配置flushAppendOnlyFile

			- freeClientsInAsyncFreeQueue

				- 异步关闭client_closed

			- clientsArePaused

				- paushed的client如果不需要暂停,放入到unblocked_clients中

			- replicationCron

			- 如果集群

				- clusterCron

			- 如果哨兵

				- sentinelTimer

			- migrateCloseTimedoutSockets

				- 超过10s的socket过期,释放

		- 目前只有serverCron一个时间事件

- 初始化其他

	- 32位系统使用3G内存

	- 如果cluster设置,clusterInit

	- replication初始化

	- bioInit

### aeMain(死循环)

- aeProcessEvents

	- 1.搜索时间事件列表,拿到最近即将触发的事件

		- 搜索整个链表

	- 2.fd事件epoll_wait 上面拿到的时间

	- 3.处理时间事件

## 复制

### 书本内容

- 旧版本 <2.8的复制缺陷

	- Slave 发出SYNC信号之后,主机所有的rdb文件都要拉过来

- 新版本机制

	- 同步信息

		- 主机使用环状队列(1M大小)记录当前的复制积压缓存(replication backlog)

			- 环状队列内容格式

				- 偏移量

				- 字节

					- SET就会表示成S E T三个字节

		- 主机和从机使用复制偏移量表示当前的同步进度(replication offset)

		- 通过主机运行id(runid)标识机器身份

	- 同步机制

		- 从机给主机发送psync

			- 如果当前偏移在环状队列中,在环状队列返回

			- 如果偏移超过了环状队列,需要整个rdb发送

		- 从机如果断线,给主机发送psync的时候,主机根据从机偏移量进行全量或者部分同步

		- 每个机器根据启动时候有自己的runid.

			- 如果机器宕机重启,会拿回原来的runid给主机发送,表示自己身份.

			- 如果runid从新生成了,主机需要把rdb发送过去

	- PSYNC流程

		- 从机发出 PSYNC runid offset

		- 主机根据从机offset返回

			- +FULLRESYNC runid offset

				- 需要全部复制

			- +CONTINUE 

				- 部分同步

			- -ERR

				- 主机<2,8 gg

	- 复制步骤

		- 1.从机设置主机ip port

			- 客户端给从机发送 SLAVEOF ip port,表示主机的ip port

		- 2.从机向主机发出connect请求,建立socket通信

			- 主机把从机当做一个cli对待

			- 从机可以给主机发命令

		- 3.主机机向从机发送PING命令

			- 作用

				- 1.socket建立不代表通信正常.PING用于检查socket读写是否正常

				- 2.后续复制步骤需要主机正常状态才能完成.PING确认主机状态

			- 返回

				- timeout,网络gg

				- 主机报错,当前主机busy不能搞同步

				- 主机返回PONG,表示oxxk

		- 4.身份验证

			- 从机设置了masterauth

				- 主机设置requirepasss

					- 验证从机发送的密码,失败gg

				- 主机没设置requirepass

					- 过

			- 从机没设置masterauth

				- 不用验证

		- 5.发送端口消息

			- 从机发送命令 REPLCONF listening-port <port>

				- 从机发送自己的监听端口,让主机connect

		- 6.同步

			- 从机发送PSyNC

			- 主机connect从机

				- 只有这样才能把全部或者部分数据发给从机

		- 7.命令传播

			- 主机把写命令发送给从机,从机接收并执行

	- 心跳检测

		- 流程

			- 每秒一次

			- 从机向主机发送 REPLCONF ACK offset

		- 作用

			- 检查主从网络

				- 超过1s没有包,网络gg

			- 实现min-slaves

				- Min-slaves表示主机在少于多少台从机或者从机的网路延迟都大于某个阈值,不进行写命令.确保同步状态

			- 检测命令丢失

				- 定期发送,如果中间丢包,主机会补发包

				- 通过offset感知

### 代码实现

- slaveofCommand

	- 用于成为主机/从机

	- 如果是cluster,不允许使用

	- SLAVEOF NO ONE

		- 成为主机

		- replicationUnsetMaster

			- server.masterhost置空

			- freeReplicationBacklog

			- freeClient(server.master)

			- replicationDiscardCachedMaster

				- 清理master缓存

			- cancelReplicationHandshake

				- 停止下载rdb

				- undoConnectWithMaster

					- aeDeleteFileEvent

					- close fd

			- server.repl_state = REDIS_REPL_NONE

				- 到prepareClientToWrite的时候改变

	- SLAVEOF ip port

		- 成为从机

		- replicationSetMaster

			- freeClient(server.master)

				- 释放之前的master client

			- disconnectSlaves

				- 释放所有server.slaves,如果之前是主机,会有server.slaves

			- replicationDiscardCachedMaster

				- 清理master缓存

			- cancelReplicationHandshake

				- 停止下载rdb

				- undoConnectWithMaster

					- aeDeleteFileEvent

					- close fd

			- 进入链接状态

				- server.repl_state = REDIS_REPL_CONNECT

					- 进入待连接状态

					- 在replicationCron中改变

				- server.master_repl_offset = 0

				- server.repl_down_since = 0

- replicationCron

	- 每秒一次,主机从机日常操作

	- 从机

		- REDIS_REPL_CONNECTING || REDIS_REPL_RECEIVE_PONG 但是上次连接时间至今超时

			- 说明主机和从机通信gg

			- undoConnectWithMaster

		- REDIS_REPL_TRANSFER但是上次连接至今超时

			- 和主机通信gg

			- replicationAbortSyncTransfer

				- 停止rdb传输,删除临时文件

		- REDIS_REPL_CONNECTED但是上次链接至今超时

			- 和主机连接上,但是通信gg

			- freeClient(server.master)

		- REDIS_REPL_CONNECT

			- connectWithMaster

				- 非阻塞连接master,gg从来

				- Fd添加到读写监听,使用syncWithMaster处理

					- 检查和maseter连接的fd,有错误的话

						- aeDeleteFileEvent读写

					- 如果当前是REDIS_REPL_CONNECTING

						- aeDeleteFileEvent写,因为后续要读

						- server.repl_state = REDIS_REPL_RECEIVE_PONG,因为能被触发的话,肯定是主机ping了

						- 给fd写PING,发送ping,等待pong

						- 退出

					- REDIS_REPL_RECEIVE_PONG

						- aeDeleteFileEvent(server.el,fd,AE_READABLE)

						- 阻塞server.repl_syncio_timeout*1000,等待master返回pong

							- master返回错误

								- 关闭fd

								- server.repl_state = REDIS_REPL_CONNECT

								- server.repl_transfer_s = -1

							- 成功 进入验证身份流程

					- server.masterauth

						- 如果设置,给fd写入AUTH,masterauth,阻塞server.repl_syncio_timeout*1000

							- 客户端返回失败或者发送失败

								- 关闭fd

								- server.repl_state = REDIS_REPL_CONNECT

								- server.repl_transfer_s = -1

					- 从机向主机发送REPLCONF

						- 阻塞阻塞server.repl_syncio_timeout*1000,写入REPLCONF listening-port server.port

					- slaveTryPartialResynchronization

						- 根据当前是否有缓存,给主机请求PSYNC还是SYNC

							- 缓存存在,进行PSYNC master_run_id offset

							- 不存在,PSYNC ? -1

							- 阻塞等待fd返回

								- 返回FULLRESYNC

									- 记录返回的run_id到server.repl_master_runid

									- replicationDiscardCachedMaster,清出之前master的cache

									- 生成1个临时文件,fd记录在server.repl_transfer_fd 

									- 监听fd的读事件,触发readSyncBulkPayload

										- 如果server.repl_transfer_size == -1

											- 阻塞读,阻塞时间server.repl_syncio_timeout*1000,读完为止,读入到buf

											- 出错或者格式错,gg

											- 成功把buff大小写入server.repl_transfer_size,返回

											- 目标拿出buff大小,并校验buf是否正确

										- server.repl_transfer_size-server.repl_transfer_read看剩下读多少.比buf小读小点

										- read fd到buf

											- 读失败replicationAbortSyncTransfer

											- 读成功

												- 写入到server.repl_transfer_fd

												- 更新server.repl_transfer_read

												- server.repl_transfer_read > server.repl_transfer_last_fsync_off + REPL_MAX_WRITTEN_BEFORE_FSYNC(8M)

													- fsync fd,防止刷爆磁盘

													- server.repl_transfer_last_fsync_off += sync_size(server.repl_transfer_read - server.repl_transfer_last_fsync_off)

												- server.repl_transfer_read == server.repl_transfer_size(当前都读完)

													- 临时文件改名dump.rdb

													- 清空db 

														- signalFlushedDb

														- emptyDb

													- rdbLoad

													- 关闭临时文件

														- zfree(server.repl_transfer_tmpfile)

														- close(server.repl_transfer_fd);

													- 设置当前的repl_transfre_s为client,后续master直接同步

														- server.master = createClient(server.repl_transfer_s)

													- server.master->authenticated = 1

													- server.repl_state = REDIS_REPL_CONNECTED

													- 保存master的runid

														- server.master->replrunid

													- 如果主机不支持PSYNC

														- server.master->flags |= REDIS_PRE_PSYNC

													- 如果开启了持久化

														- 关闭aof

														- 重试10次startAppendOnly,开启aof.目的是强制使用新db的aof文件

									- 更新统计信息

								- 返回CONTINUE

									- replicationResurrectCachedMaster,将缓存中的master设置为当前master

										- 监听master读

										- 如果需要写,监听master写

									- 返回

								- 返回ERR

									- 关闭fd

									- server.repl_state = REDIS_REPL_CONNECT

									- server.repl_transfer_s = -1

				- server.repl_transfer_lastio = server.unixtime

				- server.repl_transfer_s = fd;

				- server.repl_state = REDIS_REPL_CONNECTING

		- 如果主机版本>2.8

			- replicationSendAck

				- REPLCONF ACK offset

	- 主机

		- 看当cronloops是不是满足ping一次的次数

			- (server.cronloops % (server.repl_ping_slave_period * server.hz)

				- Ping周期时间*serer次/秒,表示一个ping周期需要多少个server的次数

			- 如果上述不满足,就是%结果是0,所以达到了一个ping的周期.

				- replicationFeedSlaves

					- 给所有slaves 状态为ONLINE的发送PING

			- 如果slave当前是REDIS_REPL_WAIT_BGSAVE_START或者REDIS_REPL_WAIT_BGSAVE_END

				- 直接write “\n”

				- 防止上面ping执行时间较慢,从机断开

		- 断开超时的从机

			- 遍历server.slaves

				- 跳过不是REDIS_REPL_ONLINE的机器

				- 旧版本从机跳过

				- 如果当前时间比从机之前的repl_ack_time大于server.repl_timeout,freeClient

		- 没有从机,server.repl_backlog_time_limit

			- 释放backlog

		- 没有从机,aof已关闭

			- 清空脚本

		- refreshGoodSlavesCount

			- Min-slaves 或者max-leg选项开启

				- 统计goodslaves数量,符合online且延迟小于repl_min_slaves_max_lag算符合

				- 记录到server.repl_good_slaves_count

- syncCommand

	- 主机收到sync命令,给从机同步

	- 步骤

		- c->flags & REDIS_SLAVE,SLAVE跳过

		-  如果本机是从机而且server.repl_state != REDIS_REPL_CONNECTED

			- 从机不能乱搞

		- C.reply或者c.bufpos,此时客户端还有数据等待输出,不能执行

		- 如果是psync

			- 是

				- masterTryPartialResynchronization

					- 如果命令的runid和server.runid不一致

						- fullresync

							- 发送 FULLRESYNC server.runid server.master_repl_offset

							- 异步释放cli

							- 返回REDIS_ERR

					- 入参offset不合法

						- fullresync

							- 发送 FULLRESYNC server.runid server.master_repl_offset

							- 异步释放cli

							- 返回REDIS_ERR

					- server没有backlog,或者入参offset比backlog要小,或者offset比backlog+历史长度还要打

						- 入参offset超出能支持的范围

						- fullresync

							- 发送 FULLRESYNC server.runid server.master_repl_offset

							- 异步释放cli

							- 返回REDIS_ERR

					- 进行psync

						- 把cli添加到server.slave

						- 对c.fd写入+CONTINUE\r\n

						- addReplyReplicationBacklog

							- repl_backlog_histlen==0

								- 之前没有写过psync,return 0

							- 计算同步的位置进行统计

								- repl_backlog_off 表示不需要放在backlog的位置

								- repl_backlog_idx 当前backlog的idx

								- repl_backlog_size 环形数组的长度

								- repl_backlog_histlen 表示repl_backlog_off到现在最新的backlog的距离

								- <off><histlen>表示至今的总长度

								- j=(server.repl_backlog_idx +  
								          (server.repl_backlog_size-server.repl_backlog_histlen)) %  
								          server.repl_backlog_size 表示backlog_off在环形队列的idx

									- 就是idx-histlen这个大小在环形队列的位置

								- skip=offset - server.repl_backlog_off offset是客户端传入当前的offset.因为off已经不存在.这个长度表示可以从off开始跳过多少开始算

								- j=(j+skip)%size,表示当前的offset在环形队列的位置

								- 环形队列同步可能涉及头尾问题.到了尾部需要切换到头部,发送给客户端

							- 返回同步的长度

						- refreshGoodSlavesCount

						- 退出

			- 否

				- 不是psync,c->flags |= REDIS_PRE_PSYNC 避免接收replconf ack

		- 当前正在执行bgsave

			- 是

				- 遍历server.slaves

					- 有REDIS_REPL_WAIT_BGSAVE_END,跳出

					- 当前有slave是bgsave完的

						- copyClientOutputBuffer(c,slave)

							- 把有bgsave的buffer复制给当前的从机客户端

						- c->replstate = REDIS_REPL_WAIT_BGSAVE_END

					- 当前全部都没有bgsave完

						- c->replstate = REDIS_REPL_WAIT_BGSAVE_START 

						- 等下个bgsave

			- 否

				- 触发rdbSaveBackground

				- c->replstate = REDIS_REPL_WAIT_BGSAVE_END

				- replicationScriptCacheFlush

		- c->repldbfd = -1

		- server.slaveseldb = -1;

		- listAddNodeTail(server.slaves,c)

		- 如果当前是第一个slave

			- createReplicationBacklog

				- backlog为0,触发全同步

## 哨兵

### 书

- 作用:高可用.同时管理主机和从机,防止主机gg的时候不及时切换

- 启动/初始化流程

	- 启动命令使用redis-sentinel

	- 初始化服务器

		- Db操作相关不能用

			- SET

			- DEL

			- ...

		- 事务不能用

		- 脚本不能用

		- aof rdb不能用

		- slaveof内部使用

		- PUBLISH可用

		- fd事件可用

		- 时间事件可用

	- 使用哨兵逻辑

	- 初始化哨兵状态

		- sentinelState

	- 初始化sentinelState的masters

		- 字典,内容是sentinelRedisInstance

		- sentinelRedisInstance

			- flags

			- name

			- runid

			- config_epoch

			- sentinelAddr

			- down_after_period

			- quorum

			- ...

	- 创建面向主服务器的网络连接

		- 命令连接,cli

		- 订阅连接,专门订阅主服务器的__sentinel__:hello频道

			- 通过订阅连接收到其他哨兵的消息,因为订阅是一个哨兵发送,所有哨兵都收到

			- 通过命令连接发送,订阅连接接收

- 获取主服务器信息

	- 10s一次,向主服务器发送INFO

	- 主服务器返回

		- 自己信息

			- role

			- runid

		- slave信息

			- ip 

			- port

			- role

			- runid

	- 哨兵更新主从机器的信息

- 获取从服务器信息

	- 获取主机信息后有从机信息

	- 创建订阅,命令连接

- 向主机和从机发送信息

	- 2s一次,给所有主从机通过命令连接发送PUBLISH

	- 内容:s_ip s_port s_runid s_epoch m_name m_ip m_port m_epoch

		- 哨兵自己的信息以及主机的信息

- 接收来自主机从机的频道信息

	- 哨兵和主从机建立订阅连接后,订阅频道__sentinel__:hello

	- 直到和服务器断开,连接都维持

	- 哨兵可以通过给各个服务器publish使得各个其他哨兵收到本哨兵的消息

	- 从哨兵根据订阅信息及时更新

		- 更新哨兵字典

			- key:[ip:port]

			- value:sentinelRedisInstance

		- 通过订阅能获取到其他哨兵的信息

			- 已存在:更新

			- 新哨兵:新增

			- 自己的丢弃

	- 创建连接其他哨兵的命令连接

		- 每个哨兵之间相互连接

		- 用于信息交换,选主

		- 哨兵之间不需要订阅连接

			- 订阅连接使用订阅的方式,一个哨兵的信息可以所有哨兵都收到

- 检测主观下线

	- 哨兵每秒一次给主从机,其他哨兵发送ping,根据ping返回判断是否在线

		- 规定时间内有效返回

			- +PING

			- -LOADING

			- -MASTERDOWN

		- 其他都无效

	- 对比机器的down-after-milliseconds配置

		- 超过的话,机器的flag打上SRI_S_DOWN

		- 主从机和其他哨兵都是用同一个配置

	- 不同的哨兵主观下线配置可能不同,每个哨兵根据自己的配置进行判断

- 检查客观下线状态

	- 哨兵发发现一个主机主观下线后为了确认,向同样监视这个主机的其他哨兵询问,看是不是进入下线状态(主观客观都可以),足够数量的下线判定可以判定为客观下线,进行故障转移

	- 向其他哨兵发送SENTINEL is-master-down-by-addr <ip> <port> <current_epoch> <runid>

		- 询问其他哨兵这个主机是不是下线了

		- current_epoch是当前哨兵的配置时期

		- runid是哨兵自己的runid

			- *表示检查主机客观下线状态

			- 写入runid表示要选举头哨兵

	- 收到问询后回复

		- 根据问询的ip port检查主机是否下线

			- //TODO 怎么检查?

		- 返回

			- down_state

				- 1 下线

				- 0 未下线

			- leader_runid

				- * 仅仅检查是否下线

				- 具体runid:当前认为的头哨兵runid

			- leader_epoch

				- 上面参数是*,是0

				- 当前认为的头哨兵的epoach

	- 接收回复后

		- 统计个数

		- 超过哨兵中的quorum数量后,认为主机已经客观下线

			- 不同哨兵的客观下线判断标准可能不同,因为quorum不同

- 选举头哨兵

	- 主机客观下线之后,各个哨兵会协商选出头哨兵,对下线主机进行故障转移

	- 规则(类似raft)

		- 所有成员公平

			- 所有在线哨兵都有资格成为头哨兵

		- 需要同一个epoch内,先到先得

			- 每次选举后所有哨兵epoch+1

			- 每个epoch都有一次将某个哨兵设置为局部头哨兵的机会.只要设置了,不能更改

			- 每个发现主机客观下线的哨兵都要求其他哨兵将自己设置成局部头哨兵

			- A给B发is-master-down-by-addr,如果runid设置A的runid了,要求B把A设置为局部头哨兵

			- 局部头哨兵先到先得,被设置过的不能再设置

			- 回复is-master-down-by-addr中的runid表示当前哨兵认定的局部头哨兵

			- 收到回复的哨兵检查epoch是否和自己的一致.一致的话取出回复中的runid,和自己一致表示被选了

		- 超过半数成功

			- 某个哨兵被一半以上哨兵设置成局部头哨兵,成为头哨兵

		- 限时gg从来

			- 规定时间内没有一个哨兵被选成功,每个哨兵间隔一段时间后再选举

- 故障转移

	- 选出主机

		- 规则

			- 1.删除所有下线或者断线的从机

			- 2.删除最近5s没有回复头哨INFO的从机,确保最近成功通信

			- 3.删除所有和之前主机连接段考超过down-after-milliseconds*10 ms的从机,确保从机和主机断开时间较短,数据新鲜

			- 4.根据从机优先级排序,找优先级最高的

				- //TODO 优先级怎么来

			- 5.优先级一样高,看从机复制offset,找出最高的,表示同步进度最新

			- 6.连offset都一样,就看id最小的

		- 选出后,头哨10s一次给新主机发送INFO,看新主机role变化,成为master就ok

	- 修改从机复制目标

		- 老主机的从机成为新主机的从机

		- 给从机下发slaveoff 新主机

	- 旧主机成为新主机的从机

		- SLAVEOFF 新主机

### 代码

- 结构体

	- sentinelState

		- current_epoch

		- dict *masters

			- key:主机名字

			- value:sentinelRedisInstance*

				- 公共信息

					- int flags

					- name

					- runid

					- config_epoch

					- addr

					- int role_reported

						- 角色

					- role_reported_time

						- 更新时间

					- down_after_period

					- last_ping_time

					- last_pong_time

				- 主机特有

					- dict *slaves

						- 主机的从机

							- sentinelRedisInstance

					- dict *sentinels

						- 监控这个主机的哨兵

							- sentinelRedisInstance

					- int quorum

						- 客观下线需要支持的票数

					- parallel_syncs

						- 故障转移时候可以同时对新主进行的从机数量

					- auth_pass

						- 主从同步的密码

				- 从机特有

					- master_link_down_time

						- 主断开时间

					- slave_priority

						- 优先级

					- slave_reconf_sent_time

						- 故障转移时从服务器发送slaveof的时间

					- struct sentinelRedisInstance *master

						- 主机

					- char *slave_master_host

						- INFO回复的主机ip

					- int slave_master_port

						- INFO回复的端口

					- int slave_master_link_status

						- INFO命令恢复的主从连接状态

				- 故障转移使用

					- leader

						- 哨兵的头哨

					- leader_epoch

						- 头哨的epoch

					- failover_state

						- 当前故障转移的状态

							- SENTINEL_FAILOVER_STATE_NONE

								- 没有转移

							- SENTINEL_FAILOVER_STATE_WAIT_START

								- 开始转移

							- SENTINEL_FAILOVER_STATE_SELECT_SLAVE

								- 选新主

							- SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE

								- 新主发SLAVEOF ON ONE

							- SENTINEL_FAILOVER_STATE_WAIT_PROMOTION

								- 等待,INFO新观察成新主

							- SENTINEL_FAILOVER_STATE_RECONF_SLAVES

								- SLAVEOF 其他从

							- SENTINEL_FAILOVER_STATE_UPDATE_CONFIG

								- 监视新主

					- failover_state_change_time

						- 状态改变的时间

					- failover_start_time

						- 最后一次故障时间

					- failover_timeout

						- 故障迁移最大时间限制

					- failover_delay_logged

						- 迁移延迟

					- struct sentinelRedisInstance *promoted_slave

						- 要升的从机

					- ...

		- int tilt

		- int running_scripts

		- mstime_t tilt_start_time;

		- mstime_t previous_time

		- list *scripts_queue

- serverCron

	- sentinelTimer

		- sentinelCheckTiltCondition

			- 判断是否进入TITL模式

				- 通常两次哨兵之间差在100ms左右

				- 时间差<0或者>2s要么是进程因为一些原因阻塞,要么系统时钟出现变化

					- 上述情况哨兵可能掉线,无法判定是不是真的失效

					- 进入tilt模式,让哨兵等待SENTINEL_TILE_PERIOD秒,不做任何操作,只搜集信息

			- 步骤

				- 求出当前时间和上次哨兵时间的差

				- 小于0或者超过2s

					- sentinel.tilt=1

					- sentinel.tilt_start_time=mstime()

		- sentinelHandleDictOfRedisInstances(masters)

			- 日常定期操作

			- 步骤

				- sentinelHandleRedisInstance

					- 步骤

						- sentinelReconnectInstance

							- 创建网络连接

							- 步骤

								- 不是SRI_DISCONNECTED,返回

								- sentinelRedisInstance.cc为空

									- redisAsyncConnect

										- 非同步创建连接

								- cc存在,connet起码是EHOSTUNREACH

									- redisAeAttach

										- 初始化cc

										- cc.ev.data初始化

											- Cc.data初始化

												- Data就是新建的redisAeEvents

												- context=cc

												- fd=cc.c.fd

												- reading=writing=0

												- loop=server.el

											- cc.ev.addRead=redisAeAddRead

												- aeCreateFileEvent,添加读事件

													- redisAeReadEvent处理

														- 读入放在cc.reader.buf中

											- ac->ev.delRead = redisAeDelRead

											- ac->ev.addWrite = redisAeAddWrite

											- ac->ev.delWrite = redisAeDelWrite

											- ac->ev.cleanup = redisAeCleanup

									- redisAsyncSetConnectCallback

										- ac->onConnect=sentinelLinkEstablishedCallback

											- 如果cc.data==pc

												- sentinelEvent +pubsub-link

													- pc publish client

											- 如果不是

												- sentinelEvent +cmd-link

													- cc cmd client

									- redisAsyncSetDisconnectCallback

										- 断线回调 sentinelDisconnectCallback

									- sentinelSendAuthIfNeeded

										- 主机使用ri.auth_pass 从机使用ri.master.auth_pass

										- 密码吗设置了,异步AHTH 到cc

									- sentinelSetClientName

										- 对cc设置名字,用server.runid设置

									- sentinelSendPing

										- PING cc

						- sentinelSendPeriodicCommands

							- 根据情况发送PING INFO 或者PUBLISH

							- 步骤

								- SRI_DISCONNECTED 返回

								- 待发送命令>=SENTINEL_MAX_PENDING_COMMANDS 返回

								- last_pong_period取down_after_period和1s最大值

								- 从机的主机SRI_O_DOWN或者SRI_FAILOVER_IN_PROGRESS

									- info_period为1s

										- 为加快捕捉master情况

									- 否则10s

								- 自己不是哨兵,超过info_period,发送INFO

								- last_pong_time至今超过ping_period 1s,发送PING

								- last_pub_time至今超过2s,发送PUBLISH

									- sentinelSendHello

										- 自己是master, 发自己的ip port信息  不是的话发master的ip port信息

										- 获取实例自己的信息

										- 异步 publish

						- 如果tilt模式,在TILT_PERIOD时间内直接返回

							- 否则,tilt模式作废,tilt=0

						- sentinelCheckSubjectivelyDown

							- 检测是否主观下线

								- 看cc pc连接是否要kill

								- 看ping时间是否过大

							- 步骤

								- cc

									- 上次链接时间距今>1.5s && 之前ping过 && 上次ping时间距今>down_after_period/2 && 上次pong时间距今>down_after_period/2

										- sentinelKillLink

								- pc

									- 上次连接时间距今>1.5s && pc_last_activity距今>0.6ms

										- sentinelKillLink

								- 更新SRI_S_DOWN

									- 上次ping距今>down_after_period  或者 (标记是主机但是报告是从机,上次role_reported_time比down_after_period+ 2个INFO_PERIOD都大)

										- 是

											- 如果不是SRI_S_DOWN

												- s_down_since_time更新为当前时间

												- ri->flags |= SRI_S_DOWN

										- 否

											- 说明之前ping时间在限期内  或者  机器正常

											- 如果已经是SRI_S_DOWN

												- 移除SRI_S_DOWN 和 SCRIPT_KILL_SENT

									- SDOWN是subjectively down,主观下线

						- 如果是主机

							- sentinelCheckObjectivelyDown

								- 看是否客观下线

									- 只有本哨兵主观发现某个主机主观下线,才能开始判定是否客观下线

								- 步骤

									- 如果当前哨兵认为SRI_S_DOWN,统计+1(当前哨兵判定主观下线才能开始)

										- 遍历所有master的sentinel,如果SRI_MASTER_DOWN,统计++

										- 统计>quorum,断定客观下线

									- 判断主观下线,打标,发送事件

							- 如果sentinelStartFailoverIfNeeded

								- 看是否需要故障转移

									- 如果需要故障转移的话会在epoach频道和failover频道发布需要转移的情况

								- 步骤

									- 不是O_DOWN,跳过

									- SRI_FAILOVER_IN_PROGRESS,跳过

									- 故障开始时间至今>两个failover_timeout

										- master->failover_delay_logged != master->failover_start_time

											- master->failover_delay_logged = master->failover_start_time,跳过

									- sentinelStartFailover  

										- 初始化故障转移

										- 步骤

											- master->failover_state = SENTINEL_FAILOVER_STATE_WAIT_START

											- master->failover_epoch = ++sentinel.current_epoch

											- sentinelEvent 在频道+new-epoch,publish 新epoch

											- sentinelEvent 在频道+try-failover,尝试故障转移

								- 需要故障转移的话,sentinelAskMasterStateToOtherSentinels

									- 认为下线会给这个master的其他哨兵发is-master-down-by-addr

									- 步骤

										- 遍历master的哨兵

											- 哨兵last_master_down_reply_time至今超过5个SENTINEL_ASK_PERIOD(一共0.5s)

												- 说明这个哨兵很久没有更新信息,信息可能不准确

												- 去掉这个哨兵的SRI_MASTER_DOWN信息

												- 去掉这个哨兵的leader信息

											- 本机认为这个master SRI_S_DOWN,跳过

											- 哨兵SRI_DISCONNECTED,跳过

											- 哨兵不是SENTINEL_ASK_FORCED而且last_master_down_reply_time至今小于SENTINEL_ASK_PERIOD,跳过

											- 能到这里的,说明哨兵连接正常,而且最近是有连接的

												- 异步发送命令SENTINEL is-master-down-by-addr

													- 回调的时候记录哨兵的投票

							- sentinelFailoverStateMachine

								- 故障转移流程

								- 步骤

									- flag不是SRI_FAILOVER_IN_PROGRESS,跳过

									- 根据failover_state做转移(切换了状态之后才做对应的操作,不是操作完之后进入对应状态)

										- SENTINEL_FAILOVER_STATE_WAIT_START

											- sentinelFailoverWaitStart

												- 准备进行故障转移

													- 异常:如果主哨选出来的主机很久没有更新,清理主哨选出的信息,退回到SENTINEL_FAILOVER_STATE_NONE

												- 步骤

													- 选主哨 sentinelGetLeader(ri, ri->failover_epoch)

														- 选主

														- 步骤

															- 遍历机器的哨兵

																- 使用runid做key,投票做value,统计各个哨兵选出的主哨

															- 遍历用于统计的字典

																- 找出投票最多的哨兵runid

															- 如果当前有选出的主哨

																- 是

																	- 本哨投当前的主哨sentinelVoteLeader

																		- 如果请求epoch比当前epoch要大

																			- +new-epoch

																		- 如果leader epoch比请求的epoch小,或者当前的epoch<=请求epoch

																			- 发布+vote-for-leader

																- 当前没投票

																	- 本哨投自己

																		- 同上

															- 投票后如果超过一半以上(主机的哨兵就是总数),选举成功,返回runid

															- 否则没选成功,返回null

													- 当前没有主哨,这次超过故障转移时长和选举时长的最大值(说明本次选主哨gg,重来)

														- 发布 -failover-abort-not-elected

														- 取消本次故障转移  sentinelAbortFailover

															- 只能是被选中的从机升主机中间调用

															- 步骤

																- flags移除SRI_FAILOVER_IN_PROGRESS|SRI_FORCE_FAILOVER

																-  ri->failover_state = SENTINEL_FAILOVER_STATE_NONE

																- 如果主哨有待升的从机

																	- 清空待升的从机

																	- ri->promoted_slave->flags &= ~SRI_PROMOTED

																	- ri->promoted_slave = NULL;

													- 发布+elected-leader

													- 进入选从机状态

														- ri->failover_state = SENTINEL_FAILOVER_STATE_SELECT_SLAVE

													- 发布+failover-state-select-slave

										- SENTINEL_FAILOVER_STATE_SELECT_SLAVE

											- sentinelFailoverSelectSlave

												- 从主机适合的从机中找出适合的机器做主机

												- 步骤

													- sentinelSelectSlave

														- 物色好slave

														- 步骤

															- 遍历主机的slave

																- 不要SRI_S_DOWN|SRI_O_DOWN|SRI_DISCONNECTED

																- 不要last_avail_time至今>0.5s

																- slave_priority==0不要

																- info_refresh超过合法时间,不要

																	- SDOWN 5个ping-period

																	- 其他 3个info period

																- 排除后剩下的从机中选

																	- 优先级小的先拿

																	- 优先级用于,偏移量大的先拿

																	- 偏移量一样,runid小的先拿

													- 没有适合的slave

														- 是

															- -failover-abort-no-good-slave

															- sentinelAbortFailover

														- 否

															- 打标新的slave,进入下一个阶段

															- +selected-slave

															- slave->flags |= SRI_PROMOTED

															- ri->promoted_slave = slave

															- ri->failover_state = SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE

															- ri->failover_state_change_time = mstime(

															- +failover-state-send-slaveof-noone

										- SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE

											- sentinelFailoverSendSlaveOfNoOne

												- 选中的从机升主

												- 步骤

													- ri->promoted_slave->flags & SRI_DISCONNECTED

														- failover_state_change_time至今超过failover_timeout

															- -failover-abort-slave-timeout

															- sentinelAbortFailover

													- sentinelSendSlaveOf

														- 给从机slaveof no one

														- 步骤

															- 没有host参数,SLAVEOF NO ONE

															- redisAsyncCommand SLAVEOF ip port

															- ri->pending_commands++

															- redisAsyncCommand CONFIG REWRITE

															- ri->pending_commands++

													- sentinelSendSlaveOf失败 返回

													- sentinelSendSlaveOf成功

														- +failover-state-wait-promotion

														- ri->failover_state = SENTINEL_FAILOVER_STATE_WAIT_PROMOTION

														- 更新ri->failover_state_change_time

										- SENTINEL_FAILOVER_STATE_WAIT_PROMOTION

											- sentinelFailoverWaitPromotion

												- 等待升级生效,超时重选

												- 步骤

													- 如果超时

														- -failover-abort-slave-timeout

														- sentinelAbortFailover

										- SENTINEL_FAILOVER_STATE_RECONF_SLAVES

											- sentinelFailoverReconfNextSlave

												- 让其他的slave都slave of 新主

													- 如果sentinelRefreshInstanceInfo成功看到从机升级成主机,状态会改成RECONF_SLAVES

												- 步骤

													- 遍历master的slave

														- SRI_RECONF_SENT|SRI_RECONF_INPROG,in_progress++

													- 如果in_progress<parallel_syncs,遍历slaves

														- 如果SRI_PROMOTED|SRI_RECONF_DONE,说明是新主或者已配置,跳过

														- SRI_RECONF_SENT且超时

															- -slave-reconf-sent-timeout

															- 去掉SRI_RECONF_SENT,加上SRI_RECONF_DONE

														- 如果SRI_DISCONNECTED|SRI_RECONF_SENT|SRI_RECONF_INPROG

															- 跳过

														- sentinelSendSlaveOf

															- slaveof 新主

														- slaveof成功

															- 加上SRI_RECONF_SENT

															- 更新slave_reconf_sent_time

															- +slave-reconf-sent

															- in_progress++

													- sentinelFailoverDetectEnd

														- 判断所有从机是否同步完成

															- 超时重新slaveof

															- 都成功,切换状态到SENTINEL_FAILOVER_STATE_UPDATE_CONFIG

														- 步骤

															- 遍历slave

																- 下线跳过

																- SRI_PROMOTED|SRI_RECONF_DONE跳过

																- 统计未reconfig的

															- 当前超时,+failover-end-for-timeout

															- 没有要reconfig的

																- +failover-end

																- master->failover_state = SENTINEL_FAILOVER_STATE_UPDATE_CONFIG

																- 更新master->failover_state_change_time

															- 如果超时

																- 遍历slave

																	- 跳过SRI_RECONF_DONE|SRI_RECONF_SENT|SRI_DISCONNECTED

																	- slaveof 新主

																		- 成功

																			- +slave-reconf-sent-be

																			- slave->flags |= SRI_RECONF_SENT

							- sentinelAskMasterStateToOtherSentinels

				- 当前是主机

					- 遍历主机的从机

						- sentinelHandleDictOfRedisInstances

					- 遍历主机的哨兵

						- sentinelHandleDictOfRedisInstances

					- 如果这个主机SENTINEL_FAILOVER_STATE_UPDATE_CONFIG

						- 故障迁移完成,sentinelFailoverSwitchToPromotedSlave

							- 旧master切换成新主

							- 步骤

								- 如果有prompted_slave,使用新主切换.否则用老主

								- +switch-master

								- sentinelResetMasterAndChangeAddress

									- 新主代替老主

									- 步骤

										- 遍历slave

											- 跳过新主

											- 记录slave信心

										- 重置master,清空内容

										- 把旧的slave新加到清空的master

										- 把新主ip port写入重置后的master

		- sentinelRunPendingScripts

		- sentinelCollectTerminatedScripts

		- sentinelKillTimedoutScripts

		- 更新server hz

## 集群

### 书

- 节点

	- 每个redis实例是一个节点

	- 通过CLUSTER MEET ip port拉入集群

		- Ip是集群节点的ip,port是集群节点的port

		- Cli给服务发出后,希望这个节点加入到指定ip port节点的集群中

	- 启动节点

		- 配置cluster_enabled配置是yes,开机集群模式节点

			- 和单机相同

				- 集群模式fd事件,时间事件不变

				- 继续rdb aof

				- 订阅,发布

				- 脚本

			- 和单机不同

				- 集群模式mget会慢很多

				- 集群模式不支持multi

				- 集群模式时间事件会多clustreCron,增加集群的常规操作

					- 给其他节点发送gossip消息

					- 检查节点是否断线

					- 是否需要对下线节点自动故障转移

		- 否则,普通单机

	- 记录其他以及自己节点

		- 使用clusterNode记录节点

			- 创建时间

			- 节点名字

			- flag

			- epoch

			- ip

			- port

			- clusterLink

				- 创建时间

				- fd

				- 输入buf

				- 输出buf

			- ...

		- 使用clusterState记录集群状态

			- 当前节点

			- 集群当前epoch

			- 集群当前状态

				- 在线/下线

			- 当前集群处理的槽节点数量

			- 集群节点名单

				- dict,内容是clusterNode

	- CLUSTER MEET

		- 场景:A收到MEET ipB portB

		- 流程

			- 1.A给B新建clusterNode,添加到clusterState.nodes中

			- 2.A根据ipB,portB给B发MEET消息

			- 3.正常情况B收到A的MEET,B给A创建一个clusterNode,添加到B的clusterState.node

			- 4.B向A发一条PONG

			- 5.A收到PONG,表示A知道B收到了信息

			- 6.A给B发PING,表示A收到了

		- 类似TCP三次握手

		- 之后节点A会将节点B信息通过gossip协议传播到其他节点,让其他节点和B握手

			- //TODO 这里应该是B把A传播.因为A请求加入B的集群,此时应该是B已经有了其他节点的信息A没有

- slot分配

	- 一个集群总是16384个slot

		- 16K

		- 所有slot都被处理才算集群上线,否则下线

		- 使用CLUSTER ADDSLOTS xxx给指定的节点分配slot

	- 节点记录slot信息

		- clusterNode.slots[16384/8]

			- Bitmap表示当前那个slot属于当前节点,1是被分配

		- clusterNode.numslots

			- slot数量

	- 节点slot信息同步

		- 通过消息告知其他节点自己slot信息

			- 其他节点对改节点进行更新

	- 记录所有slot分配信息

		- clusterState.slots[16384]

			- 内容是clusterNode指针

			- 数组存储slot对应的node

			- 如果是NULL,表示未指定派给任何节点

				- //TODO 如果冲突会怎样?

			- 对比clusterNode.slots

				- 方便查询某个slot属于哪个node

				- 当某个节点需要把自己的slot信息发给其他节点,使用clusterNode.slots更方便

					- 不需要遍历clusterState.slots

	- CLUSTER ADDSLOTS

		- clusterState.slots[i]被占据,报错

		- 更新clusterState.slots[i] 指向对应自己node

		- 更新对应node的clusterNode.slots

		- 发送消息告知其他node自己的slots

- 执行命令

	- 所有slot分配后上线

		- 接收指令,如果key所在slot属于自己,执行

		- 如果不属于自己,把slot对应node ip port取出后,给cli返回MOVED ipNode ipPort

	- key的slot算法

		- CRC(key)&0x3FFF

	- MOVED

		- Cli会封装,用户不感知

		- cli收到MOVED后定向给新的ip port,重新执行命令

	- 节点db

		- 过期等日常特性一致

		- 只能使用0号db,不能用其他db

			- 和单机有区别

			- //TODO 为什么不能用其他db

		- 使用clusterState.slots_to_keys

			- 跳表

			- score:slot value:key

			- 节点往db添加新kv,clusterState.slots_to_keys会写入

			- 节点从db删除某kv,clusterState.slots_to_key被删除

- slot重新分配

	- 通过redis-trib执行

	- 步骤

		- 1.给目标node发送 CLUSTER SETSLOT <slot> IMPORTING <source_id>

			- 让目标node准备impot

		- 2.给源node发送CLUSTER SETSLOT <slot> MIGRATING <target_id>

			- 让源node准备migrat

		- 3.给源node发送CLUSTER GETKEYSINSLOT <slot> <count>

			- 让源node返回slot中最多count个key name

		- 4.对步骤3每个key都给源发送一个MIGRATE <target_ip> <target_port> <key_name> 0 <time_out>

			- 将备选中key原子地从源迁移到目标node

		- 5.复制3 4 到完成

		- 6.向集群任意一个节点发CLUSTER SETSLOT <slot> NODE <target_id>

			- 迁移信息发给整个集群

			- 所有节点都收到,更改节点中的slot配置

- ASK

	- Slot迁移中,如果源节点某个key已经迁移了但是访问到源节点

		- 返回ASK ip port,指向导入节点

		- 类似MOVED,ASK会被隐藏,cli自行实现了重定向.

	- CLUSTER SETSLOT IMPORTING

		- clusterState.importing_slots_from[16384]

			- 记录clusterNode节点

			- 当前节点从clusterNode导入

			- 命令执行后,对应的slot位置被更改

	- CLUSTER SETSLOT MIGRATING

		- clusterState.migrating_slots_to[16384]

			- 记录clusterNode节点

			- 记录当前节点要导入到该节点

			- 非空,表示当前对应slot给迁移

	- ASK错误

		- 迁移走的key 会被删除.迁移中的时候如果已经迁走了,发送ASK,引导客户端找正确的

		- 返回: ASK <slot> ip port

	- ASKING

		- 客户端问新ip之前先发送ASKING,表示必须在slot中查找

		- 如果不发ASKING,因为当前还没导入完成,这个slot还不真正属于当前节点,所以会发送MOVED,让cli找之前的节点

		- 步骤

			- Slot给了本节点?

				- 是,执行

			- 否:是否正在导入slot?

				- 是:是否有ASKING?

					- 是:执行

					- 否:MOVED

				- 否:MOVED

	- ASK和MOVED区别

		- MOVED

			- 明确slot不在本节点

		- ASK

			- slot在本节点,但是迁走了

- 复制&故障迁移

	- 复制

		- 集群中可以有主节点/从节点

			- 主节点gg,从节点通过选举出个主节点,顶上

			- 原来从节点要认新的主节点

		- 设置从节点

			- CLUSTER REPLICATE <node_id>

			- 让节点成为从节点

			- clusterNode.slaveof

				- 记录clusterNode指针

				- 让指针指向slaveof的主机节点

			- clusterState.myself.flags

				- 从REDIS_NODE_MASTER改成REDIS_NODE_SLAVE

			- 调用复制代码,相当于执行了slaveof,等待下次replicationCron进行复制

			- 成为从节点,开始复制某个主节点这些信息会通过消息发送给集群其他节点

			- 所有节点都会在主节点的clusterNode.slaves和clusterNode.numslaves更新属性

	- 故障

		- 故障检测

			- 每个几点定期向集群中其他节点发送PING

				- 确认在线

				- 没有定期返回,意思在线,flags|=PFAIL

			- 每个几点通过护法消息交换集群中各个节点的状态信息

				- PFAIL

				- FAIL

			- 主A从主B得知主C疑似下线,主A在自己的clusterState.node中找到主C对应的clusterNode,把主B的报告写入

				- clusterStateA.clusterNodeC.fail_reports.addclusterNodeFailReportB

			- 半数以上主节点将x主节点报告为疑似下线,N节点发觉后把x标记为FAIL,并且广播x的FAIL消息(给所有节点)

		- 故障迁移

			- 1.选中其中一个从节点

			- 2.被选中从节点SLAVEOF no one

			- 3.新主撤销所有对久主的slot,把slot指向自己

				- 可以看出,从机自己是没有slot的

			- 4.新主向集群广播PONG,表示成为新主,接管旧主的slot

			- 5.新主接受处理slot相关命令请求.故障转移完成

		- 选举新主

			- 1.集群epoch自增,开始为0

			- 2.故障转移开始,epoch++

			- 3.每个epoch,每个主node有一次投票机会,第一个想主节点要求投票的从节点得到该主节点投票

			- 4.从节点发现自己的主已经下线,广播CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST,要求所有收到消息的主给自己投票

			- 5.如果一个主还没投给别的从,返回CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK,投票给这个从

			- 6.每个参与的从都会收到CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK,根据收到多少条统计

			- 7.超过半数主投票了,成为新主

			- 8.一个epoch里选不出来,进入下一轮选举,选到有为止

- 消息

	- 各个节点通过消息通信

		- 不像主从机用cli通信

		- 消息由消息头和正文组成

	- 消息类型

		- MEET

			- 发送MEET的请求加入集群

		- PING

			- 每秒从节点列表随机选5个,选出最长时间没有PING过的PING,检测是否在线

			- 如果最后一次收到PONG的节点时间距离当前超过了节点配置的cluster-node-timeout一半,给这个节点PING,防止对方长时间没有收到自己的PING,信息之后

		- PONG

			- MEET或者PING的返回

			- 可以主动广播PONG,更新其他节点对自己的认识

		- FAIL

			- 主A发现主B进入FAIL,广播FAIL.收到FIAL的节点标记B下线

		- PUBLISH

			- 节点收到PUBLISH命令,会执行,并给集群广播PUBLISH消息

	- 消息头

		- count

			- gossip协议使用

		- epoch

		- sender

			- 发送者名字

		- myslots[REDIS_CLUSTER_SLOTS/8]

			- 发送者当前的slot信息

			- 可以察觉slot变化

		- slaveof

			- 主:REDIS_NODE_NULL_NAME

			- 从:主名字

		- port

			- 发送者端口

		- flags

			- 发送者的flags

				- 可以察觉角色是否变化,状态是否变化

		- clusterMsgData

			- 内容 union

			- clusterMsgDataGossip ping

			- clusterMsgDataFail fail

			- clusterMsgDataPublish publish

	- 消息体

		- MEET,PING,PONG

			- clusterMsgDataGossip

				- nodename

				- ping_sent

					- 最后一次向节点发送ping的时间

				- pong_reveived

					- 最后一次从节点收到pong的时间

				- ip

					- 节点ip

				- port

					- 节点port

				- flags

					- 节点flags

			- 接受者收到两个clusterMsgDataGossip结构

				- 如果不认识,接受者根据ip port进行握手

				- 如果认识,更新信息

		- FAIL

			- clusterMsgDataFail

				- 只有节点名

			- 使用广播方式

				- Gossip协议有延迟性

				- FAIL情况一定是主机gg,需要及时故障转移

				- 标记了机器是fail之后再广播

				- 收到广播的节点标记包体的节点fail

		- PUBLISH

			- 客户端向集群节点发送PUBLISH,收到publish的节点不但向channel发送消息,也会在集群中广播一条publish消息

				- clusterMsgDataPublish

					- channel_len

					- message_len

					- bulk_data

						- channel,message信息都在里面

### 代码

- 数据结构

	- clusterState

		- clusterNode *myself

			- 指向自己的指针

		- uint64_t currentEpoch

			- 用于故障转移

		- int state

			- 集群当前状态

				- REDIS_CLUSTER_OK

				- REDIS_CLUSTER_FAIL

		- int size

			- 主节点数量

		- dict *nodes

			- 集群所有节点(包括自己)

			- 内容是clusterNode

		- dict *nodes_black_list

			- 一段时间内不会加入集群中

		- clusterNode *migrating_slots_to[REDIS_CLUSTER_SLOTS]

			- 迁出的节点

		- clusterNode *importing_slots_from[REDIS_CLUSTER_SLOTS]

			- 迁入的节点

		- clusterNode *slots[REDIS_CLUSTER_SLOTS]

			- 各个slot对应的clusterNode

		- zskiplist *slots_to_keys

			- score:slot

			- value:key

			- 需要迁移slot的时候可以快速拿到所有key

		- mstime_t failover_auth_time

			- 上次选举或者下次执行选举的时间

		- int failover_auth_count

			- 节点获得投票的数量

		- int failover_auth_sent

			- 1:本节点已经向其他节点发送了投票请求

		- int failover_auth_rank

			- slave的rank

		- uint64_t failover_auth_epoch

			- 当前的故障迁移epoch

		- mstime_t mf_end

			- 手动故障迁移时间限制

			- 0表示没有手动故障迁移

		- clusterNode *mf_slave

			- 手动故障迁移状态

		- long long mf_master_offset

			- 手动故障迁移的slave需要的offset

		- int mf_can_start

			- 手动故障迁移时候可以开始

		- uint64_t lastVoteEpoch

			- 集群最后一次迁移的epoch

		- int todo_before_sleep

			- 在clusterBeforeSleep中事件循环需要做的事情

		- long long stats_bus_messages_sent

			- 通过cluster连接发送的消息数量

		- long long stats_bus_messages_received

			- 通过cluster接收到的消息数量

	- clusterNode

		- mstime_t ctime

			- 创建时间

		- char name[REDIS_CLUSTER_NAMELEN]

			- 节点名字

		- int flags

			- 节点标识

		- uint64_t configEpoch

			- 当前配置epoch

		- unsigned char slots[REDIS_CLUSTER_SLOTS/8]

			- 节点自己占有的slot,使用bitmap

		- int numslots

			- 节点负责处理的slot数量

		- int numslaves

			- 如果是主机,他slaves节点数

		- struct clusterNode **slaves

			- 数组,指向slave节点

		- struct clusterNode *slaveof

			- 从机的主机节点

		- mstime_t ping_sent

			- 最后一次发出ping的时间

		- mstime_t pong_received

			- 最后一次接收PONG的时间

		- mstime_t fail_time

			- 最后一次被设置fail的时间

		- mstime_t voted_time

			- 最后一次给某个从节点投票的时间

		- mstime_t repl_offset_time

			- 最后一次从这个节点收到offset的时间

		- long long repl_offset

			- 最后一次知道的这个节点的复制offset

		- char ip[REDIS_IP_STR_LEN]

			- 节点的ip

		- int port

			- 节点port

		- clusterLink *link

			- 保存连接节点所需有关信息

		-  list *fail_reports

			- 记录所有其他节点对该节点的下线报告

	- clusterNodeFailReport

		- clusterNode *node

			- 报告下线的节点

		- mstime_t time

			- 报告时间

	- clusterLink

		- mstime_t ctime

			- 创建时间

		- int fd

			- socket fd

		- sds sndbuf

			- 输出buf

		- sds rcvbuf

			- 输入buf

		- struct clusterNode *node

			- 连接的node,没有节点的话为NULL

	- clusterMsgDataGossip

		- char nodename[REDIS_CLUSTER_NAMELEN]

		- uint32_t ping_sent

			- 最后一次向该节点发送PING的时间戳

		- uint32_t pong_received

			- 最后一次从该节点收到PONG的时间戳

		- char ip[REDIS_IP_STR_LEN]

			- 节点ip

		- uint16_t port

			- 节点端口号

		- uint16_t flags

			- 节点flag

		- uint32_t notused

			- 字节对其,不使用.为64bit对齐使用

	- clusterMsg

		- char sig[4]

			- 签名

		- uint32_t totlen

			- 消息总长度(头+正文)

		- uint16_t ver

			- 版本(当前0)

		- uint16_t notused0

			- 不使用,应该是对齐

		- uint16_t type

			- 消息类型

				- CLUSTERMSG_TYPE_PING

					- ping

				- CLUSTERMSG_TYPE_PONG

					- pong

				- CLUSTERMSG_TYPE_MEET

					- meet

				- CLUSTERMSG_TYPE_FAIL

					- 某个节点标记为fail

				- CLUSTERMSG_TYPE_PUBLISH

					- 发布/订阅

				- CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST

					- 申请投自己

				- CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK

					- 同意投票给自己

				- CLUSTERMSG_TYPE_UPDATE

					- 调整slot

				- CLUSTERMSG_TYPE_MFSTART

					- 手动故障转移

		- uint16_t count

			- 只在MEET PING PONG三种gossip信息采用

		- uint64_t currentEpoch

			- 发送者的当前epoch

		- uint64_t configEpoch

			- 主机:自己配置epoch

			- 从机:对应主机的配置epoch

		- uint64_t offset

			- 节点复制偏移

		- char sender[REDIS_CLUSTER_NAMELEN]

			- 发送者名字

		- unsigned char myslots[REDIS_CLUSTER_SLOTS/8]

			- 发送者的slot信息

		- char slaveof[REDIS_CLUSTER_NAMELEN]

			- 从节点:主机名字

			- 主机:REDIS_NODE_NULL_NAME

		- char notused1[32]

			- 没有使用

		- uint16_t port

			- 发送者port

		- uint16_t flags

			- 发送者flag

		- unsigned char state

			- 发送者的集群状态

		- unsigned char mflags[3]

			- 消息标志

		- union clusterMsgData data

			- 正文

			- clusterMsgData

				- union

					- ping:clusterMsgDataGossip[1]

						- char nodename[REDIS_CLUSTER_NAMELEN]

							- 节点名字

							- 开始的时候随机.MEET之后集群为节点设置正式的名字

						- uint32_t ping_sent

							- 最后一次向该节点发送ping的时间戳

						- uint32_t pong_received

							- 最后一次从该节点接收PONG的时间戳

						- char ip[REDIS_IP_STR_LEN]

							- 节点ip

						- uint16_t port

							- 节点端口

						- uint16_t flags

							- 节点标记

						- uint32_t notused

							- 字节对齐

					- fail:clusterMsgDataFail

						- char nodename[REDIS_CLUSTER_NAMELEN]

							- 下线节点名字

					- publish:clusterMsgDataPublish

						- uint32_t channel_len

							- 频道字节数

						- uint32_t message_len

							- 消息长度

						- unsigned char bulk_data[8]

							- 消息内容

							- 8字节为了对齐

					- Update:clusterMsgDataUpdate

						- uint64_t configEpoch

							- 节点配置epoch

						- char nodename[REDIS_CLUSTER_NAMELEN]

							- 节点名字

						- unsigned char slots[REDIS_CLUSTER_SLOTS/8]

							- 节点slot

- 逻辑

	- clusterInit(在serverInit中被调用)

		- 初始化集群

			- server.cluster内容清空

		- 步骤

			- server.cluster

				- clusterState

				- 初始化

			- clusterLoadConfig

				- 加载配置

			- 如果加载配置失败

				- 生成名字

					- /dev/urandom

				- 创建自己节点

				- 添加自己节点

				- 保存配置文件

			- 监听端口

				- aeCreateFileEvent

					- 监听ipv4 ipv6

					- 回调事件clusterAcceptHandler

						- Listen监听,用于绑定.通信fd使用clusterReadHandler回调

							- 收到的都是clusterMsg

							- 如果读完整条消息,使用clusterProcessPacket处理

								- 作用:node->rcvbuf有待处理的数据要处理.

								- 步骤

									- 检查合法性

										- 长度

											- 根据不同类型验证长度.gossip系列的要根据count配合验证

										- 版本

										- 类型

									- 使用sender记录当前消息发送者在cluster中的节点

									- sender存在且不是handshake节点

										- 消息中的currentEpoch比server.cluster的要大,更新成消息的

										- 消息中的configEpoch比server.cluster的要大,更新城消息节点的,再clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG| CLUSTER_TODO_FSYNC_CONFIG)

											- clusterDoBeforeSleep

												- 把flag打到server.cluster->todo_before_sleep,表示结束一个事件循环的时候要做的事情

										- 更新sender的repl_offset,repl_offset_time

										- 如果手动故障结束,而且本节点是sender的slave,而且server.cluster->mf_master_offset == 0

											- 更新server.cluster->mf_master_offset

									- 如果是PING或者是MEET

										- 如果sender不存在,且是MEET

											- 新建节点,把ip,link,port写入到节点

											- 把节点添加到cluster

											- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG)

										- clusterProcessGossipSection

											- 分析消息中的gossip,并更新本节点

											- 步骤

												- 根据消息中的count,获取节点信息

												- 遍历gossip中的节点

													- 节点在cluster中存在

														- Sender是主节点,且不是自己

															- 如果节点FAIL或者PFAIL

																- clusterNodeAddFailureReport

																	- 添加sender对节点的下线报告,

																	- 步骤

																		- fail_reports中查找,如果是sender的报告,更新;否则添加个报告

																- markNodeAsFailingIfNeeded

																	- 尝试将node标记为FIAL

																	- 步骤

																		- 标准是集群节点数/2+1

																		- 节点不是PFAIL,跳过

																		- 节点是FIAL,跳过

																		- 拿出失败的报告数clusterNodeFailureReportsCount

																			- 统计将node标记PFAIL或者FAIL的节点数

																			- 步骤

																				- clusterNodeCleanupFailureReports

																					- 去掉过期报告

																					- 步骤

																						- 超过2倍server.cluster_node_timeout的,从列表中删除

																				- 返回长度

																		- 如果自己是master,统计数++

																		- 如果统计数比标准小,返回

																		- 节点flags打上FAIL,去掉PFAIL.更新fail_time

																		- 如果自己的master,执行clusterSendFail

																			- 向其他节点发送报告节点的fail信息

																			- 步骤

																				- clusterBuildMessageHdr 创建FAIL类型信息

																				- 写入FAIL节点信息

																				- clusterBroadcastMessage

																					- 广播消息

																					- 步骤

																						- 遍历cluster中的节点

																						- clusterSendMessage

																							- 发送消息

																							- 步骤

																								- aeCreateFileEvent 对节点添加写监听,回调函数是clusterWriteHandler

																									- 集群信息写回调

																									- 步骤

																										- nwritten = write(fd, link->sndbuf, sdslen(link->sndbuf))

																										- aeDeleteFileEvent fd的写事件

																								- sndbuf写入缓冲区

																								- server.cluster->stats_bus_messages_sent++

																		- clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG)

															- 如果节点正常

																- clusterNodeDelFailureReport

														- 如果节点是FAIL或者PFAIL但是ip,port发生了变化

															- clusterStartHandshake

																- 还没握手的进行握手

																- 步骤

																	- 检查ip合法性

																	- 检查port合法性

																	- 检查是否已经握手 clusterHandshakeInProgress 是直接返回

																		- 看cluster.node是否存在相同ip port且handshaked的节点

																	- createClusterNode(NULL,REDIS_NODE_HANDSHAKE|REDIS_NODE_MEET)

																	- ip port写入到新节点

																	- clusterAddNode

													- 节点在cluster中不存在

														- Sender有地址,不在blacklist

															- clusterStartHandshake

										- clusterSendPing

											- 回复一个PONG

									- CLUSTERMSG_TYPE_PING或者CLUSTERMSG_TYPE_PONG或者CLUSTERMSG_TYPE_MEET

										- link.node存在

											- 如果node处于handshanke状态

												- 如果sender存在,

													- 尝试更新ip port

													- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_UPDATE_STATE)

													- freeClusterNode(link->node),返回

												- 对节点重命名

												- 关闭handshake状态,设置节点flags

												- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG)

											- 如果节点存在,但是名字不同

												- 设置连接为NOADDR,freeClusterLink

													- 信cluster不信link

												- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG),返回

										- 如果是PING,sender不在handshake状态

											- 更新ip,端口

											- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_UPDATE_STATE)

										- 如果是PONG,link.node存在

											- 更新link.node.pong_received

											- 清零最近一次ping_sent

											- 如果link.node节点超时

												- 清空PFAIL标记

												- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_UPDATE_STATE)

											- 如果link.node已经有FAIL标记

												- clearNodeFailureIfNeeded

													- 去掉节点的失败标记

													- 步骤

														- 如果是从节点,或者node->numslots == 0

															- 去掉REDIS_NODE_FAIL

															- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_UPDATE_STATE)

														- 如果是主节点,而且有slot,而且现在时间-之前记录的fail_time小于时间限制

															- 去掉REDIS_NODE_FAIL

															- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_UPDATE_STATE)

										- 如果sender存在

											- 需要更新sender信息

											- 如果消息头中slaveof是REDIS_NODE_NULL_NAME

												- 是

													- 成为主节点,clusterSetNodeAsMaster(sender)

												- 否

													- 节点是从节点

													- 在集群中查找slave的主机master

													- 如果sender之前是主机

														- 说明sender之前是主机,现在从机

														- clusterDelNodeSlots

														- 去掉sender的master标记,加上slave标记

														- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_UPDATE_STATE)

													- 如果sender的master存在,且sender的slaveof不是master

														- 说明slave的主节点变更

															- 把集群中sender节点的master干掉,更新成信息中的master

															- 信息中的master加上sender这个slave

														- 如果sender.slaveof存在

															- 说明sender的主机信息旧了,需要更新

															- clusterNodeRemoveSlave(sender->slaveof,sender)

														- clusterNodeAddSlave(master,sender)

														- sender->slaveof = master

														- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG)

										- 如果sender存在

											- 对比消息头的slot和当前sender的(或者其主机,根据规则)的slot,memcmp记录要更改的slot

										- Sender是主节点,存在要更改的slot

											- clusterUpdateSlotsConfigWith

										- 如果存在要更改的slot

											- 如果本机的配置epoch比消息的要大

												- 给sender发消息更新slot

										- 如果本机是主机,sender是主机,sender的配置epoch和本机一样

											- clusterHandleConfigEpochCollision

												- server.cluster.currentEpoch++

												- Myself.configEpoch=server.cluster.currentEpoch

										- clusterProcessGossipSection

											- 取出消息中gossip部分,更新对应节点

									- 如果是CLUSTERMSG_TYPE_FAIL

										- 在cluster中查找下线节点

										- 如果下线节点存在

											- 下线节点打开FAIL标记

											- 下线节点fail_time更新

											- 去掉PFAIL

											- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_UPDATE_STATE)

									- 如果是CLUSTERMSG_TYPE_PUBLISH

										- 如果当前有订阅频道或者订阅的模式

											- 解析消息

											- pubsubPublishMessage

									- 如果是CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST

										- 故障迁移授权请求

										- clusterSendFailoverAuthIfNeeded

											- 情况允许的话向sender投票支持

											- 步骤

												- 本节点是slave,或者本节点没有任何slot处理

													- 跳过

												- cluster的当前epoch比请求的要大

													- 跳过

												- server.cluster->lastVoteEpoch == server.cluster->currentEpoch,表示当前投票epoch已经参与了

													- 跳过

												- 请求节点是个master,或者请求节点的主节点是空,或者请求节点的主节点没有失败且不是force_ack

													- 跳过

												- 2倍timeout内已经投票过

													- 跳过

												- 请求的slot中如果cluster中的slot对应节点configEpoch比请求的要大

													- 跳过

												- clusterSendFailoverAuth

													- 为节点投票

													- 步骤

														- 组消息CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK

														- 给sender发消息

												- cluster更新最后投票epoch是当前epoch

												- 更新node.slaveof.vote_time

									- 如果是CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK

										- sender不存在,gg

										- sender是主机,有slot,而且sender的configEpoch不比自己的小

											- server.cluster->failover_auth_count++

											- clusterDoBeforeSleep(CLUSTER_TODO_HANDLE_FAILOVER)

									- 如果是CLUSTERMSG_TYPE_MFSTART

										- Sender不存在,或者sender的主机不是本机,报错

										- resetManualFailover

									- 如果是CLUSTERMSG_TYPE_UPDATE

										- 通过nodename找对应节点

										- 节点不存在.gg

										- 如果消息中的configEpoch不比自己的大,报错

										- 节点是slave,设置为主节点

											- 能发送这个消息的是主机,主机slot更新了

										- 更新节点的configEpoch

										- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_FSYNC_CONFIG)

										- clusterUpdateSlotsConfigWith

			- server.cluster->slots_to_keys

				- 初始化

			- resetManualFailover

				- 手动故障初始化

	- clusterCron

		- 遍历所有节点

			- 超过handshake时间的,释放节点

			- 没有建立连接的,创建连接

				- 非阻塞连接,如果连接失败,下次继续

				- 给connect的fdaeCreateFileEvent 回调函数是clusterReadHandler

				- 新节点标记是MEET,clusterSendPing MEET消息.否则发送PING消息

				- 去掉MEET标记

		- 随机获取5个节点,ping一个pong了最久的

		- 遍历所有节点

			- 跳过自己节点,NOADDR节点,HANDSHAKE节点

			- 如果自己是从机,节点是主机,节点正常

				- 统计节点正常的slave

				- 如果节点正常的slave个数为0且节点是有slot的,孤儿主节点统计++

				- 记录最大的slaves个数

				- 如果节点是自己的主机  this_slaves=统计的slave

			- 如果有link,但是创建link时间>node_timeout 而且之前有过ping pong链接,当前等待pong 而且ping已经超过半个node_timeout时间

				- freeClusterLink

				- 下次clusterCron会自动重连

			- 有link,没有ping过,pong过期

				- ping

			- mf_end,自己是master,而且mf_slave是当前节点

				- ping

			- 如果之前没有ping

				- 跳过

			- Ping了至今超时,但是不是PFAIL或者FAIL

				- 打标PFAIL

				- 后续需要update

		- 自己是从节点但是没有主机ip port,但是自己的主机节点有ip port

			- 设置自己的slaveof.ip slaveof.port

		- 重置手工故障时间

		- 如果自己是slave

			- clusterHandleManualFailover

				- mf_end==0跳过

				- mf_can_start 已经触发了手动故障.跳过 

				- mf_master_offset==0 等待配置offset 跳过

				- mf_master_offset和主机的复制offset一致

					- mf_master_offset=1

					- 可以开始迁移

			- clusterHandleSlaveFailover

				- 当前是从节点,主机gg了,要选主,做故障迁移

				- 步骤

					- 跳过以下情况

						- 自己是主机

						- 自己的主机是空的

						- 主机没有fail而且没有手工失败

						- 主机没有slot

					- 统计节点的数据时间 data_age

						- 复制状态是REDIS_REPL_CONNECTED

							- 是

								- data_age是主机lastinteraction至今

							- 否

								- data_age是之前down至今

					- 如果data_age>cluster_node_timeout,减去timeout时间

					- data_age比是上次ping之后timeout的10倍要大

						- 如果没有手工迁移,退出

					- 如果上次auth时间超过了两次auth_timeout

						- 初始化failover_auth

							- failover_auth_time 是当前时间往后随机一个数

							- failover_auth_count = 0

							- failover_auth_sent = 0

							- failover_auth_rank = clusterGetSlaveRank()

								- 偏移越多越高

							- 根据rank加上failover_auth_time

						- 如果是手工迁移,failover_auth_time=当前时间,rank是0

						- clusterBroadcastPong

						- 返回

					- failover_auth_sent == 0且mf_end == 0

						- 计算rank

						- 如果自己rank比failover_auth_rank要大

							- 大多少级,大多少秒,记作delay

							- 刷新failover_auth_rank

							- failover_auth_time += delay

					- failover_auth_time 比当前大,退出

					- auth_age>auth_timeout 返回

					- 如果failover_auth_sent == 0

						- 向其他节点发送故障转移请求

						- currentEpoch++

						- failover_auth_epoch=currentEpoch

						- clusterRequestFailoverAuth

							- 给其他节点请求做主

							- 步骤

								- 包装CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST信息

								- clusterBroadcastMessage

						- failover_auth_sent = 1

						- clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_FSYNC_CONFIG)

					- 如果failover_auth_count >= needed_quorum

						- 获得足够多的投票

							- 超过半数

						- clusterSetNodeAsMaster

						- replicationUnsetMaster

						- 设置旧master的slot

							- clusterNodeGetSlotBit

							- clusterDelSlot

								- 旧master节点的slot清空,cluster中的数组清空

							- clusterAddSlot

								- 改为自己的

						- myself->configEpoch = server.cluster->failover_auth_epoch

						- clusterUpdateState

						- clusterSaveConfigOrDie

						- clusterBroadcastPong(所有节点)

						- resetManualFailover

			- 存在孤儿master,有主机的slaves超过两个,而且自己所在的master有最多的slave

				- clusterHandleSlaveMigration

					- 集群状态不是ok,跳过

					- 本机没有设置slaveof,跳过

					- 自己的master的从机个数小于server.cluster_migration_barrier,跳过

					- 遍历node,找有slot的而且没有okslave的正常主机

						- 在node中找有最多slave但是名字最小的slave做备选

					- 如果备选就是自己,把自己的主机设置成找到的孤儿master

		- 需要更新节点或者当前state是REDIS_CLUSTER_FAIL

			- clusterUpdateState

	- clusterBeforeSleep

		- 进入下一个事件循环之前调用

		- 步骤

			- todo_before_sleep标

				- CLUSTER_TODO_HANDLE_FAILOVER

					- clusterHandleSlaveFailover

				- CLUSTER_TODO_UPDATE_STATE

					- clusterUpdateState

						- 更新集群状态

						- 步骤

							- 去掉todo_before_sleep的更新标志

							- 临时new_state=OK  static among_minority_time=0

							- 自己是主机,而且第一次更新的时间至今<REDIS_CLUSTER_WRITABLE_DELAY 退出

							- 检查各个slot的归属节点,如果有NULL的或者是FAIL的,

								- 临时记下new_state是CLUSTER_FAIL

							- 遍历所有节点,如果是有管理slot的 而且FAIL或者PFAIL,统计个数 unreachable_masters

							- 如果unreachabel_masters比当前节点总数一半要多,

								- 临时state是CLUSTER_FAIL

								- 临时among_minority_time=当前时间

									- 因为是静态变量,会影响下次的beforesleep

							- 如果临时state和当前cluster.state不一致

								- 临时变量rejoin_delay记录cluster_node_timeout

								- 超过5s或者小于0.5s,取边界

								- 如果new_state还是OK,而且自己是主机.among_minority_time至今<rejoin_delay

									- 退出

									- 能进入到这个流程,证明之前是有过主节点gg的情况出现

								- server.cluster->state = new_state

				- CLUSTER_TODO_SAVE_CONFIG

					- 根据有没有CLUSTER_TODO_FSYNC_CONFIG进行是否刷盘的clusterSaveConfigOrDie

			- 重置todo_before_sleep = 0













