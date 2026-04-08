# Chapter 4: Traps and system calls

## 概述

陷阱（trap）是从用户代码切换到内核代码的机制。有三种类型的陷阱：
1. **系统调用**：用户程序执行`ecall`指令请求内核服务
2. **异常**：用户程序执行了非法操作（如除以零、访问无效地址）
3. **设备中断**：设备发出信号需要注意（如磁盘完成读操作）

## 4.1 RISC-V陷阱机制

### 陷阱相关寄存器

| 寄存器 | 名称 | 用途 |
|-------|------|------|
| `stvec` | Supervisor Trap Vector | 陷阱处理程序的地址 |
| `sepc` | Supervisor Exception PC | 发生陷阱时的程序计数器 |
| `scause` | Supervisor Cause | 陷阱原因 |
| `stval` | Supervisor Trap Value | 附加信息（如出错的虚拟地址） |
| `sstatus` | Supervisor Status | 状态信息，包括中断使能、之前的权限模式 |

### sstatus寄存器字段

- **SIE**：Supervisor Interrupt Enable，监管者模式中断使能
- **SPIE**：Supervisor Previous Interrupt Enable，陷阱前的中断使能状态
- **SPP**：Supervisor Previous Privilege，陷阱前的权限模式（0=用户，1=监管者）

### 陷阱处理流程（硬件部分）

当发生陷阱时，RISC-V硬件：

1. 如果是中断，清除`sstatus.SIE`（禁用中断）
2. 保存`pc`到`sepc`
3. 保存当前`sstatus.SIE`到`sstatus.SPIE`
4. 保存当前权限模式到`sstatus.SPP`
5. 设置权限模式为监管者模式
6. 设置`pc`为`stvec`（陷阱处理程序地址）
7. 开始执行陷阱处理程序

### sret指令（从陷阱返回）

`sret`指令：
1. 将`pc`恢复为`sepc`
2. 将`sstatus.SIE`恢复为`sstatus.SPIE`
3. 将权限模式恢复为`sstatus.SPP`
4. 继续执行

## 4.2 从用户空间的陷阱

### xv6陷阱处理的挑战

xv6需要处理以下复杂性：
1. RISC-V硬件做的太少，软件必须做更多工作
2. 陷阱可以在任何时候发生，不能假设用户代码处于良好状态
3. 用户程序可能故意触发陷阱来攻击内核
4. 需要处理来自用户空间和内核空间的陷阱
5. 陷阱后需要能够恢复被中断的代码

### 完整的陷阱路径（用户空间→内核）

1. 用户代码执行`ecall`（或发生异常/中断）
2. RISC-V硬件执行陷阱动作（如上所述）
3. 跳转到`stvec`指向的`uservec`（kernel/trampoline.S:16）
4. `uservec`保存所有用户寄存器到`trapframe`
5. `uservec`调用`usertrap`（kernel/trap.c:37）
6. `usertrap`确定陷阱原因，处理它
7. `usertrap`调用`usertrapret`（kernel/trap.c:90）
8. `usertrapret`设置硬件寄存器以准备返回用户空间
9. `usertrapret`跳转到`userret`（kernel/trampoline.S:88）
10. `userret`从`trapframe`恢复寄存器
11. `userret`执行`sret`返回用户空间

### TRAMPOLINE和trapframe

- **TRAMPOLINE**：映射在每个进程页表顶部的一页
- 包含`uservec`和`userret`汇编代码
- `trapframe`：每个进程的一页，用于保存用户寄存器
- 两者都映射在内核和用户页表中

### trapframe内容

```
kernel_satp      // 内核页表
kernel_sp        // 内核栈
kernel_trap      // 内核陷阱处理函数（usertrap）
kernel_hartid    // 硬件线程ID
epc              // 保存的用户pc
ra               // 保存的用户寄存器
sp
gp
tp
t0
t1
t2
s0
s1
a0
a1
a2
a3
a4
a5
a6
a7
s2
s3
s4
s5
s6
s7
s8
s9
s10
s11
t3
t4
t5
t6
```

## 4.3 代码：调用系统调用

### 系统调用流程（用户侧）

1. 用户程序调用系统调用包装函数（如`fork()`）
2. 包装函数将系统调用号放入`a7`寄存器
3. 执行`ecall`指令
4. 内核返回后，包装函数返回结果

### 示例：user/usys.S（生成的）

```assembly
.global fork
fork:
 li a7, SYS_fork
 ecall
 ret
```

### 示例：user/ulib.c

```c
int fork(void) {
  return syscall(SYS_fork);
}
```

## 4.4 代码：系统调用参数

### 系统调用参数传递

- 系统调用号在`a7`中
- 参数在`a0`-`a5`中
- 结果在`a0`中返回

### 获取系统调用参数

`argint()`、`argaddr()`、`argfd()`函数从`trapframe`中提取系统调用参数：

```c
int argint(int n, int *ip) {
  *ip = myproc()->trapframe->a0 + n;
  return 0;
}
```

### 从用户空间复制数据

- `copyin()`：从用户空间复制到内核空间
- `copyout()`：从内核空间复制到用户空间
- 这两个函数必须处理用户虚拟地址到物理地址的转换

## 4.5 从内核空间的陷阱

### 内核陷阱设置

- 内核运行时，`stvec`指向`kernelvec`（kernel/kernelvec.S:1）
- 内核陷阱使用当前内核栈，不需要切换栈
- 内核陷阱保存寄存器到内核栈

### 内核陷阱路径

1. 发生陷阱（如中断）
2. 跳转到`kernelvec`
3. `kernelvec`保存寄存器到内核栈
4. 调用`kerneltrap`（kernel/trap.c:148）
5. `kerneltrap`处理陷阱
6. 返回到`kernelvec`
7. `kernelvec`恢复寄存器
8. 执行`sret`

### 内核陷阱的特殊性

- 内核已经在监管者模式，不需要改变权限
- 内核有自己的栈，不需要切换
- 内核代码是可信任的，不需要像用户陷阱那样谨慎

## 4.6 页面错误异常

### 页面错误类型

RISC-V定义了三种页面错误：
1. **加载页面错误**（Load page fault）：尝试读取无效地址
2. **存储/AMO页面错误**（Store/AMO page fault）：尝试写入无效地址
3. **指令页面错误**（Instruction page fault）：尝试从无效地址取指

### 页面错误信息

- `scause`指示页面错误类型
- `stval`包含出错的虚拟地址

### xv6的页面错误处理

xv6对页面错误的处理很简单：
- 如果来自用户空间：杀死进程
- 如果来自内核：恐慌（panic）

### 现实世界中的页面错误使用

现代操作系统使用页面错误实现高级功能：
- **写时复制（Copy-on-write）**：`fork`时共享内存，写入时才复制
- **按需分页（Demand paging）**：`exec`时不立即加载所有代码，访问时才加载
- **交换（Swapping）**：将不常用的页写入磁盘，需要时再读入
- **内存映射文件（Memory-mapped files）**：将文件映射到地址空间

## 4.7 现实世界

- 像xv6一样，真实的操作系统使用陷阱、异常和中断
- 真实操作系统更复杂地处理页面错误（COW、分页、交换）
- 系统调用开销是一个重要的性能考虑因素
- 一些CPU使用特殊寄存器（如x86的`sysenter`/`sysexit`）加速系统调用
- 微内核在用户空间运行更多服务，需要更多IPC陷阱
- 中断处理通常分为两部分：快速的"top half"和较慢的"bottom half"

## 4.8 练习

（见书籍中的练习题目）
