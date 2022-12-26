---
title: GDB_Tutorial
date: 2022-12-14 10:09:09
tags: 调试
author: xuaidong
---

***
### 什么是GDB

GDB，GNU 项目调试器,允许您查看另一个程序在执行时“内部”发生了什么——或者另一个程序在崩溃时正在做什么。
GDB 可以做四种主要的事情(加上其他支持这些的事情)来帮助你在行动中捕捉错误:

- 启动程序,设置可能影响其行为的任何内容
- 可以让程序在特定条件下停止
- 当程序停止时查看发生了什么
- 更改程序中的内容,这样就可以尝试修改一个错误并看到表现

被调试的程序可以用 Ada、C、C++、Objective-C、Pascal(以及许多其他语言)编写。这些程序可能在与GDB(本机)相同的机器上或在另一台机器(远程)上执行. GDB 可以在大多数流行的 UNIX 和 Microsoft Windows 变体上运行。

***
### GDB调试信息/符号表

符号是变量和函数,符号表是可执行文件中的变量和函数表,通常符号表只包含符号的内存地址,比如0x898964d类似的地址,计算机不关心命名的变量和函数,但是gdb为了调试,需要能够引用变量和函数名称而不是地址,因此我们需要"调试信息"编译在代码中.
"调试信息"告诉 GDB 2件事:
- 如何将符号的地址和源代码的名称关联
- 如何将机器代码的地址与每一行源代码关联

检查可执行文件是否携带调试信息一般使用file programe_path查看,如果提示*with debug_info, not stripped*代表携带调试信息,如果提示*striped*代表调试信息被剥离,或者使用objdump -syms program_path | grep debug,如果有debug字段代表就是携带调试信息. 或者直接gdb ./syms_file,出现Reading symbols..., 执行程序发布的时候通常编译选项是-g -O3,然后剥离一份带调试信息一份不带调试信息的执行程序,用于后续奔溃时调试定位问题.
例如步骤:
- *objcopy --only-keep-debug program program.dbg* 只保留调试信息的可执行文件  (无所谓尾缀是什么,dbx xxx都可以)
- *strip --strip-all program* 剥离所有调试符号信息的可执行文件
- *objcopy --add-gnu-debuglink=program.dbg program* 将指向调试信息的链接添加到剥离的可执行文件中 (调试时可执行程序会在当前目录或者.debug下查找携带调试信息的program.dbg文件)

携带调试信息的执行程序执行时，调试信息不会加载到内存中,除非 GDB 加载可执行文件。这意味着带有调试信息的可执行文件不会比没有调试信息的可执行文件运行得慢(一个常见的误解)

***
### GDB启动调试的方式

- *gdb --args programe_path arg1 garg2 ...*     直接调试可执行文件 
- *gdb programe_path core_dump_path* or *gdb -core core_dump_path*     调试可执行文件生成的核心转储文件
- *gdb attach pid_of_programe*     调试运行中的可执行文件
- *...*

***
### GDB常用调试命令

|命令|简写|描述|
|---|---|----|
|run|r|run [arg1 ...]启动调试可执行程序|
|layout||layout src或者ctrl+x+a查看当前断点源码|
|disassemble|disa|查看汇编代码|
|continue|c|继续运行程序知道遇到断点或者错误|
|finish|fin|运行直到当前函数完成|
|until||运行直到当前代码块结束或循环退出|
|frame|f|f [num]跳转当前调用堆栈帧到指定调用堆栈帧,无参数显示当前帧信息|
|backtrace|bt|显示调用堆栈,bt full显示完全堆栈|
|breakpoint|b|设置断点|
|print|p|打印变量值,只打印一次|
|display|disp|打印变量值,每次执行流程经过变量时打印|
|step|s|步进,遇到函数进入函数内部|
|next|n|单步,遇到函数作为单行执行跳过|
|info|i|i b/register/threads/locals,用于查看对应信息|
|info shared||查看加载的动态库信息|
|commands||commands break_num 断点命令列表,断点/观察点触发时可执行一系列命令|
|thread thread_num||切换线程,thread_num是info threads对应的编号|
|call syscall/function||调用系统函数或者编写的函数,例如 call system("env>/tmp/log") 查看运行时环境变量|

***
### GDB命令的一些细节

#### GDB断点

**1.设置断点**
- *break linenumber*  在行号处设置断点
- *break function*    在函数处设置断点
- *break filename:linenumber* 在文件指定行号处设置断点
- *break filename:function* 在文件指定函数处设置断点
- *break *address* 在内存地址处设置断点,一般用于没有调试信息的执行文件处设置断点
- *tbreak args* 临时断点,断点触发一次删除
- *rbreak filename:function_regex* 正则断点,匹配到regex的所有函数设置断点,例如 `rbreak file.c:.` 给文件file.c所有函数下断点

**2.设置条件断点:**
- *break ... if condition* 
  "..."指代前面设置断点的通用方式, condition可以是断点上下文中的变量例如 i == 1, 字符串其中可以使用strcmp来比较,**不过不建议使用**, 因为可能会影响调试时调用堆栈环境导致奔溃,建议使用gdb内置的函数. 如果condition在断点上下文无效,GDB禁止定义断点
- *break ... if $_streq(str, "xxx")*  字符串相等判断断点
- *break ... if $_caller_is("main"[, frame_num])* 函数调用者判断,frame_num如果设置代表指定的帧
- *break ... if $_memeq(buf1,buf2,length)* 内存比较
- *break ... if $_regex(str, regex)* 字符串正则比较

- *break ... -force-condition if condition 可能condition在断点处上下文无意义但未来有意义比如动态库的加载,使符号有意义,此情况的考虑下,则GDB强制定义断点

具体其他函数可以在gdb内调用 help function 查看

**3.设置观察点**
- *watch [-l location] expr [thread thread_id] [mask maskvalue]* 设置观察点,如果观察点expr或者值发生变化则GDB中断,[-l location]使GDB改为设置引用的内存为观察点
- *rwatch ...* 该观察点再内存或者值被读取时中断
- *awatch ...* 该观察点再内存读取或者写入时中断

当局部变量或者涉及的变量的表达式超出范围既当前执行离开块的时候,GDB自动删除观察点. 有技巧可以在程序如果设置断点,断点时自动设置观察点

**4.查看断点:**
- *info breakpoints*

**5.清除断点:**
- *clear linenumber* 清除行号处所有的断点
- *clear function* 清除函数入口处所有的断点
- *clear filename:linenumber/function* 清除指定文件行或函数处的所有断点
- *delete* 清除所有断点
- *delete* n 清除指定编号的断点 

**6.断点命令列表:**
- commands break_num 可用在相应的断点触发时,执行一系列的gdb命令,silent隐藏断点触发时打印的一些信息,结束设置commands设置end结束.断点命令的一个应用是补偿一个错误，以便您可以测试另一个错误
    ...
    10  int value;
    ...
    (gdb) b 10
    Breakpoint 6 at 0x40122f: file ...cpp, line 10.
    (gdb) commands 10
    (gdb) silent
    (gdb) set value = 200
    (gdb) c
    (gdb) end

***
##### 查看变量

显示变量的类型可以使用ptype简写pt,会直接显示变量的基础类型并忽略C++的using别名及typedef. whatis只是显示声明变量时的类型
查看变量的值使用print,简写p,格式是"print [/FMT] value",无参数/FMT则是以"最舒适的"显示状态打印值,print 无法打印一些std::string,或者宽字符,必须强转为char*类型才能显示
(/FMT: /o = octal,/x = hex, /d = decimal, /u	= unsigned decimal, /t = binary, /f = float, /a	= address, /c = char)
    ...
    uint32_t value;
    char mychar;
    QString str = "testing";
    ...
    (gdb) pt value
    type = unsigned int
    (gdb) whatis value
    type = uint32_t
    (gdb) p mychar
    $33 = 65 'A'
	(gdb) p /o mychar
	$34 = 0101
	(gdb) p /x mychar 
	$35 = 0x41
	(gdb) p /d mychar 
	$36 = 65
	(gdb) p /u mychar 
	$37 = 65
	(gdb) p /t mychar 
	$38 = 1000001
	(gdb) p /f mychar 
	$39 = 65
	(gdb) p /a mychar 
	$40 = 0x41
  (gdb) p str.toUtf8().constData()
  $41 = "testing"

***
##### 设置变量

set value = 200 可在调试过程中改变变量的值,用于调试其他逻辑代码走向,通常也可在断点触发时改变值,以此持久改变值
    ...
    10 int value;
    ...
    (gdb) b 10
    Breakpoint 6 at 0x40122f: file ...cpp, line 10.
    (gdb) commands 10
    (gdb) set value = 200
    (gdb) c
    (gdb) end

这样每次断点触发时,自动设置值,然后continue执行程序

#### GDB分离执行程序IO和GDB IO输出

gdb调试执行程序时,执行程序会输出大量IO扰乱调试界面或者gdb IO输出,可在调试时tty指定到其他终端的tty
    ...
    //终端1
    $ tty
    /dev/pts/6
    ...
    //终端2
    (gdb) tty /dev/pts/6
    (gdb) ...
    ...

#### 不退出GDB重新编译程序

不退出GDB调试,直接重新编译程序,然后在GDB中run/r重新运行GDB,GDB会注意到可执行程序文件更新重新加载新的二进制文件,至少比重新启动GDB调试快很多,在调试远程程序挂载时有一定的帮助

#### 向后执行程序

在调试程序的时候,往往会调试过快,发现错过了某些重点函数,GDB允许向后执行程序,撤销执行的指令,反向执行源代码之后,该代码所有的副作用都会撤销,内存和寄存器恢复到之前的状态,不过有时设备IO会撤销失败

- *record*
  激活反向执行命令记录
- *record stop*
  停止命令记录
- *reverse-continue [ignore-count]*
- *rc [ignore-count]*
  回到上次continue/c停止的地方
- *reverse-step [ignore-count]*
- *rs [ignore-count]*
  回到上次step停止的地方
- *reverse-next*
- *rn [ignore-count]*
  回到上次next停止的地方
- *reverse-finish [ignore-count]*
- *rfin [ignore-count]*
  回到上次函数跳出的地方

#### 远程调试时源码相对位置替换

有时本地或者远程调试的程序,源码文件位置不在原编译时的路径,导致gdb调试时找不到对应源码,一般有两种方式解决:
- 比较直接的方式就是在对应编译路径上,软连接或这直接mv原来的源码会到之前的位置,如果是远程的sshfs挂载的情况,比如编译目录是/home/xxxx/project/Coding/前缀,那就在远程电脑中执行 `mkdir -p /home/xxxx/project/Coding/ && sudo sshfs -oallow_other,default_permissions user@host:/home/xxxx/project/Coding/ /home/xxxx/project/Coding/`这样gdb也可以找到
- 另一种方式就是要知道gdb有个`source path:$cdir:$cwd`路径搜索规则,`$cdir`代表 compile path 编译路径, `$cwd`代表 current working directory 等于 gdb启动时的当前路径.
例如源码路径是 /home/xxxx/project/Coding/main.cpp,则搜索路径就是:

  - /home/xxxx/project/Coding/main.cpp
  - $cdir/home/xxxx/project/Coding/main.cpp
  - $cwd/home/xxxx/project/Coding/main.cpp
  - $cdir/main.cpp
  - $cwd/main.cpp

  调试信息记录的源码路径只会全路径拼接和filename拆分下来查找,不会部分拆分去查找.
  例如不会这么拼接, /home/xxxx/main.cpp or $cdir/Coding/main.cpp 去查找.
  如果原始文件的路径是这个 /home/xxxx/project/Coding/main.cpp
  迁移之后的文件在其他位置 /home/testing/xxx/Coding/main.cpp
  则可以在gdb内使用命令替换源码路径规则: 
  - set substitute-path from to
  - set substitute-path /home/xxxx/project/Coding/ /home/testing/xxx/Coding/
  - 或者直接在gdb内使用 directory /home/testing/xxx/Coding,则代表 `source path:/home/testing/xxx/Coding:$cdir:$cwd` 添加搜索路径,不过这样比较鸡肋如果文件路径有相对的且较多的情况比较麻烦,再次调用directory无参数,重置 回默认,如果需要查看`source path`, 需要gdb调试时调用 info source 查看路径信息

#### GDB中执行shell命令

- shell command-string
- !command-string
  调用标准shell($SHELL)来执行command-string,!command-string不需要空格
- pipe [command] | shell_command | [command] | shell_command
  执行command并输出给shell_command去使用

  例子:

  (gdb) p var
  $1 = {
  black = 144,
  red = 233,
  green = 377,
  blue = 610,
  white = 987
  }
  (gdb) pipe p var|wc
  7      19      80
  (gdb) |p var|wc -l
  7
  (gdb) p /x var
  $4 = {
  black = 0x90,
  red = 0xe9,
  green = 0x179,
  blue = 0x262,
  white = 0x3db
  }
  (gdb) ||grep red
  red => 0xe9,
  (gdb) | -d ! echo this contains a | char\n ! sed -e 's/|/PIPE/'
  this contains a PIPE char
  (gdb) | -d xxx echo this contains a | char!\n xxx sed -e 's/|/PIPE/'
  this contains a PIPE char!
  (gdb)