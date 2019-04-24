## 删除
### 序列式容器(数组式容器)
因为数组式容器删除节点之后后续的内容需要挪动位置,所以迭代器会失效(下一个的内存地址会变化).erase的时候会返回下一个迭代器的位置.
```cpp
for (ite=vec.begin();ite!=vec.end();){
    (*ite).Dosomething();
    if(ShouldDelete(ite)){
        ite=vec.erase(ite);
    }
    else{
        ite++;
    }
}
```
### 关联式迭代器(树)
关联式(map,set,multimap,multiset)  
删除当前ite,只会让当前的失效.递增迭代器即可
```cpp
for(ite=map.begin();ite!=map.end();){
    (*ite).DoSomething();
    if(ShouldDelete(ite)){
        map.erase(ite++);
    }
    else{
        ite++;
    }
}
注意:这里原来的ite会不存在的.使用ite++成功是因为ite++的时候,其实有一个过程是生成了临时变量,返回的是临时变量.执行`map.erase(ite++)`的时候,erase的是ite,但是ite最后是被临时变量ite
```

### 链式容器
list删除的时候不影响后继,而且**也会返回下一个ite**
```cpp
for(ite=list.begin();ite!=end();){
    (*ite).DoSomething();
    if(ShouldDelete(ite)){
        list.erase(ite++);//ite = list.erase(ite);
    }
    else{
        ite++;
    }
}
```
注意:和上面原理一样,其实ite已经不存在