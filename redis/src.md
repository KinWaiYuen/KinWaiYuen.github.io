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







