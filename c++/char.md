```cpp

char *p = "assdfdf";
p[3]='µ'; //此处会core.因为p指向一个常量,常量不能更改
```