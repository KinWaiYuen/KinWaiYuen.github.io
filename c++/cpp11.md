## auto
根据上下文情况确定auto变量真正的类型  
auto作为函数返回值时，只能用于定义函数，不能用于声明函数。   
在有文件中引用不能编译通过,但是写在源文件中可以编译通过  
类似inline成员函数  

## nullptr
因为以前的NULL都是0.为了区分开真正的NULL和空指针,所以有一个固定的指针表示空指针  

## for
```cpp
int numbers[] = {1,2,3};
for (auto number:numbers){}
```
类似foreach.  
STL可用

## STL
### array
```cpp

#include <iostream>
#include <array>

int main()
{
    std::array<int, 4> a = {1,2,3};

    for(auto i:a){
        std::cout << i << std::endl;
    }
    std::cout << "sizeof:" << sizeof(a)<<"capacity"<< std::endl;
    std::cout << "size:" <<a.size()<<" maxsize:" <<a.max_size()<<std::endl;
    return 0;
}
```
输出
```
1
2
3
0
sizeof:16capacity
size:4 maxsize:4
```

注意:没有capacity函数  
size和maxsize一样  

### forward_list
单向链表.没有size,可以使用**std::distance**获取大小  
```cpp
#include <iostream>
#include <forward_list>

void printForwardList(std::forward_list<int> *pFL){
    std::cout << "list" << std::endl;
    for(auto i:*pFL){
        std::cout << i << std::endl;
    }
    std::cout << "<<<<<<<<<<" << std::endl;
}

int main()
{
    std::forward_list<int> l = {0,1,2,3,3,3,3,3};
    printForwardList(&l);
    std::cout << "list.maxsize"<< l.max_size()<<std::endl;
    std::cout << "size" << std::distance(l.begin(), l.end());
    l.remove(3);
    printForwardList(&l);
    std::cout << "list.maxsize after remove"<< l.max_size()<<std::endl;
    std::cout << "size" << std::distance(l.begin(), l.end());
    l.remove(3);
    return 0;
}
```

输出  
```
list
0
1
2
3
3
3
3
3
<<<<<<<<<<
list.maxsize1152921504606846975
size8list
0
1
2
<<<<<<<<<<
list.maxsize after remove1152921504606846975
size3
```

### underorder_map
底层是哈希表,查找快 
官网源码
```cpp
#include <iostream>
#include <string>
#include <unordered_map>

void printUnorderMap(std::unordered_map<std::string, std::string> *pUM){

    unsigned n = pUM->bucket_count();
    std::cout << "mymap has " << n << " buckets.\n";
    for (unsigned i = 0; i<n; ++i)
    {
        std::cout << "bucket #" << i << " contains: ";
        for (auto it = pUM->begin(i); it != pUM->end(i); ++it)
            std::cout << "[" << it->first << ":" << it->second << "] ";
        std::cout << "\n";
    }

    std::cout << "myMap size is" << pUM->size() << "mymap maxsize is " << pUM->max_size() << std::endl;
    std::cout << "print elem"<< std::endl;
    std::unordered_map<std::string, std::string>::iterator i = pUM->begin();
    for(;i != pUM->end(); i++){
        std::cout << "["<< i->first << "]"<< i->second  << std::endl;
    }
    return;
}
int main()
{
    std::unordered_map<std::string, std::string> mymap =
    {
        { "house","maison" },
        { "apple","pomme" },
        { "tree","arbre" },
        { "book","livre" },
        { "door","porte" },
        { "grapefruit","pamplemousse" }
    };

    printUnorderMap(&mymap);

    mymap["apple"] = "test";
    printUnorderMap(&mymap);

    auto pr=mymap.insert(std::pair<std::string, std::string>{"apple","test_add"});
    std:: cout << ">>>>>>>element " << (pr.second ? "was" : "was not") << " inserted." << std::endl;
    printUnorderMap(&mymap);
    pr=mymap.insert(std::pair<std::string, std::string>{"apple1","test_add"});
    std:: cout << ">>>>>>>element " << (pr.second ? "was" : "was not") << " inserted." << std::endl;
    printUnorderMap(&mymap);

    mymap.erase("apple");
    mymap.erase("tree");
    mymap.erase("book");
    printUnorderMap(&mymap);

    mymap.rehash(3);
    printUnorderMap(&mymap);
    mymap.rehash(20);
    printUnorderMap(&mymap);




    return 0;
}
```
输出

```
mymap has 7 buckets.
bucket #0 contains: [tree:arbre]
bucket #1 contains: [book:livre] [apple:pomme]
bucket #2 contains: [door:porte]
bucket #3 contains:
bucket #4 contains: [house:maison]
bucket #5 contains:
bucket #6 contains: [grapefruit:pamplemousse]
myMap size is6mymap maxsize is 576460752303423487
print elem
[grapefruit]pamplemousse
[door]porte
[tree]arbre
[book]livre
[apple]pomme
[house]maison
mymap has 7 buckets.
bucket #0 contains: [tree:arbre]
bucket #1 contains: [book:livre] [apple:test]
bucket #2 contains: [door:porte]
bucket #3 contains:
bucket #4 contains: [house:maison]
bucket #5 contains:
bucket #6 contains: [grapefruit:pamplemousse]
myMap size is6mymap maxsize is 576460752303423487
print elem
[grapefruit]pamplemousse
[door]porte
[tree]arbre
[book]livre
[apple]test
[house]maison
>>>>>>>element was not inserted.
mymap has 7 buckets.
bucket #0 contains: [tree:arbre]
bucket #1 contains: [book:livre] [apple:test]
bucket #2 contains: [door:porte]
bucket #3 contains:
bucket #4 contains: [house:maison]
bucket #5 contains:
bucket #6 contains: [grapefruit:pamplemousse]
myMap size is6mymap maxsize is 576460752303423487
print elem
[grapefruit]pamplemousse
[door]porte
[tree]arbre
[book]livre
[apple]test
[house]maison
>>>>>>>element was inserted.
mymap has 17 buckets.
bucket #0 contains:
bucket #1 contains:
bucket #2 contains:
bucket #3 contains:
bucket #4 contains: [house:maison] [grapefruit:pamplemousse]
bucket #5 contains:
bucket #6 contains:
bucket #7 contains:
bucket #8 contains: [book:livre] [door:porte]
bucket #9 contains: [apple1:test_add]
bucket #10 contains:
bucket #11 contains: [apple:test]
bucket #12 contains:
bucket #13 contains:
bucket #14 contains:
bucket #15 contains: [tree:arbre]
bucket #16 contains:
myMap size is7mymap maxsize is 576460752303423487
print elem
[apple1]test_add
[apple]test
[tree]arbre
[book]livre
[door]porte
[house]maison
[grapefruit]pamplemousse
mymap has 17 buckets.
bucket #0 contains:
bucket #1 contains:
bucket #2 contains:
bucket #3 contains:
bucket #4 contains: [house:maison] [grapefruit:pamplemousse]
bucket #5 contains:
bucket #6 contains:
bucket #7 contains:
bucket #8 contains: [door:porte]
bucket #9 contains: [apple1:test_add]
bucket #10 contains:
bucket #11 contains:
bucket #12 contains:
bucket #13 contains:
bucket #14 contains:
bucket #15 contains:
bucket #16 contains:
myMap size is4mymap maxsize is 576460752303423487
print elem
[apple1]test_add
[door]porte
[house]maison
[grapefruit]pamplemousse
mymap has 5 buckets.
bucket #0 contains: [grapefruit:pamplemousse] [door:porte]
bucket #1 contains:
bucket #2 contains: [house:maison] [apple1:test_add]
bucket #3 contains:
bucket #4 contains:
myMap size is4mymap maxsize is 576460752303423487
print elem
[grapefruit]pamplemousse
[door]porte
[house]maison
[apple1]test_add
mymap has 23 buckets.
bucket #0 contains:
bucket #1 contains:
bucket #2 contains:
bucket #3 contains:
bucket #4 contains:
bucket #5 contains:
bucket #6 contains: [house:maison] [door:porte]
bucket #7 contains:
bucket #8 contains:
bucket #9 contains: [apple1:test_add]
bucket #10 contains:
bucket #11 contains:
bucket #12 contains:
bucket #13 contains:
bucket #14 contains:
bucket #15 contains:
bucket #16 contains: [grapefruit:pamplemousse]
bucket #17 contains:
bucket #18 contains:
bucket #19 contains:
bucket #20 contains:
bucket #21 contains:
bucket #22 contains:
myMap size is4mymap maxsize is 576460752303423487
print elem
[apple1]test_add
[house]maison
[door]porte
[grapefruit]pamplemousse
```
删除元素在没有rehash的时候是不会自动rehash的. 但是rehash并不是一定根据给定大小进行rehash.例如指定rehash 3,结果有5个桶.文档表示rehash的参数是个参考值.   

注意的是,因为使用hash,是没有顺序的  
同一个key对应的是一个value.如果key已经在unordermap中,是不能insert进去的.
```
void rehash( size_type n );
Set number of buckets
Sets the number of buckets in the container to n or more.

If n is greater than the current number of buckets in the container (bucket_count), a rehash is forced. The new bucket count can either be equal or greater than n.

If n is lower than the current number of buckets in the container (bucket_count), the function may have no effect on the bucket count and may not force a rehash.

A rehash is the reconstruction of the hash table: All the elements in the container are rearranged according to their hash value into the new set of buckets. This may alter the order of iteration of elements within the container.

Rehashes are automatically performed by the container whenever its load factor is going to surpass its max_load_factor in an operation.

Notice that this function expects the number of buckets as argument. A similar function exists, unordered_map::reserve, that expects the number of elements in the container as argument.
```

### unordered_set
```cpp

#include <iostream>
#include <string>
#include <unordered_set>
#include <set>
int main()
{
    std::unordered_set<int> unorder_set;
    unorder_set.insert(7);
    unorder_set.insert(5);
    unorder_set.insert(3);
    unorder_set.insert(4);
    unorder_set.insert(6);
    std::cout << "unorder_set:" << std::endl;
    for (auto itor : unorder_set)
    {
        std::cout << itor << std::endl;
    }
    std::cout << "size of unorder set"<< unorder_set.size()<<std::endl;

    unorder_set.insert(4);
    std::cout << "unorder_set after 4 added:" << std::endl;
    for (auto itor : unorder_set)
    {
        std::cout << itor << std::endl;
    }
    std::cout << "size of unorder set"<< unorder_set.size()<<std::endl;

    unorder_set.erase(4);
    std::cout << "unorder_set after 4 added:" << std::endl;
    for (auto itor : unorder_set)
    {
        std::cout << itor << std::endl;
    }
    std::cout << "size of unorder set"<< unorder_set.size()<<std::endl;


    std::set<int> set;
    set.insert(7);
    set.insert(5);
    set.insert(3);
    set.insert(4);
    set.insert(6);
    std::cout << "set:" << std::endl;
    for (auto itor : set)
    {
        std::cout << itor << std::endl;
    }
}
```

输出
```
unorder_set:
6
4
3
5
7
size of unorder set5
unorder_set after 4 added:
6
4
3
5
7
size of unorder set5
unorder_set after 4 added:
6
3
5
7
size of unorder set4
set:
3
4
5
6
7
```

## 多线程
### thread
c++11 引入了boost的多线程成为c++标准  
虚拟机无法执行  上代码
```cpp
#include <thread>
void threadfun1()
{
    std::cout << "threadfun1 - 1\r\n" << std::endl;
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "threadfun1 - 2" << std::endl;
}

void threadfun2(int iParam, std::string sParam)
{
    std::cout << "threadfun2 - 1" << std::endl;
    std::this_thread::sleep_for(std::chrono::seconds(5));
    std::cout << "threadfun2 - 2" << std::endl;
}

int main()
{
    std::thread t1(threadfun1);
    std::thread t2(threadfun2, 10, "abc");
    t1.join();
    std::cout << "join" << std::endl;
    t2.detach();
    std::cout << "detach" << std::endl;
}
```

### atomic
```cpp

#include <thread>
#include <atomic>
#include <stdio.h>
#include <list>
static std::atomic_bool bIsReady(false);
static std::atomic_int iCount(100);
void threadfun1()
{
    if (!bIsReady) {
        std::this_thread::yield();
    }
    while (iCount > 0)
    {
        printf("iCount:%d\r\n", iCount--);
    }
}

int main()
{
    std::list<std::thread> lstThread;
    for (int i = 0; i < 10; ++i)
    {
        lstThread.push_back(std::thread(threadfun1));
    }
    for (auto& th : lstThread)
    {
        th.join();
    }

    return 0;
}
```

### 环境变量
```cpp
#include <iostream>           // std::cout
#include <thread>             // std::thread
#include <mutex>              // std::mutex, std::unique_lock
#include <condition_variable> // std::condition_variable

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void print_id(int id) {
    std::unique_lock<std::mutex> lck(mtx);
    while (!ready) cv.wait(lck);
    // ...
    std::cout << "thread " << id << '\n';
}

void go() {
    std::unique_lock<std::mutex> lck(mtx);
    ready = true;
    cv.notify_all();
}

int main()
{
    std::thread threads[10];
    // spawn 10 threads:
    for (int i = 0; i<10; ++i)
        threads[i] = std::thread(print_id, i);

    std::cout << "10 threads ready to race...\n";
    go();                       // go!

    for (auto& th : threads) th.join();

    return 0;
}
```

//TODO 补充并行编程

## 智能指针
使用counter,管理对象增加或者减少.如果最后一个指针管理对象销毁,计数器为1.销毁指针管理对象同时把智能指针管理的指针进行delete  

### shared_ptr
```cpp
#include <iostream>
#include <memory>
class Test
{
public:
    Test()
    {
        std::cout << "Test()" << std::endl;
    }
    ~Test()
    {
        std::cout << "~Test()" << std::endl;
    }
};
int main()
{
    std::shared_ptr<Test> p1 = std::make_shared<Test>();
    std::cout << "1 ref:" << p1.use_count() << std::endl;
    {
        std::shared_ptr<Test> p2 = p1;
        std::cout << "2 ref:" << p1.use_count() << std::endl;
    }
    std::cout << "3 ref:" << p1.use_count() << std::endl;
    std::cout <<"before return"<< std::endl;
    return 0;
}
```

输出
```
Test()
1 ref:1
2 ref:2
3 ref:1
before return
~Test()
```
return 0之后自动析构  

#### shared_ptr 循环引用
```cpp
#include<memory>
#include<iostream>
using namespace std;

struct Node
{
	shared_ptr<Node> _pre;
	shared_ptr<Node> _next;

	~Node()
	{
		cout << "~Node():" << this << endl;
	}
	int data;
};

void FunTest()
{
	shared_ptr<Node> Node1(new Node);
	shared_ptr<Node> Node2(new Node);
	Node1->_next = Node2;
	Node2->_pre = Node1;

	cout << "Node1.use_count:"<<Node1.use_count() << endl;
	cout << "Node2.use_count:"<< Node2.use_count() << endl;
}

void FunTest1()
{
	shared_ptr<Node> Node1(new Node);

	cout << "Node1.use_count:"<<Node1.use_count() << endl;
}


int main()
{
    FunTest1();
	system("pause");
	FunTest();
	system("pause");
	return 0;
}
```
输出
```
Node1.use_count:1
~Node():0x189f010
sh: pause: command not found
Node1.use_count:2
Node2.use_count:2
sh: pause: command not found
```
能看到;FunTest1是可以释放只能指针,但是Funtest不能  
因此需要weak_ptr

### std::weak_ptr

```cpp
#include<memory>
#include<iostream>
using namespace std;

struct Node
{
	weak_ptr<Node> _pre;
	weak_ptr<Node> _next;

	~Node()
	{
		cout << "~Node():" << this << endl;
	}
	int data;
};

void FunTest()
{
	shared_ptr<Node> Node1(new Node);
	shared_ptr<Node> Node2(new Node);
	Node1->_next = Node2;
	Node2->_pre = Node1;

	cout <<"Node1.use_count:"<< Node1.use_count() << endl;
	cout <<"Node2.use_count:"<< Node2.use_count() << endl;
}

int main()
{
	FunTest();
	system("pause");
	return 0;
}
```

输出
```
Node1.use_count:1
Node2.use_count:1
~Node():0x1388060
~Node():0x1388010
sh: pause: command not found
```

在node中使用weak_ptr进行引用,这样在FunTest不变的时候,达到了释放的作用  

**一个强引用是指当被引用的对象仍活着的话，这个引用也存在（也就是说，只要至少有一个强引用，那么这个对象 就不会也不能被释放**  
**弱引用当引用的对象活着的时候不一定存在。仅仅是当它自身存在的时的一个引用.弱引用并不修改该对象的引用计数，这意味这弱引用它并不对对象的内存进行管理**



