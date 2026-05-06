下面我给你把这段 xv6 的 **Locking** 章节梳理成“**大纲 + 重点 + 易考点**”三个层次，方便你复习。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)

## 一、章节大纲

1. **为什么需要锁**
   - 多核并发、线程切换、中断都会让执行交错。
   - 共享数据结构可能被多个 CPU 同时读写，导致错误。 [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)

2. **race（竞态）是什么**
   - 当同一内存位置被并发访问，且至少有一次写入，就可能发生 race。
   - race 常导致“丢失更新”或“读到半更新的数据”。 [clownote.github](https://clownote.github.io/2021/04/06/xv6/Xv6-Locking/)

3. **锁如何工作**
   - 锁提供互斥：同一时刻只能有一个 CPU 持有锁。
   - 锁保护的是“数据 + 相关不变量”，不是单纯保护变量本身。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)

4. **xv6 中锁的实现**
   - 主要是 **spinlock**。
   - 通过原子指令实现 acquire/release，RISC-V 上对应 `amoswap` 一类原子操作。 [clownote.github](https://clownote.github.io/2021/04/06/xv6/Xv6-Locking/)

5. **怎么正确使用锁**
   - 什么时候需要锁。
   - 锁粒度：粗粒度 vs 细粒度。
   - 死锁与锁顺序。
   - 中断处理与锁。
   - 内存/指令重排序问题。 [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)

6. **sleep lock**
   - 用于长时间持锁的场景。
   - 等待时可以睡眠，不会像 spinlock 那样一直空转浪费 CPU。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)

7. **现实中的锁问题**
   - 锁竞争、性能、锁自由数据结构、Pthreads 等。 [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)

***

## 二、核心概念重点

### 1. Concurrency（并发）
并发不只是多核并行，也包括单核上的线程切换和中断打断。也就是说，只要多个执行流交错，就属于并发问题。 [clownote.github](https://clownote.github.io/2021/04/06/xv6/Xv6-Locking/)

### 2. Race（竞态）
race 的定义很关键：**同一内存位置被并发访问，且至少一个是写**。  
典型后果是：
- **lost update**：两个 CPU 同时更新，最后只有一个更新保留下来；
- **读到不完整状态**：数据结构正在修改中，被别的 CPU 读到了“半成品”。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)

### 3. Invariant（不变量）
锁保护的不是“某一个变量”，而是这个数据结构必须始终满足的性质。  
例如链表的不变量是：
- `list` 必须指向第一个结点；
- 每个结点的 `next` 必须正确指向下一个结点。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)

### 4. Critical section（临界区）
`acquire` 和 `release` 之间的代码叫临界区。  
xv6 的思路是：**只要共享数据相关操作都放进同一把锁的临界区，就能保证互斥和不变量。** [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)

***

## 三、spinlock 机制重点

### 1. 为什么普通 `if(lock==0) lock=1` 不行
这段代码看起来能用，但在多核上会出问题：两个 CPU 可能同时看到锁是 0，然后都把它设为 1。  
结果就是**两个人都以为自己拿到了锁**，互斥失效。 [clownote.github](https://clownote.github.io/2021/04/06/xv6/Xv6-Locking/)

### 2. 原子操作的作用
xv6 用原子指令保证“检查锁”和“设置锁”必须作为一个不可分割操作完成。  
这就是 spinlock 能成立的根本原因。 [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)

### 3. acquire / release 的要点
- `acquire`：循环尝试拿锁，拿不到就一直“spin”。
- `release`：原子地释放锁。 [clownote.github](https://clownote.github.io/2021/04/06/xv6/Xv6-Locking/)

***

## 四、锁使用原则

### 1. 什么时候必须上锁
只要一个变量可能被一个 CPU 写，而另一个 CPU 同时读/写，就应该加锁。  
如果一个不变量涉及多个内存位置，这些位置通常也要由**同一把锁**保护。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)

### 2. 不要锁太多
锁越多，并行度可能越高，但设计复杂度也越高。  
锁太粗会让并发被串行化；锁太细又容易增加死锁和实现难度。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)

### 3. 锁粒度的例子
- `kalloc`：一个 free list + 一把锁，简单但竞争大。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)
- 文件系统：对不同文件使用不同锁，允许更多并行。 [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)

***

## 五、死锁重点

### 1. 死锁怎么发生
如果两个代码路径需要锁 A 和 B，但一个先拿 A 再拿 B，另一个先拿 B 再拿 A，就可能出现：
- T1 拿 A 等 B；
- T2 拿 B 等 A；
- 两边都永远等下去。 [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)

### 2. 解决办法
所有代码路径必须遵守**统一的锁顺序**。  
在 xv6 里，这个顺序甚至算是函数接口约束的一部分。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)

### 3. xv6 的实际例子
- `consoleintr` 里要先拿 `cons.lock`，再拿 process lock。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)
- 文件系统可能形成更长的锁链，因此更依赖严格顺序。 [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)

***

## 六、interrupt 与锁

### 1. 为什么中断会死锁
如果线程拿着锁时被中断，而中断处理程序又想拿同一把锁，就会卡死：  
中断处理程序等锁释放，但锁只能由被中断的线程释放，而那个线程又要等中断返回。 [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)

### 2. xv6 的做法
xv6 在 CPU 持有 spinlock 时会关闭本 CPU 中断，并用 `push_off/pop_off` 记录嵌套层数。  
这样可以避免“同 CPU 上的中断”造成死锁。 [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)

***

## 七、重排序重点

### 1. 为什么源代码顺序不一定等于执行顺序
编译器和 CPU 都可能为了性能重排指令。  
这在单线程里通常没问题，但在共享内存并发里会破坏同步逻辑。 [clownote.github](https://clownote.github.io/2021/04/06/xv6/Xv6-Locking/)

### 2. 锁为什么需要 memory barrier
如果 `release` 之前的更新被拖到锁外面，另一个 CPU 可能拿到锁后看到“数据已经公开了，但内部字段还没初始化完”。  
所以 xv6 在 acquire/release 中使用内存屏障来阻止重排序。 [clownote.github](https://clownote.github.io/2021/04/06/xv6/Xv6-Locking/)

***

## 八、sleep lock 重点

### 1. spinlock 不适合长时间等待
如果持锁期间要等磁盘 I/O 这种慢操作，spinlock 会让其他 CPU 一直空转，浪费大量时间。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)

### 2. sleep lock 的特点
sleep lock 允许等待时让出 CPU，所以适合长操作。  
但它不能用在中断处理程序里，也不能在 spinlock 临界区里睡眠。 [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)

***

## 九、复习时最该记住的几句话

- **并发 = 多核并行 + 线程切换 + 中断交错**。 [clownote.github](https://clownote.github.io/2021/04/06/xv6/Xv6-Locking/)
- **race = 同一内存位置并发访问，至少一次写**。 [clownote.github](https://clownote.github.io/2021/04/06/xv6/Xv6-Locking/)
- **锁保护的是不变量，不只是变量本身**。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)
- **spinlock 用原子指令实现互斥**。 [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)
- **死锁的关键是统一锁顺序**。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)
- **持有 spinlock 时要关中断**。 [git.silversoft](https://git.silversoft.se/Imbus/xv6-riscv-kernel/raw/commit/944964685325aa63a59e5e84d8fd0ba5af45ec8c/web/lock.pdf)
- **长时间等待用 sleep lock，不用 spinlock**。 [cse.iitd.ac](https://www.cse.iitd.ac.in/~sbansal/os/previous_years/2011/xv6-book/lock.pdf)

如果你愿意，我下一步可以把这章再整理成一份更适合背诵的 **“考试版提纲”**，或者直接给你做成 **中英对照笔记**。


#### QA
- 为什么 线程切换需要锁 ？
  ```
  因为线程切换本身会访问共享的调度状态，所以需要锁来保证同一时刻只有一个 CPU/线程在修改这些状态，避免把线程表、就绪队列、睡眠队列等结构改乱。
  
  直观理解
  线程切换不是“纯粹把 A 停掉、换成 B”这么简单，而是要先保存 A 的寄存器、状态，再修改调度数据结构，把 B 标记成可运行，最后恢复 B。
  这些步骤里用到的数据通常是共享的，如果没有锁，两个 CPU 可能同时切换线程、同时改同一个结构，就会出现丢更新或状态错乱。
  
  为什么会出错
  比如一个 CPU 正在把某个线程从“运行中”改成“睡眠”，另一个 CPU 也在处理同一线程或同一队列。
  如果没有锁，可能出现：
  
  线程状态被改了一半；
  
  一个队列节点被重复插入或遗漏；
  
  该唤醒的线程没被唤醒，或者不该运行的线程被调度了。
  
  在 xv6 里的对应关系
  xv6 里很多和进程状态有关的数据都靠锁保护，比如 p->lock。
  当内核要改变进程状态、执行 sleep/wakeup、或者在调度器和中断之间协调时，就必须先拿锁，确保这些状态转换是互斥的。
  
  一句话总结
  线程切换需要锁，不是为了“保护线程本身”，而是为了保护“线程切换过程中要修改的共享调度数据和状态转换不变量”。
  没有锁，多个 CPU 交错执行时，调度器很容易把系统状态改坏。
  ```
- 如何理解：因为我们直接 lock; do something; unlock;那么如何对应上acquire release ?
  
     可以把它理解成：**`lock; do something; unlock` 是程序员的逻辑写法，而 `acquire/release` 是把这件事真正落到硬件上的实现接口。**  
     也就是说，`lock` 不是一条真实指令，真实发生的是 `acquire()` 把锁“原子地拿到”，`release()` 把锁“原子地放掉”。 [mit-public-courses-cn-translatio.gitbook](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec10-multiprocessors-and-locking/10.7-spin-lock-1)
     
     #### 对应关系
     
     你写的伪代码：
     
     ```c
     lock;
     do something;
     unlock;
     ```
     
     在 xv6 里通常对应成：
     
     ```c
     acquire(&lk);
     do something;
     release(&lk);
     ```
     
     这里的关键不是名字，而是语义：
     
     - `acquire`：一直尝试，直到**原子地**把 `lk->locked` 从 0 改成 1。
     - `release`：**原子地**把 `lk->locked` 设回 0。 [pdos.csail.mit](https://pdos.csail.mit.edu/6.828/2008/lec/l-lock.html)
     
     #### 为什么必须“原子地”
     
     如果只是普通代码：
     
     ```c
     if(lk->locked == 0)
       lk->locked = 1;
     ```
     
     两个 CPU 可能同时看到 0，然后都把它改成 1，结果两个 CPU 都以为自己拿到了锁，这就失去互斥了。  
     所以 `acquire` 不能只是“检查再赋值”，而必须让“检查 + 修改”成为一个不可分割的动作。 [mit-public-courses-cn-translatio.gitbook](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec10-multiprocessors-and-locking/10.7-spin-lock-1)
     
     #### `acquire` 实际做了什么
     
     `acquire` 的本质是一个循环：
     
     1. 用原子交换指令尝试把锁值改成 1。
     2. 如果返回值说明原来已经被别人拿走了，就继续等。
     3. 如果返回 0，说明你拿到了锁。 [pdos.csail.mit](https://pdos.csail.mit.edu/6.828/2008/lec/l-lock.html)
     
     所以你可以把它理解成：
     
     - **逻辑上**：`lock`
     - **实现上**：`acquire()` + 原子交换 + 忙等 + 内存屏障
     
     #### `release` 实际做了什么
     
     `release` 刚好相反：
     
     1. 先确认当前确实持有锁。
     2. 再把锁状态原子地改回 0。
     3. 让别的 CPU 之后可以成功 `acquire`。 [mit-public-courses-cn-translatio.gitbook](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec10-multiprocessors-and-locking/10.7-spin-lock-1)
     
     所以你写的 `unlock`，对应的就是 `release()`。
     
     #### 一个更准确的理解方式
     
     可以把它看成两层：
     
     ##### 1. 抽象层
     你只关心：
     
     - 进入临界区前先拿锁
     - 退出临界区后放锁
     
     ##### 2. 机器层
     系统必须保证：
     
     - 拿锁不是普通赋值，而是原子操作
     - 放锁也不是普通赋值，而是原子操作
     - 临界区内的读写不能被编译器/CPU 随便重排。 [pdos.csail.mit](https://pdos.csail.mit.edu/6.828/2008/lec/l-lock.html)
     
     #### 你可以记成一句话
     
     **`lock/unlock` 是“我想要互斥”的语义，`acquire/release` 是“系统用原子指令把互斥实现出来”的机制。** [mit-public-courses-cn-translatio.gitbook](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec10-multiprocessors-and-locking/10.7-spin-lock-1)
     
- 解释 kalloc：一个 free list + 一把锁，简单但竞争大 ？
     
     在 xv6 的原始设计中，内存分配器 (`kalloc`) 使用一个**全局的空闲链表 (`freelist`)** 和**一把全局锁 (`kmem.lock`)**。这种设计虽然逻辑清晰，但在多核高并发场景下存在严重的性能瓶颈。 [cnblogs](https://www.cnblogs.com/lilpig/p/17260254.html)
   
   - 为什么竞争大（瓶颈所在）
   1.  **串行化执行**：无论哪个 CPU 需要分配或释放内存，都必须争抢同一把 `kmem.lock`。即使多个 CPU 之间并没有真实的内存资源冲突，它们也被迫排队。 [juejin](https://juejin.cn/post/7496345865217966116)
   2.  **锁争用（Lock Contention）**：当系统负载变高、频繁调用 `kalloc` 或 `kfree` 时，CPU 会把大量时间浪费在“自旋（spinning）”等待锁上，而不是执行实际的内存管理工作，造成 CPU 资源浪费。 [cloud.tencent](https://cloud.tencent.com/developer/article/2142304)
   3.  **缓存抖动**：因为所有 CPU 都在频繁抢占同一个内存地址（锁变量），这会导致该锁所在的缓存行在各个 CPU 的 L1/L2 缓存之间高速频繁地“乒乓”迁移，导致硬件性能急剧下降。 [juejin](https://juejin.cn/post/7496345865217966116)
   
   - 如何优化（核心思路）
   为了解决这一问题，可以将“全局资源”转变为“局部资源”，这也是操作系统优化的常用范式： [cloud.tencent](https://cloud.tencent.com/developer/article/2142304)
   
   - **按 CPU 拆分**：为每个 CPU 创建独立的 `freelist`，并配备独立的 `kmem[NCPU].lock`。 [blog.miigon](https://blog.miigon.net/posts/s081-lab8-locks/)
   - **减少冲突**：大部分情况下，CPU 只在自己的 `freelist` 上操作，无需跨 CPU 争抢，从而实现了真正的并行分配。 [juejin](https://juejin.cn/post/7496345865217966116)
   - **“偷取”机制（Stealing）**：如果某个 CPU 的本地 `freelist` 空了，才去尝试从其他 CPU 的 `freelist` 中“偷”一些页出来。这样既保留了并行性，又保证了内存资源的全局调度。 [blog.csdn](https://blog.csdn.net/John_Snowww/article/details/139245029)
   
   这种**从单一全局锁到分片（Per-CPU）锁**的演进，是操作系统内核设计中解决高并发锁争用的经典案例。 [cloud.tencent](https://cloud.tencent.com/developer/article/2142304)
