---
layout: post
title: "「APP性能系列」Android Native Crash"
subtitle: 'Android应用 APP性能'
date:       2018-07-12
author: "Wangxiong"
header-style: text
tags:
  - Android
  - APP
---

与Java平台不同，C/C++没有一个通用的异常处理接口，在C层，CPU通过异常中断的方式，触发异常处理流程，不同的处理器有不同异常中断类型和中断处理方式，linux把这些中断处理，统一为信号量，每一种异常都有一个对应的信号，可以注册回调函数进行处理需要关注的信号量。

| 信号量       | 含义                                                         |
| :----------- | ------------------------------------------------------------ |
| SIGHUP 1     | 终端连接结束时发出(不管正常或非正常)                         |
| SIGINT 2     | 程序终止(例如Ctrl-C)                                         |
| SIGQUIT 3    | 程序退出(Ctrl-\ )                                            |
| SIGILL 4     | 执行了非法指令，或者试图执行数据段，堆栈溢出                 |
| SIGTRAP 5    | 断点时产生，由debugger使用                                   |
| SIGABRT 6    | 调用abort函数生成的信号，表示程序异常                        |
| SIGIOT 6     | 同上，更全，IO异常也会发出                                   |
| SIGBUS 7     | 非法地址，包括内存地址对齐出错，比如访问一个4字节的数, 但其地址不是4的倍数 |
| SIGFPE 8     | 计算错误，比如除0、溢出                                      |
| SIGKILL 9    | 强制结束程序，具有最高优先级，本信号不能被阻塞、处理和忽略   |
| SIGUSR1 10   | 未使用，保留                                                 |
| SIGSEGV 11   | 非法内存操作，与SIGBUS不同，他是对合法地址的非法访问，比如访问没有读权限的内存，向没有写权限的地址写数据 |
| SIGUSR2 12   | 未使用，保留                                                 |
| SIGPIPE 13   | 管道破裂，通常在进程间通信产生                               |
| SIGALRM 14   | 定时信号,                                                    |
| SIGTERM 15   | 结束程序，类似温和的SIGKILL，可被阻塞和处理。通常程序如果终止不了，才会尝试SIGKILL |
| SIGSTKFLT 16 | 协处理器堆栈错误                                             |
| SIGCHLD 17   | 子进程结束时, 父进程会收到这个信号。                         |
| SIGCONT 18   | 让一个停止的进程继续执行                                     |
| SIGSTOP 19   | 停止进程,本信号不能被阻塞,处理或忽略                         |
| SIGTSTP 20   | 停止进程,但该信号可以被处理和忽略                            |
| SIGTTIN 21   | 当后台作业要从用户终端读数据时, 该作业中的所有进程会收到SIGTTIN信号 |
| SIGTTOU 22   | 类似于SIGTTIN, 但在写终端时收到                              |
| SIGURG 23    | 有紧急数据或out-of-band数据到达socket时产生                  |
| SIGXCPU 24   | 超过CPU时间资源限制时发出                                    |
| SIGXFSZ 25   | 当进程企图扩大文件以至于超过文件大小资源限制                 |
| SIGVTALRM 26 | 虚拟时钟信号. 类似于SIGALRM, 但是计算的是该进程占用的CPU时间 |
| SIGPROF 27   | 类似于SIGALRM/SIGVTALRM, 但包括该进程用的CPU时间以及系统调用的时间 |
| SIGWINCH 28  | 窗口大小改变时发出                                           |
| SIGIO 29     | 文件描述符准备就绪, 可以开始进行输入/输出操作                |
| SIGPOLL 29   | 同上，别称                                                   |
| SIGPWR 30    | 电源异常                                                     |
| SIGSYS 31    | 非法的系统调用                                               |

主要关注SIGILL, SIGABRT, SIGBUS, SIGFPE, SIGSEGV, SIGSTKFLT, SIGSYS这几个信号量。

