# Chapter 2: Operating system organization

## 概述

本章介绍操作系统组织的关键要求：隔离性（isolation）、多路复用（multiplexing）和交互（interaction）。xv6使用经典的Unix设计方法实现这些目标，将操作系统组织为一个内核，这是一个特殊的程序，为运行中的进程提供服务。

## 2.1 抽象物理资源

操作系统的一个关键目标是在多个程序之间抽象硬件资源。例如：
- 文件系统抽象磁盘，使应用程序不必处理磁盘的复杂性
- 虚拟内存抽象物理内存
- 进程抽象CPU

## 2.2 用户模式、监管者模式和系统调用

### CPU权限模式

RISC-V CPU有三种权限模式，但xv6只使用两种：
- **监管者模式（Supervisor mode）**：可以执行特权指令（如禁用中断、读写页表寄存器等）
- **用户模式（User mode）**：只能执行非特权指令

### 系统调用机制

- 进程通过`ecall`指令从用户模式切换到监管者模式
- `ecall`指令将程序计数器（PC）改为内核定义的入口点
- 内核验证系统调用参数，决定是否允许执行
- 内核执行系统调用，然后使用`sret`指令返回用户模式

## 2.3 内核组织

### 单体内核（Monolithic kernel）

xv6采用单体内核设计：
- 整个操作系统在内核中，以监管者模式运行
- 所有系统调用的实现都以监管者模式运行
- 优点：简单，不同内核部分容易协作
- 缺点：内核不同部分之间的接口复杂，容易出错

### 微内核（Microkernel）

另一种设计是微内核：
- 大部分操作系统作为用户级进程运行
- 内核更小，提供消息传递等基本机制
- 优点：更好的隔离性
- 缺点：性能开销较大

## 2.4 代码：xv6组织

### xv6源代码文件

| 文件 | 描述 |
|-----|------|
| `kernel/param.h` | 系统参数定义 |
| `kernel/types.h` | 类型定义 |
| `kernel/defs.h` | 内核函数声明 |
| `kernel/memlayout.h` | 内存布局 |
| `kernel/riscv.h` | RISC-V相关定义 |
| `kernel/spinlock.h` | 自旋锁定义 |
| `kernel/sleeplock.h` | 睡眠锁定义 |
| `kernel/buf.h` | 块缓冲区定义 |
| `kernel/file.h` | 文件定义 |
| `kernel/proc.h` | 进程定义 |
| `kernel/start.c` | 启动代码 |
| `kernel/entry.S` | 入口点汇编 |
| `kernel/kalloc.c` | 物理内存分配器 |
| `kernel/vm.c` | 页表和虚拟内存 |
| `kernel/trap.c` | 陷阱和中断处理 |
| `kernel/syscall.c` | 系统调用分派 |
| `kernel/sysproc.c` | 进程相关系统调用 |
| `kernel/sysfile.c` | 文件相关系统调用 |
| `kernel/proc.c` | 进程管理 |
| `kernel/sched.c` | 调度器 |
| `kernel/spinlock.c` | 自旋锁实现 |
| `kernel/sleeplock.c` | 睡眠锁实现 |
| `kernel/bio.c` | 块I/O和缓冲区缓存 |
| `kernel/fs.c` | 文件系统 |
| `kernel/log.c` | 日志层 |
| `kernel/pipe.c` | 管道 |
| `kernel/file.c` | 文件描述符 |
| `kernel/console.c` | 控制台 |
| `kernel/uart.c` | 串口驱动 |
| `kernel/kernelvec.S` | 内核陷阱向量 |
| `kernel/trampoline.S` | 跳板代码 |
| `kernel/swtch.S` | 上下文切换 |
| `user/ulib.c` | 用户库 |
| `user/printf.c` | 用户态printf |
| `user/usys.S` | 用户系统调用存根 |

## 2.5 进程概述

### 进程状态

每个xv6进程在以下状态之间转换：
- **RUNNING**：正在CPU上执行
- **RUNNABLE**：等待在CPU上运行
- **SLEEPING**：等待事件（如I/O完成）
- **UNUSED**：未使用/已退出

### 进程结构（struct proc）

每个进程在内核中有一个`struct proc`结构，包含：
- 进程状态
- 内核栈
- 陷阱帧（trapframe）
- 页表
- PID
- 父进程
- 打开的文件
- 当前目录

## 2.6 代码：启动xv6，第一个进程和系统调用

### 启动流程

1. RISC-V计算机启动时，运行存储在ROM中的引导加载程序
2. 引导加载程序将xv6内核加载到内存（物理地址0x80000000）
3. CPU在监管者模式下从`_entry`（kernel/entry.S:1）开始执行
4. `_entry`设置栈，然后调用`start`（kernel/start.c:19）
5. `start`配置RISC-V：设置`mstatus`寄存器使`mret`切换到监管者模式，设置`mtvec`寄存器指向异常处理，禁用分页，启用时钟中断
6. `start`调用`main`（kernel/main.c:1）
7. `main`初始化设备和子系统
8. `main`调用`userinit`（kernel/proc.c:212）创建第一个进程
9. 第一个进程执行`initcode.S`，调用`exec`运行`/init`
10. `init`（user/init.c:17）创建一个控制台设备文件，然后打开它作为stdin、stdout、stderr
11. `init`启动shell

## 2.7 安全模型

xv6假设用户程序是恶意的，可能：
- 试图破坏内核或其他进程
- 试图访问它们不应该访问的内存
- 试图滥用系统调用

内核必须假设用户空间是不可信任的，必须验证所有系统调用参数。

## 2.8 现实世界

- 大多数操作系统（如Linux、BSD）使用单体内核设计
- 隔离是一个持续的挑战
- 现代CPU支持比仅用户/监管者模式更多的硬件功能，用于实现更强大的安全机制

## 2.9 练习

（见书籍中的练习题目）
