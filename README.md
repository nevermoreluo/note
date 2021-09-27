# Note
杂记

这里记录一些编程过程中的随想以及趣事






## Lua

### 为什么在早期的库里面 在luaL_openlibs之前要关闭gc
问题：  
最开始是在别人的代码中发现的这段逻辑  
这段gc看起来很高深莫测，似乎看不出道理来  
```
  lua_gc(L, LUA_GCSTOP, 0);  /* stop collector during initialization */
  luaL_openlibs(L);  /* open libraries */
  lua_gc(L, LUA_GCRESTART, 0);
```

原理：  

在搜索中发现最早的[lua5.1版本源码](https://github.com/lua/lua/blob/69ea087dff1daba25a2000dfb8f1883c17545b7a/lua.c#L332)中的就存在这段写法，
我们看到在luaL_openlibs之前强行关闭了gc，并在openlibs结束后开启  

关于这么做的用意，Roberto回答是[to reduce the GC overhead when creating large number of
objects that are not garbage](http://lua-users.org/lists/lua-l/2008-07/msg00690.html)
即在创建大量不需要垃圾回收对象时减少额外开销



结论：  
5.1和5.2是需要关掉gc的  
5.3开始已经没有类似的写法了

我们查看了我们正在使用的最新的lua5.4版本的源码，发现已经没有再使用类似的调用了，因此我们就不再沿用老的写法了



## Linux




### nfs server关闭导致异常
问题：
依赖nfs的服务如果nfs服务异常会引发一系列的异常，下面例举一些工作中遇到的情况
- `df -h` 命令卡顿长时间没有反馈
- 挂载了nfs路径的docker服务器restart长时间无效果，或者卡在Created阶段
- 挂载了nfs路径的docker服务器 无法关闭

原理：


结论：
先判断本机是否有mount过nfs的目录, 通过mount命令判断已经挂载的目录    
```bash
$ sudo mount -l -t nfs4
172.18.0.200:/games/internal on /games type nfs4 (rw,relatime,vers=4.0,rsize=524288,wsize=524288,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=172.22.156.55,local_lock=none,addr=172.18.0.200)

# 查看挂载的目录内容有没有
$ ls -l /games

# 通过umount命令， 将有问题的目录umount
$ sudo umount /games
```


## Debug

### Core dump （核心转储）
core dump是程序运行时，在进程收到某些信号而终止运行时，将此时进程地址空间的内容以及有关进程状态的其他信息写入一个磁盘文件。

对应会产生core dump的信号
| Signal  | Action | Comment                  |
|---------|--------|--------------------------|
| SIGQUIT | Core   | Quit from keyboard       |
| SIGILL  | Core   | Illegal Instruction      |
| SIGABRT | Core   | Abort signal from abort  |
| SIGSEGV | Core   | Invalid memory reference |
| SIGTRAP | Core   | Trace/breakpoint trap    |

我们可以通过使用gdb查core dump文件，最后崩溃时的信息，来进行debug  
为了更好的查看阅读core dump文件, linux下需要进行以下配置


#### 设置core文件生成位置， 默认为当前目录
可以修改/proc/sys/kernel/core_pattern，将core文件生成到指定目录下
```
mkdir /cores
echo "/cores/core.%t.%e.%p" | sudo tee /proc/sys/kernel/core_pattern
```
参数包含
```
%e  Executable name
%h  Hostname
%p  PID of dumped process
%s  Signal causing dump
%t  Time of dump
%u  UID
%g  GID
```

####  设置系统ulimit core size
可以通过`ulimit -c` 查看当前 core size， 默认为0，即不会生成core dump文件

- 通过`ulimit -c unlimited`设置当前会话中的ulimit， 退出或者新开会话会失效
- 为docker 设置ulimit， 默认会跟随 dockerd的配置，也可以在运行时指定 `docker run --ulimit core=-1 --security-opt seccomp=unconfined -v /cores:/cores <后续命令>`
- 在程序中直接设定，下面是一个例子

```cpp
#include <unistd.h>
#include <sys/time.h>
#include <sys/resource.h>
#include <stdio.h>
#define CORE_SIZE   -1
int main()
{
    struct rlimit rlmt;
    if (getrlimit(RLIMIT_CORE, &rlmt) == -1) {
        return -1; 
    }   
    printf("Before set rlimit CORE dump current is:%d, max is:%d\n", (int)rlmt.rlim_cur, (int)rlmt.rlim_max);

    rlmt.rlim_cur = (rlim_t)CORE_SIZE;
    rlmt.rlim_max  = (rlim_t)CORE_SIZE;

#if DEBUG
    // 主要是这句 设定 core size
    if (setrlimit(RLIMIT_CORE, &rlmt) == -1) {
        return -1; 
    }   
#endif

    if (getrlimit(RLIMIT_CORE, &rlmt) == -1) {
        return -1; 
    }   
    printf("After set rlimit CORE dump current is:%d, max is:%d\n", (int)rlmt.rlim_cur, (int)rlmt.rlim_max);

    /*测试非法内存，产生core文件*/
    int *ptr = NULL;
    *ptr = 10; 

    return 0;
}
```
由于我们大多数工程测试时都是跑在clion+wsl环境中的， 当clion启动wsl时会通过 wsl另外启动一个sh的壳导致我们在系统内部设置的ulimit失效，所以我们项目中选择在程序内设定。 通过`C:\Windows\system32\wsl.exe --distribution Ubuntu-18.04 --exec /bin/sh -c "ulimit -c"`可以验证  


#### 添加编译参数，在查看core dump文件时可以拿到更详细的信息
```
CMakeLists.txt
        add_definitions(-DDEBUG=true)
        add_definitions(-DRELEASE=false)

        # core dump config
        add_definitions("$ENV{CXXFLAGS} -O0 -g")
        SET(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -g")
```

#### 至此 当程序异常退出时，我们可以在debug环境下愉快的拿到core文件了

假设core文件为`/cores/core.1630405848.pixelpai.19`,我们就可以通过gdb解析对应的文件    
bt # 获取最后退出堆栈的详细信息  
frame 3 # 简写 f 3 切到第3个frame 并输出相关代码  
p value # 展示所在帧value对象的值  
up # 移到上一个帧  
down # 移到下一个帧  

```
gdb main /cores/core.1630405848.pixelpai.19
...
GNU gdb (Ubuntu 8.1.1-0ubuntu1) 8.1.1
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from cmake-build-debug/build/bin/main...done.
[New LWP 17765]
[New LWP 2336]
[New LWP 29510]
[New LWP 11379]
[New LWP 2335]
[New LWP 2334]
[New LWP 20475]
[New LWP 20476]
[New LWP 20477]

warning: Could not load shared library symbols for 2 libraries, e.g. ./cjson.so.
Use the "info sharedlibrary" command to see the complete listing.
Do you need "set solib-search-path" or "set sysroot"?
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Core was generated by `/home/never/work/pixelpai_server/cmake-build-debug/build/bin/main -props /home/'.
Program terminated with signal SIGABRT, Aborted.
#0  raise (sig=<optimized out>) at ../sysdeps/unix/sysv/linux/raise.c:50
50      ../sysdeps/unix/sysv/linux/raise.c: No such file or directory.
[Current thread is 1 (Thread 0x7fca28917700 (LWP 17765))]
(gdb) bt
#0  raise (sig=<optimized out>) at ../sysdeps/unix/sysv/linux/raise.c:50
#1  0x0000562abaf64e47 in console::handler (sig=11) at /home/never/work/pixelpai_server/src/server/main/console_linux.cpp:244
#2  <signal handler called>
#3  0x00007fca316e0108 in Sprite::getSpriteSerialize (this=0x7fca00a66d00) at /home/never/work/pixelpai_server/src/server/world/virtualworld/scene/sprite.cpp:595
#4  0x00007fca3169a87a in Scene::sendEnterSceneToAll (this=0x7fca0217ba60, actorId=1685436067) at /home/never/work/pixelpai_server/src/server/world/virtualworld/scene/scene.cpp:535
...
(gdb) frame 3
#3  0x00007fca316e0108 in Sprite::getSpriteSerialize (this=0x7fca00a66d00) at /home/never/work/pixelpai_server/src/server/world/virtualworld/scene/sprite.cpp:595
(gdb) p animationSptr
$1 = std::shared_ptr<IAnimation> (empty) = {get() = 0x0}
(gdb) p spriteSptr
$2 = std::shared_ptr<op_client::Sprite> (use count 1, weak count 0) = {get() = 0x7fca0d84cea0}
(gdb) up
#4  0x00007fca3169a87a in Scene::sendEnterSceneToAll (this=0x7fca0217ba60, actorId=1685436067) at /home/never/work/pixelpai_server/src/server/world/virtualworld/scene/scene.cpp:535
535         auto characterProto = pCharacter->getSpriteSerialize();
(gdb) down
#3  0x00007fca316e0108 in Sprite::getSpriteSerialize (this=0x7fca00a66d00) at /home/never/work/pixelpai_server/src/server/world/virtualworld/scene/sprite.cpp:595

```

#### 参考
https://ctring.github.io/blog/2021/how-to-get-core-dump-in-a-Docker-container/  
https://le.qun.ch/en/blog/core-dump-file-in-docker/  
https://zhuanlan.zhihu.com/p/24311785  
https://www.cnblogs.com/hazir/p/linxu_core_dump.html



## 性能分析  

### Linux Performance
![image](image/linux_performance.png)



### Strace
Strace是Linux环境下用于调试诊断应用程序调用systemcall的工具。

由于Strace只检测系统调用，因此Strace只是一个分析的侧面。  
例如：命令`perl -e 'while(1){}'`不会产生任何系统调用

系统调用包括以下几个方面file, process, network, signal, ipc, desc， 默认将检测所有系统调用即all

#### 使用Strace统计运行期间系统调用占比
```
$ sudo strace -f -c main
strace: Process 30879 detached
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 64.79    2.184606       35236        62         3 futex
 28.13    0.948503         566      1675         1 clock_nanosleep
  2.85    0.096019           8     12105           getsockopt
  1.35    0.045417           3     16768           mprotect
  0.82    0.027561           7      4002      3633 recvfrom
  0.65    0.021963           3      7108           lseek
 ...
------ ----------- ----------- --------- --------- ----------------
100.00    3.371847                 55680      5957 total
```

#### 使用Strace 记录程序运行期间的系统调用   
可以通过添加-tt记录调用时间戳，与程序日志对比可区分出程序各个阶段的系统调用  

 

```
$ sudo strace -f -tt -T -o /tmp/test/trace.log main

$ cat /tmp/test/trace.log
18721 21:02:31.936942 execve("/bin/bash", ["bash", "-c", "APOWO_SERVER_ROOT=/home/never/wo"...], 0x7ffef5c46408 /* 14 vars */) = 0 <0.000077>
18721 21:02:31.937082 brk(NULL)         = 0x563070e9c000 <0.000017>
18721 21:02:31.937133 arch_prctl(0x3001 /* ARCH_??? */, 0x7ffe20222180) = -1 EINVAL (Invalid argument) <0.000017>
18721 21:02:31.937183 access("/etc/ld.so.preload", R_OK) = -1 ENOENT (No such file or directory) <0.000019>


PID  调用时间 系统调用名称 返回 消耗时间  
18721 21:02:31.936942 execve("/bin/bash", ["bash", "-c", "APOWO_SERVER_ROOT=/home/never/wo"...], 0x7ffef5c46408 /* 14 vars */) = 0 <0.000077> 
```

#### 参考
https://linux.die.net/man/1/strace



### Perf
首先我们需要大致理解一下Perf的原理：  
每隔一个固定的时间，就在CPU上（每个核上都有）产生一个中断，在中断上看看，当前是哪个pid，哪个函数，然后给对应的pid和函数加一个统计值，这样，我们就知道CPU有百分几的时间在某个pid，或者某个函数上了
![image](image/perf.png)

即本质上是一种采样的模式，我们预期，运行时间越多的函数，被时钟中断击中的机会越大，从而推测，那个函数（或者pid等）的CPU占用率就越高。
`即得到的结果有时并不能完全依赖，往往需要根据实际业务做进一步分析。`

关于perf有很多命令，今天先举例`perf top`的使用

perf top -p <pid>
![image](image/perf_top.png)
图中可以看到在lua中traversestrongtable调用的最多，即可得知 卡顿主要问题出现在lua层对象在一个forloop内进行了比较密集的运算导致cpu持续占用，之后的就需要进一步查看业务代码了