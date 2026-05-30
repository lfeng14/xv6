# Chapter 7: Scheduling

## 概述

调度器是操作系统的核心组件，负责决定下一个运行哪个进程。xv6使用简单的轮转（round-robin）调度算法。

## 7.1 多路复用（Multiplexing）

### 上下文切换

xv6通过以下机制在进程之间切换：
1. 时钟中断
2. 将CPU从用户进程切换到内核
3. 调用`sched()`
4. `sched()`调用`swtch()`切换到调度器线程
5. 调度器选择另一个进程
6. 调用`swtch()`切换到新进程的内核线程
7. 返回到用户空间

### 进程状态

| 状态 | 含义 |
|-----|------|
| `UNUSED` | 进程槽未使用 |
| `SLEEPING` | 进程睡眠等待事件 |
| `RUNNABLE` | 进程准备运行 |
| `RUNNING` | 进程正在运行 |
| `ZOMBIE` | 进程已退出，等待父进程收集 |

### 每个CPU的调度器

每个CPU都有自己的调度器线程，它：
- 循环查找可运行的进程
- 切换到选中的进程
- 当进程让出CPU时，调度器继续查找下一个进程

## 7.2 代码：上下文切换

### 上下文（Context）

上下文保存CPU寄存器状态：

```c
struct context {
  uint64 ra;  // 返回地址
  uint64 sp;  // 栈指针
  uint64 s0;  // 保存的寄存器
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```

### swtch()

`swtch()`（kernel/swtch.S:1）执行实际的上下文切换：

```assembly
swtch:
  sd ra, 0(a0)
  sd sp, 8(a0)
  sd s0, 16(a0)
  sd s1, 24(a0)
  sd s2, 32(a0)
  sd s3, 40(a0)
  sd s4, 48(a0)
  sd s5, 56(a0)
  sd s6, 64(a0)
  sd s7, 72(a0)
  sd s8, 80(a0)
  sd s9, 88(a0)
  sd s10, 96(a0)
  sd s11, 104(a0)

  ld ra, 0(a1)
  ld sp, 8(a1)
  ld s0, 16(a1)
  ld s1, 24(a1)
  ld s2, 32(a1)
  ld s3, 40(a1)
  ld s4, 48(a1)
  ld s5, 56(a1)
  ld s6, 64(a1)
  ld s7, 72(a1)
  ld s8, 80(a1)
  ld s9, 88(a1)
  ld s10, 96(a1)
  ld s11, 104(a1)

  ret
```

`swtch(a0, a1)`：
- 保存当前寄存器到`*a0`（旧上下文）
- 从`*a1`（新上下文）加载寄存器
- 返回（返回到新上下文的`ra`）

## 7.3 代码：调度

### sched()

`sched()`（kernel/proc.c:471）执行从进程到调度器的切换：

```c
void sched(void) {
  int intena;

  if(!holding(&p->lock))
    panic("sched p->lock");
  if(mycpu()->noff != 1)
    panic("sched locks");
  if(p->state == RUNNING)
    panic("sched running");
  if(intr_get())
    panic("sched interruptible");

  intena = mycpu()->intena;
  swtch(&p->context, &mycpu()->context);
  mycpu()->intena = intena;
}
```

### scheduler()

`scheduler()`（kernel/proc.c:414）是每个CPU的调度器循环：

```c
void scheduler(void) {
  struct proc *p;
  struct cpu *c = mycpu();

  c->proc = 0;
  for(;;){
    // 避免死锁
    intr_on();

    acquire(&ptable.lock);

    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->state != RUNNABLE)
        continue;

      // 切换到选中的进程
      c->proc = p;
      uvmswitch(p);
      p->state = RUNNING;

      swtch(&c->context, &p->context);

      // 进程现在已返回
      c->proc = 0;
    }

    release(&ptable.lock);
  }
}
```

### yield()

`yield()`（kernel/proc.c:499）让进程自愿放弃CPU：

```c
void yield(void) {
  struct proc *p = myproc();

  acquire(&p->lock);
  p->state = RUNNABLE;
  sched();
  release(&p->lock);
}
```

时钟中断处理程序调用`yield()`实现抢占式调度。

## 7.4 代码：mycpu和myproc

### mycpu()

`mycpu()`（kernel/proc.c:72）返回当前CPU的`struct cpu`指针：

```c
struct cpu* mycpu(void) {
  return &cpus[cpuid()];
}
```

### myproc()

`myproc()`（kernel/proc.c:90）返回当前CPU上运行的进程：

```c
struct proc* myproc(void) {
  struct cpu *c;
  struct proc *p;

  push_off();
  c = mycpu();
  p = c->proc;
  pop_off();
  return p;
}
```

注意：`push_off()`/`pop_off()`避免在读取`c->proc`时被中断干扰。

## 7.5 睡眠和唤醒（Sleep and wakeup）

### 问题：等待事件

进程经常需要等待事件发生：
- 等待磁盘I/O完成
- 等待子进程退出
- 等待管道中有数据

### 忙等待（Busy waiting）

```c
// 不好的做法！
while(pipe->readable == 0)
  ;
```

问题：浪费CPU周期。

### 睡眠和唤醒原语

- `sleep(chan, lock)`：让进程在`chan`上睡眠，释放`lock`
- `wakeup(chan)`：唤醒所有在`chan`上睡眠的进程

### sleep/wakeup协议

调用`sleep()`的进程必须遵循：
1. 获取保护条件的锁
2. 检查条件是否为真，如果是则返回
3. 调用`sleep(chan, lock)`
4. 循环（回到步骤2）

调用`wakeup()`的进程必须：
1. 获取保护条件的锁
2. 更新条件
3. 调用`wakeup(chan)`
4. 释放锁

## 7.6 代码：睡眠和唤醒

### sleep()

```c
void sleep(void *chan, struct spinlock *lk) {
  struct proc *p = myproc();

  // 必须持有lk
  if(lk != &p->lock){
    acquire(&p->lock);
    release(lk);
  }

  // 进入睡眠
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // 醒来
  p->chan = 0;

  // 恢复原始锁
  if(lk != &p->lock){
    release(&p->lock);
    acquire(lk);
  }
}
```

### wakeup()

```c
void wakeup(void *chan) {
  struct proc *p;

  acquire(&ptable.lock);
  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
    if(p->state == SLEEPING && p->chan == chan){
      p->state = RUNNABLE;
    }
  }
  release(&ptable.lock);
}
```

### wakeup1()

`wakeup1()`是一个内部版本，假设已经持有`ptable.lock`。

## 7.7 代码：管道

### 管道实现中的sleep/wakeup

管道使用sleep/wakeup处理读写：

```c
int pipewrite(struct pipe *p, char *addr, int n) {
  // ...
  for(int i = 0; i < n; i++){
    while(p->nwrite == p->nread + PIPESIZE){
      // 管道满，等待
      wakeup(&p->nread);
      sleep(&p->nwrite, &p->lock);
    }
    p->data[p->nwrite % PIPESIZE] = addr[i];
    p->nwrite++;
  }
  wakeup(&p->nread);
  // ...
}

int piperead(struct pipe *p, char *addr, int n) {
  // ...
  while(p->nread == p->nwrite && p->writeopen){
    // 管道空，等待
    sleep(&p->nread, &p->lock);
  }
  for(int i = 0; i < n; i++){
    if(p->nread == p->nwrite)
      break;
    addr[i] = p->data[p->nread % PIPESIZE];
    p->nread++;
  }
  wakeup(&p->nwrite);
  // ...
}
```

## 7.8 代码：Wait, exit, and kill

### exit()

`exit()`终止当前进程：

```c
void exit(int status) {
  struct proc *curproc = myproc();
  struct proc *p;
  int fd;

  // 关闭所有打开的文件
  for(fd = 0; fd < NOFILE; fd++){
    if(curproc->ofile[fd]){
      fileclose(curproc->ofile[fd]);
      curproc->ofile[fd] = 0;
    }
  }

  // 释放当前目录
  begin_op();
  iput(curproc->cwd);
  end_op();
  curproc->cwd = 0;

  acquire(&ptable.lock);

  // 唤醒父进程
  wakeup(curproc->parent);

  // 将子进程交给init
  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
    if(p->parent == curproc){
      p->parent = initproc;
      wakeup(initproc);
    }
  }

  // 进入僵尸状态
  curproc->xstate = status;
  curproc->state = ZOMBIE;

  sched();
  panic("zombie exit");
}
```

### wait()

`wait()`等待子进程退出：

```c
int wait(int *status) {
  struct proc *curproc = myproc();
  struct proc *p;
  int havekids, pid;

  acquire(&ptable.lock);
  for(;;){
    // 查找僵尸子进程
    havekids = 0;
    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->parent == curproc){
        havekids = 1;
        if(p->state == ZOMBIE){
          // 找到一个僵尸
          pid = p->pid;
          if(status)
            *status = p->xstate;
          p->state = UNUSED;
          p->pid = 0;
          p->parent = 0;
          p->name[0] = 0;
          p->killed = 0;
          release(&ptable.lock);
          return pid;
        }
      }
    }

    // 没有子进程了
    if(!havekids || curproc->killed){
      release(&ptable.lock);
      return -1;
    }

    // 等待子进程退出
    sleep(curproc, &ptable.lock);
  }
}
```

### kill()

`kill()`向进程发送信号：

```c
int kill(int pid) {
  struct proc *p;
  acquire(&ptable.lock);

  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
    if(p->pid == pid){
      p->killed = 1;
      // 如果目标正在睡眠，唤醒它
      if(p->state == SLEEPING)
        p->state = RUNNABLE;
      release(&ptable.lock);
      return 0;
    }
  }

  release(&ptable.lock);
  return -1;
}
```

## 7.9 进程锁

### 锁的层次

进程相关的锁：
1. `ptable.lock`：保护进程表和状态转换
2. `p->lock`：保护单个进程的字段

### 锁顺序

必须先获取`ptable.lock`，然后获取`p->lock`。

### 无锁访问

- `myproc()`在禁用中断时返回`proc`指针，安全
- `mycpu()`返回CPU指针，安全

## 7.10 现实世界

### 调度算法

真实操作系统使用更复杂的调度算法：
- **多级反馈队列（MLFQ）**：根据历史行为调整优先级
- **完全公平调度器（CFS）**：Linux使用的红黑树调度器
- **实时调度**：保证截止期限
- **调度器激活（Scheduler activations）**：用户级线程与内核调度器协作

### 同步原语

真实系统提供更丰富的同步原语：
- POSIX信号量（semaphores）
- POSIX条件变量（condition variables）
- 读写锁（reader-writer locks）
- Futexes（快速用户空间互斥锁）

### 其他主题

- **处理器亲和性（Processor affinity）**：尝试让进程在同一CPU上运行
- **负载均衡（Load balancing）**：在CPU之间移动进程以平衡负载
- **节能调度（Energy-aware scheduling）**：降低功耗
- **组调度（Gang scheduling）**：同时调度相关进程组

## 7.11 练习

（见书籍中的练习题目）
