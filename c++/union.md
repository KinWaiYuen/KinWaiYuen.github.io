
union特性
会根据member中最大的成员开辟内存空间,在类中也是
```cpp
#include <iostream>

class A{
private: struct xxx{
        char b;
    };
public:
         union{
        xxx ox;
         A* next;
         long long int c;} data;

};


int main()
{

    union oxxk{
        int a;
        char b;
        long long int c;
    }test;
    test.b='j';
    std::cout << "union size" << sizeof(test)<<std::endl;
    A *a = new A();
    std::cout << "A.size"<< sizeof(*a) << std::endl;
    a->data.c = 13240234040320;
    std::cout << "A.size"<< sizeof(*a) << std::endl;

    return 0;
}
```
输出
```
union size8
A.size8
A.size8
```
可以看到就算union在类中,类的成员只有一个char,最终还是会分配了size未longlongint的给类

union可以看成是 对同一个内存空间可以根据不同的成员变量来获取.在内存管理中左右比较优雅,因为逻辑上如果未分配,指向next;已经分配的话,就可以直接放成员变量.这样可以拿到一个紧凑的内存池