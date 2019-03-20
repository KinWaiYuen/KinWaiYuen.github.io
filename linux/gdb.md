常用


- l list 列出代码
- b break 例如b x.c 10 在x.c的第10行打断点 b main 在main打断点
- r run 执行
- n next 吓一跳
- s step 跳进去循环体
- p print p i 打印变量
- x 看十六进制内存
- c continue 继续断点后指令 
- ctrl-x ctrl-a 图形界面 再次操作返回代码界面
- bt 查看core的栈
- info thread 查看线程信息
- thread x 看第几线程的
- 查看core: 直接gdb 二进制之后run,程序停止的位置就是core的位置
- start 单步执行  打开gdb之后start默认单步执行不需要断点
- finish 结束当前函数调用,返回到调用点
- 带参数的main运行  set args xx yy zz 回车  然后start/run
- 直接run xx yy zz也可以传入参数
- info b 查看断点
- b 41 if i=4 条件断点.符合条件的时候才断.在for循环里面用的较多 格式:b <位置> if <条件>
- ptype x 查看变量类型
- bt 查看当前调用栈
- frame x 查看对应栈帧 当函数调用的时候查看其它栈帧的变量,可以使用frame跳过去对应栈帧(ex:main中有变量,没有传过去函数里,跳入到函数中时查看时候需要跳过栈帧来看对应变量,因为当前栈帧看不到)
- display x 设置跟踪变量.例如需要跟踪某个变量在循环中,display x可以每次next的时候都看到x的值
- undisplay <display编号> 取消跟踪变量
- 编译的时候没-g:没有符号表 file <有符号表的二进制> gdb会从二进制中再次读取符号表
