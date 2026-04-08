# Chapter 6: Locking

## 概述

锁是一种同步机制，用于保护共享数据，防止多个执行线程同时访问和修改数据导致的竞争条件（race condition）。

## 6.1 竞争条件（Races）

### 竞争条件示例

考虑两个CPU同时调用`kalloc()`的情况：

```c
void kfree(void *p) {
  struct run *r;
  r = (struct run*)p;
  r->next = kmem.freelist;  // 1: 读取
  kmem.freelist = r;         // 2: 写入
}
```

如果两个CPU同时执行：
1. CPU1读取`kmem.freelist`
2. CPU2读取`kmem.freelist`
3. CPU1设置`kmem.freelist`指向它的页
4. CPU2设置`kmem.freelist`指向它的页

结果：CPU1的页丢失了！

### 临界区（Critical section）

临界区是访问共享数据的代码序列，必须原子地执行。

### 锁的作用

锁确保一次只有一个CPU可以执行临界区代码。

### 为什么不关闭中断？

关闭中断对于单CPU有效，但对于多CPU无效：
- 两个CPU可以同时在临界区
- 长时间关闭中断会延迟设备中断处理

## 6.2 代码：锁

### 自旋锁（Spinlock）

xv6使用自旋锁作为基本锁类型：

```c
struct spinlock {
  uint locked;        // 锁是否被持有？
  char *name;         // 锁名称（用于调试）
  struct cpu *cpu;    // 持有锁的CPU
};
```

### RISC-V原子指令

RISC-V提供`amoswap.w.aq`（原子交换字，获取语义）：
- 原子地读取内存位置的值
- 写入新值
- 返回旧值
- `aq`：获取语义（acquire），确保后续读写不能重排到此操作之前

### acquire()实现

```c
void acquire(struct spinlock *lk) {
  push_off();         // 禁用中断
  if(holding(lk))
    panic("acquire");

  // 原子地交换：如果旧值是0，我们获取了锁
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0)
    ;

  __sync_synchronize();  // 内存屏障
  lk->cpu = mycpu();
}
```

### release()实现

```c
void release(struct spinlock *lk) {
  if(!holding(lk))
    panic("release");

  lk->cpu = 0;
  __sync_synchronize();  // 内存屏障

  // 原子地释放锁
  __sync_lock_release(&lk->locked);

  pop_off();  // 恢复中断
}
```

## 6.3 代码：使用锁

### xv6中的锁使用示例

| 锁 | 保护的内容 |
|---|-----------|
| `kmem.lock` | 空闲内存链表 |
| `pid_lock` | 下一个PID |
| `p->lock` | 进程状态 |
| `tickslock` | 时钟ticks |
| `bcache.lock` | 缓冲区缓存 |
| `cons.lock` | 控制台输入 |
| `uart_tx_lock` | UART发送 |

### 锁的粒度

- **粗粒度锁（Coarse-grained）**：一个锁保护大量数据
  - 优点：简单，锁开销小
  - 缺点：并发度低
- **细粒度锁（Fine-grained）**：多个锁分别保护不同数据
  - 优点：并发度高
  - 缺点：复杂，锁开销大

## 6.4 死锁和锁排序

### 死锁示例

考虑两个锁`L1`和`L2`：
1. CPU1获取`L1`
2. CPU2获取`L2`
3. CPU1尝试获取`L2`（等待）
4. CPU2尝试获取`L1`（等待）

两个CPU都永远等待！

### 锁排序（Lock ordering）

解决死锁的方法：始终以相同的顺序获取锁。

### xv6中的锁顺序

xv6定义了锁获取的顺序：
- `bcache.lock`在`ide_lock`之前
- `p->lock`在`ptable.lock`之前
- 等等...

## 6.5 可重入锁（Re-entrant locks）

### 可重入锁

可重入锁允许同一CPU多次获取同一把锁。

### xv6不使用可重入锁

xv6故意不使用可重入锁：
- 可重入锁使代码更复杂
- 更容易引入bug
- 如果需要，最好重构代码避免递归获取锁

## 6.6 锁和中断处理程序

### 中断和锁的交互

考虑一个场景：
- 进程持有锁`L`
- 时钟中断发生
- 中断处理程序也尝试获取`L`
- 死锁！

### 解决方法

`acquire()`在获取锁之前禁用中断，`release()`在释放锁之后恢复中断。

### push_off()和pop_off()

```c
void push_off(void) {
  int old = splget();  // 保存当前中断状态
  splhigh();            // 禁用中断
  if(mycpu()->noff == 0)
    mycpu()->intena = old;
  mycpu()->noff += 1;
}

void pop_off(void) {
  struct cpu *c = mycpu();
  if(splget() == 1)
    panic("pop_off - interruptible");
  c->noff -= 1;
  if(c->noff == 0 && c->intena)
    spl0();  // 恢复中断
}
```

使用计数器支持嵌套的`acquire()`/`release()`。

## 6.7 指令和内存排序

### 编译器和CPU重排序

编译器和CPU可能会重排序指令以提高性能：
- 编译器：编译时重排序
- CPU：运行时重排序

### 内存屏障（Memory barriers）

`acquire()`和`release()`使用内存屏障确保正确的顺序：
- `__sync_synchronize()`：全内存屏障
- 确保屏障前的内存操作在屏障后的操作之前完成

### 获取-释放语义（Acquire-Release semantics）

- **获取（Acquire）**：`acquire()`后的所有读写不能重排到`acquire()`之前
- **释放（Release）**：`release()`前的所有读写不能重排到`release()`之后

## 6.8 睡眠锁（Sleep locks）

### 自旋锁的局限性

自旋锁在等待时会忙等待（spin），浪费CPU周期。

### 睡眠锁

睡眠锁让进程在等待锁时睡眠（让出CPU）：

```c
struct sleeplock {
  uint locked;
  struct spinlock lk;
  char *name;
  int pid;
};
```

### acquiresleep()和releasesleep()

```c
void acquiresleep(struct sleeplock *lk) {
  acquire(&lk->lk);
  while(lk->locked) {
    sleep(lk, &lk->lk);
  }
  lk->locked = 1;
  lk->pid = myproc()->pid;
  release(&lk->lk);
}

void releasesleep(struct sleeplock *lk) {
  acquire(&lk->lk);
  lk->locked = 0;
  lk->pid = 0;
  wakeup(lk);
  release(&lk->lk);
}
```

### 睡眠锁 vs 自旋锁

| 特性 | 自旋锁 | 睡眠锁 |
|-----|-------|-------|
| 等待方式 | 忙等待 | 睡眠 |
| CPU使用 | 浪费 | 让出 |
| 持有时间 | 短 | 可以长 |
| 中断安全 | 是 | 否 |
| 可以在等待中切换进程 | 否 | 是 |

## 6.9 现实世界

- 真实操作系统使用各种同步原语：
  - 读写锁（reader-writer locks）
  - 信号量（semaphores）
  - 条件变量（condition variables）
  -  futexes（快速用户空间互斥锁）
- 无锁数据结构（Lock-free data structures）：使用原子操作而非锁
- 事务内存（Transactional memory）：用事务替代锁
- 真实系统避免长时间持有锁
- 性能分析工具帮助找出锁竞争（lock contention）
- 死锁检测和避免是研究的活跃领域

## 6.10 练习

（见书籍中的练习题目）
