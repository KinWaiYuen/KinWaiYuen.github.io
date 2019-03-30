# libevent
开源,精简,跨平台,专注于网络通信.

./configure 检查依赖,生成makefile
make 编译
make install so等必要资源放在系统对应目录下

编译需要 -levent 连接libevent的库

libevent.so -> /usr (unix system resource)
/usr/local/lib

### libevent 编写框架
- 创建event_base
    - struct event_base *event_base_new(void);
        - struct event_base *base = ebent_base_new()
- 创建event
    - 常规事件event
        - event_new
    - bufferevent
        - bufferevent_socket_new()
- event->event_base
    - event_add
- 循环监听事件满足
    - event_base_dispatch(base)l
- 释放event_base
    - event_base_free(base)



### 状态机
- 未决 有资格被处理,但是还没有被处理 (等待触发)
- 非未决 没有资格被处理(等待加入,加入之后才算)

**自己看法**
pending就是加入了事件监听,还没处理的.
not pending就是要么还没加入事件监听,要么处理了还没再加入事件监听.

create_new之后,因为还没加入到event中,没有资格被处理.
event_add之后,添加之后就是未决,但是还没被触发
在循环监听中,对应事件触发的时候,成为激活态.
激活态调用回调后,重新进入非未决状态.等到event_add之后才是未决


```ditaa
                                                                               │         
                                                                               │         
                                                                               │         
                                                                          event_new      
                                                                               │         
                                                                               │         
                                                                               │         
┌────────EV_PERSIST-event_add────────────────┐                                 │         
│                                            │                                 │         
│                                            ▼                                 ▼         
│                                   ┌─────────────────┐               ┌─────────────────┐
│                                   │                 │               │                 │
│                                   │      event      │               │    new event    │
│           event_del───────────────│    (pending)    │◀───event add──│  (not pending)  │
│           │                       │                 │               │                 │
│           │                       └─────────────────┘               └─────────────────┘
│           │                                │                                           
│           │                                │                                           
│           │                                │                                           
│           │                                │                                           
│           │                                │                                           
│           ▼                                │                                           
│  ┌─────────────────┐                       │                                           
│  │                 │                       │                                           
│  │      event      │                       │                                           
└──│  (not pending)  │              event_base_dispatch                                  
   │                 │                      &&                                           
   └─────────────────┘                 fd triggered                                      
            ▲                                │                                           
            │                                │                                           
            │                                │                                           
            │                                │                                           
        after dealed                         │                                           
            │                                │                                           
            │                                ▼                                           
   ┌─────────────────┐              ┌─────────────────┐                                  
   │                 │              │                 │                                  
   │      event      │              │      event      │                                  
   │    (deal)       │◀─call_back───│  (activation)   │                                  
   │                 │              │                 │                                  
   └─────────────────┘              └─────────────────┘                                                                 
```

### bufferevent
管理读缓冲,写缓冲.
根据读缓冲和写缓冲是否能读/能写分别cb(call back)

使用bufferevent可以实现半关闭 全双工
