## resize
```cpp
#include <iostream>
#include <iostream>
#include <string>
using namespace std;
int main ()
{
    int size = 0;
    int length = 0;
    unsigned long maxsize = 0;
    int capacity=0;
     string str ("12345678");
     /* string str_custom; */
     /* str_custom = str; */
     str.resize (5);
     size = str.size();
     length = str.length();
     maxsize = str.max_size();
     capacity = str.capacity();
     cout << "size = " << size << endl;
     cout << "length = " << length << endl;
     cout << "maxsize = " << maxsize << endl;
     cout << "capacity = " << capacity << endl;
     cout << str.c_str() << endl;
     cout <<  "str 7:" << str[7] << endl;
     str.resize(15);
     cout << "after  resize to 15" << endl;
          

     size = str.size();
     length = str.length();
     maxsize = str.max_size();
     capacity = str.capacity();
     cout << "size = " << size << endl;
     cout << "length = " << length << endl;
     cout << "maxsize = " << maxsize << endl;
     cout << "capacity = " << capacity << endl;
     cout << str.c_str() << endl;
     cout <<  "str 7:" << str[7] << endl;
     
     return 0;
 }


```
运行结果
```
size = 5
length = 5
maxsize = 4611686018427387897
capacity = 8
12345
str 7:8
after  resize to 15
size = 15
length = 15
maxsize = 4611686018427387897
capacity = 16
12345
str 7:
```
`resize`后capacity不变,但是size会缩短,可以理解为内存大小不变,但是size(字符串长度)变了.
//问题:1.resize变大 2.resize后原来有的字符还是否存在

### 1.resize变大:
如果直接resize变大,上述程序直接resize成15,结果:
```
size = 15
length = 15
maxsize = 4611686018427387897
capacity = 16
12345678
str 7:8
```
`size`成15,`length`是15
取`str[7]`能看到是8,原来的不影响

### 2.resize后原有字符
大缩到小的时候,看到`str[7]`是存在的,所以缩小的时候其实就在原来内存中加上`'\0'`,内存大小不变
小变大的时候,看到后面多余的会被删掉.

## []和at()
上面`resize`到5后,使用at(7)会返回报错
```
terminate called after throwing an instance of 'std::out_of_range'
  what():  basic_string::at
[2]    17445 abort (core dumped)  ./resize
```
### []和at()的区别
`[]`返回的是对应第几个字符,**不会检查字符的长度合法性**,也就是不会对`'\0'`位置合法性进行检查.ex:
```
1234'\0'5678
```
里面的5 6 7 8是可以正常访问的
`at()`会访问时候针对合法性进行判断.例如上述的例子,5 以后的不会访问到,访问会返回`std::out_of_range`报错
问题:访问`[]`超过size返回是什么
ex:resize到5后,访问10(超过原有长度)
返回:
```
terminate called after throwing an instance of 'std::out_of_range'
  what():  basic_string::at
[2]    19029 abort (core dumped)  ./resize
```
非法访问导致core


