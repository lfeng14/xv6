先把结论放前面：这段对话主要围绕「xv6 的进程同步与锁」展开，核心知识点可以按从浅到深整理成一个编号清单，方便你回顾整章内容。

***

## 1. sleep / wakeup 的基本模型

1.1 经典场景：管道读写  
- 管道读进程没数据时，不忙等，而是调用 sleep 睡在某个 *channel*（等待队列）上。 [cse.iitb.ac](https://www.cse.iitb.ac.in/~mythili/os/notes/old-xv6/xv6-sync.pdf)
- 写进程写入数据后，调用 wakeup 在同一个 channel 上把睡着的读进程唤醒。 [pdos.csail.mit](https://pdos.csail.mit.edu/6.S081/2020/lec/l-coordination.txt)
- 这构成了一种「通知机制」：没事干就睡，有事了再叫醒。

1.2 等待队列 + channel  
- sleep 会把当前进程挂到与 channel 关联的等待队列里，并把状态改为 SLEEPING。 [velog](https://velog.io/@dixeris/Understanding-xv6-Scheduling)
- wakeup 会扫描进程表，唤醒所有在该 channel 上睡眠的进程，使其变为 RUNNABLE。 [cs.hmc](https://www.cs.hmc.edu/~rhodes/courses/cs134/sp19/lectures/Lecture11.pdf)

1.3 和 P/V、条件变量的类比  
- xv6 的 sleep / wakeup 类似于「条件变量 + signal/broadcast」或信号量里的 P/V，只是接口更原始。 [juejin](https://juejin.cn/post/7032495720926019615)
- 管道里的同步就是在 sleep/wakeup 之上封装出的 pipewrite / piperead，相当于把「P/V」嵌在读写操作中。 [cse.iitb.ac](https://www.cse.iitb.ac.in/~mythili/os/notes/old-xv6/xv6-sync.pdf)

***

## 2. spinlock、sleeplock 和一般 sleep/wakeup 的关系

2.1 spinlock 的特性  
- spinlock 获取不到就一直在 CPU 上忙等，适合临界区很短的场景。 [xiayingp.gitbook](https://xiayingp.gitbook.io/build_a_os/lock/locking-in-xv6)
- 不能在持有 spinlock 时主动睡眠或被调度切走，否则容易死锁。 [cse.iitb.ac](https://www.cse.iitb.ac.in/~mythili/os/anno_slides/lecture29.pdf)

2.2 sleeplock 的特性  
- sleeplock 内部用一个短临界区的 spinlock + sleep/wakeup 组合实现。 [kkmtyyz.github](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_10_lock.html)
- 线程拿不到 sleeplock 时会 **睡眠**，而不是原地疯狂自旋，适合访问时间较长的资源。 [xiayingp.gitbook](https://xiayingp.gitbook.io/build_a_os/lock/locking-in-xv6)
- 你总结得对：sleeplock 也是一种「先睡，等别人处理完再通知我」的同步方式，只是它包装成了“锁”的接口。

2.3 一般 sleep/wakeup 的通用性  
- sleep / wakeup 是底层原语，很多更高层的同步结构（管道、sleeplock、wait/exit/kill 等）都是在它之上封装出来的。 [pages.cs.wisc](https://pages.cs.wisc.edu/~gerald/cs537/Summer17/handouts/scheduling.pdf)

***

## 3. 管道 pipewrite / piperead 的具体行为

3.1 管道缓冲区与计数器  
- 管道内部有环形缓冲区，以及 nread / nwrite 指针用来判断「空」还是「满」。 [pekopeko11.sakura.ne](https://pekopeko11.sakura.ne.jp/unix_v6/xv6-book/en/Scheduling.html)

3.2 piperead：没数据就 sleep  
- piperead 获取锁，发现缓冲区空（nread == nwrite），就把自己 sleep 在某个 channel 上（比如 &p->nread），释放锁，等待写端叫醒。 [pdos.csail.mit](https://pdos.csail.mit.edu/6.S081/2020/lec/l-coordination.txt)
- 被唤醒后重新拿锁，再次检查条件（while 循环），有数据才真正读，否则可能再次睡眠。 [juejin](https://juejin.cn/post/7032495720926019615)

3.3 pipewrite：写完 / 写满时 wakeup  
- pipewrite 往缓冲里写数据，每写入一些数据后，会 wakeup 在读 channel 上睡眠的进程。 [pdos.csail.mit](https://pdos.csail.mit.edu/6.S081/2020/lec/l-coordination.txt)
- 如果缓冲区满了，写端也会 sleep 在写者自己的 channel 上，等读者读走一些数据再被唤醒继续写。 [cs.hmc](https://www.cs.hmc.edu/~rhodes/courses/cs134/sp19/lectures/Lecture11.pdf)
- 你后面归纳的那句其实很准确：pipewrite 会调用 wakeup，piperead 会在空时调用 sleep，底层依赖等待队列和锁。

***

## 4. wait / exit / kill 与 sleep/wakeup 的关系

4.1 exit：子进程退出  
- 子进程 exit 时，状态变为 ZOMBIE，释放大部分资源，然后 wakeup 父进程（父亲一般 sleep 在「有子退出」这个条件上）。 [xiayingp.gitbook](https://xiayingp.gitbook.io/build_a_os/scheduling/how-does-wait-exit-kill-work)

4.2 wait：父进程等待子进程  
- 父进程 wait 时，遍历子进程。  
  - 如果发现有 ZOMBIE 子进程，就回收并返回。  
  - 如果没有任何子进程退出，就 sleep 在一个 channel 上等待被唤醒。 [pages.cs.wisc](https://pages.cs.wisc.edu/~gerald/cs537/Summer17/handouts/scheduling.pdf)
- 所以你说得对：wait 本质上也是「基于 sleep/wakeup 的同步」。  

4.3 kill：与 sleep 的竞态  
- kill 只是设置目标进程的 p->killed 标志，并把它从 SLEEPING 转为 RUNNABLE（类似 wakeup）。 [xiayingp.gitbook](https://xiayingp.gitbook.io/build_a_os/scheduling/how-does-wait-exit-kill-work)
- 很多内核睡眠循环像：  
  - while 条件不满足且 !p->killed 时 sleep。  
  - 被唤醒后再次检查条件或 killed 标志，决定继续等还是中止系统调用。 [pages.cs.wisc](https://pages.cs.wisc.edu/~gerald/cs537/Summer17/handouts/scheduling.pdf)

***

## 5. kill 与 sleep 之间的竞态（练习题里的 “fix the race”）

5.1 问题场景  
- 典型模式：  

  - 进程在循环里先检查 `p->killed`，然后发现条件不满足，准备 `sleep(chan, lock)`。 [github](https://github.com/palladian1/xv6-annotated/blob/main/syscalls_routing.md)
  - 正在「检查 killed 与真正进入 sleep」这段空隙中，另一个 CPU 调用 kill：设置 killed 并尝试唤醒它。 [xiayingp.gitbook](https://xiayingp.gitbook.io/build_a_os/scheduling/how-does-wait-exit-kill-work)
  - 进程还没真的进入 SLEEPING 状态，所以 wakeup 找不到它；接着进程立刻 sleep，结果被杀了也还在睡，导致“错过 kill”。 [pages.cs.wisc](https://pages.cs.wisc.edu/~gerald/cs537/Summer17/handouts/scheduling.pdf)

5.2 修复思路  
- 把「检查 killed」和「进入 sleep」放在同一把锁保护下：  
  - 先持有锁（通常是条件锁或 ptable.lock），  
  - 在持锁状态下检查条件和 p->killed，  
  - 决定要不要 sleep，如果要，就直接调用 sleep，sleep 会在内部配合 ptable.lock 做原子的「标记 SLEEPING + 调度切换」。 [velog](https://velog.io/@dixeris/Understanding-xv6-Scheduling)
- 这样 kill 也会在持有相同锁的情况下设置 killed 并 wakeup，不会发生「介于检查和 sleep 之间」的竞态。 [juejin](https://juejin.cn/post/7032495720926019615)

5.3 你提到的练习题原文  
- “Fix the race mentioned above between kill and sleep so that a kill that occurs after the victim’s sleep loop checks p->killed but before it calls sleep results in the victim abandoning the current system call.” [pages.cs.wisc](https://pages.cs.wisc.edu/~gerald/cs537/Summer17/handouts/scheduling.pdf)
- 含义就是：即便 kill 刚好发生在「检查 killed 后、调用 sleep 前」，最终也要确保这个进程不会继续傻睡，而是尽快结束当前系统调用退出。  

***

## 6. signal / broadcast 与 wakeup 的联系

6.1 signal / broadcast 的语义  
- 在条件变量语境下：  
  - signal：唤醒等待队列中的一个线程；  
  - broadcast：唤醒等待队列中的所有线程。 [cs.hmc](https://www.cs.hmc.edu/~rhodes/courses/cs134/sp19/lectures/Lecture11.pdf)

6.2 和 xv6 wakeup 的对应  
- xv6 的 wakeup 实现更偏向「broadcast」语义：唤醒所有在某个 channel 上睡眠的进程。 [pdos.csail.mit](https://pdos.csail.mit.edu/6.S081/2020/lec/l-coordination.txt)
- 你问「signal 怎么做到只唤醒一个」：  
  - 一般实现就是操作等待队列，只从队列头取出一个等待者唤醒，其余仍然保持睡眠。 [github](https://github.com/palladian1/xv6-annotated/blob/main/syscalls_routing.md)
- signal 更精细，可以避免惊群（所有 wait 线程同时被吵醒又抢锁）；wakeup/ broadcast 更简单粗暴。

***

如果以后你再看 xv6 的代码（例如 sleep、wakeup、pipe、wait/exit/kill 那几段），可以按这 6 条知识线索去对照，就比较容易把“概念章节”和“代码页”串起来。你现在主要是在哪本讲义或 PPT 上看这章？如果你愿意，我可以按你手上的页码帮你把每一页的要点都做一个小笔记版。  
先给你把这一整章你贴出来的部分，整理成「大纲 + 每节重点」，方便你通篇把握脉络，然后你如果愿意，我们可以再针对某一小节细抠。下面只总结你发的这几节（7.1–7.9）。

***

## 整体章节主线（Scheduling）

- 系统里通常进程数 > CPU 数，需要用调度来让大家「轮流用 CPU」，营造每个进程都有自己 CPU 的错觉。 [studocu](https://www.studocu.com/en-us/document/southern-new-hampshire-university/computer-operating-systems/operating-system-concepts-ch-3-processes-and-scheduling-overview/134231523)
- xv6 通过「上下文切换 + 定时器中断 + sleep/wakeup + 一套锁规则」来实现这个虚拟 CPU 幻觉，并解决并发下的一堆 race / 资源回收问题。 [cs.uic](https://www.cs.uic.edu/~jbell/CourseNotes/OperatingSystems/6_CPU_Scheduling.html)

可以把整章理解成：**三条线交织**——  
1）CPU 怎么在不同进程之间切换（swtch + scheduler）；  
2）进程如何挂起/被唤醒（sleep/wakeup + 管道、wait/exit/kill）；  
3）为保证这些行为正确，需要怎样的锁设计（p->lock、wait_lock、mycpu/myproc）。

***

## 7.1 Multiplexing（多路复用：为什么要调度）

**要点：**

- 问题：进程数 > CPU 数，必须 time-share，让一个 CPU 看起来像被多个进程「轮流使用」；对用户进程来说最好是透明的（感觉自己独占 CPU）。 [studocu](https://www.studocu.com/en-us/document/southern-new-hampshire-university/computer-operating-systems/operating-system-concepts-ch-3-processes-and-scheduling-overview/134231523)
- xv6 两种切换场景：  
  - 主动挂起：进程在等待 I/O、子进程退出、sleep 系统调用等，通过 sleep/wakeup 让出 CPU。  
  - 被动抢占：定时器周期性中断，防止进程「一直算、不睡觉」，强制切换。  
- 实现这个幻觉，带来几个核心问题：  
  - 如何实现上下文切换（context switch）？  
  - 如何用定时器中断驱动抢占，并保证对用户透明？ [cs.uic](https://www.cs.uic.edu/~jbell/CourseNotes/OperatingSystems/6_CPU_Scheduling.html)
  - 多核共享同一进程集合，如何加锁避免并发 bug？  
  - 进程退出时，内核栈、内存等资源如何安全回收（进程不能自己把正在用的栈 free 掉）？  
  - 每个 CPU 如何记住自己当前跑的是哪个进程？  
  - sleep/wakeup 如何避免丢失唤醒（lost wakeup）？  

**可以记一句：**  
调度 = 定时器驱动的多路复用 + sleep/wakeup 协调等待 + 一套精细的锁/状态不变量。

***

## 7.2 Code: Context switching（swtch 的底层）

**大纲：**

- 上下文切换路径：用户进程 A → A 的内核线程 → scheduler 线程 → B 的内核线程 → 用户进程 B。  
- 每个 CPU 有一个专门的 scheduler 线程（自己的栈和 context），不能用进程自己的内核栈跑 scheduler（否则可能被别的核同时使用）。  
- swtch：纯粹保存/恢复寄存器（context），不懂「线程」的概念，只操作 struct context。  

**关键细节：**

- swtch 的调用关系：  
  - 用户态 → trap → usertrap → yield → sched → swtch(&p->context, &cpu->context)。  
  - 之后 swtch 返回到 scheduler 上一次调用 swtch 的位置（co-routine 风格）。  
- swtch 只保存 **callee-saved 寄存器**，caller-saved 由 C 调用方自己在栈上保存。 [user.it.uu](https://user.it.uu.se/~leomu203/osvt08/lec4.pdf)
- 不直接保存 PC，而是保存 ra（返回地址），恢复时通过 ra 回到「上一次调用 swtch 的地方」，并且栈指针也切换到新线程的栈上。  

可以把它抽象成：  
> 逻辑上：线程切换；  
> 实现上：函数 swtch(old_ctx, new_ctx) 做「寄存器快照 + 恢复」，靠 ra 实现「回到某个函数内部」。

***

## 7.3 Code: Scheduling（scheduler / sched / yield）

**大纲：**

- 每个 CPU 一条 scheduler 内核线程，死循环：找 RUNNABLE → 切到它 → 它跑一阵 → yield/sleep/exit → 回到 scheduler。  
- 进程要放弃 CPU 时：  
  - 持有自己的 p->lock，释放其它锁，修改 p->state，然后调用 sched。  
- sched：  
  - 检查持锁、检查中断已关闭，然后 swtch 到 cpu->context（scheduler 线程）。  

**几个不太直观但很重要的点：**

- p->lock 在 swtch 过程中跨线程传递：  
  - yield 里拿着 p->lock 调用 sched → sched 调 swtch 切到 scheduler → scheduler 在自己的栈上，最终再释放 p->lock。  
  - 这违背「谁 acquire 谁 release」的常规习惯，但为的是：在进程从 RUNNING → RUNNABLE 的转换过程中，**必须一直有人持有 p->lock 保证不变量成立**。  
- 如果不在 swtch 期间持有 p->lock，会有严重竞态：  
  - 比如：yield 把 state 设为 RUNNABLE，但还没真正停止用自己的内核栈；此时另一个 CPU 的 scheduler 可能看到 RUNNABLE，把它调度起来，结果两个 CPU 共用一个内核栈，炸。  
- 大致不变量：  
  - RUNNING：真实寄存器里是该进程上下文，c->proc 指向它，没人用 p->context 里的快照。  
  - RUNNABLE：寄存器状态保存在 p->context，没人在它的内核栈上跑，也没有 c->proc 指向它。  

可以记为：  
> scheduler / sched / yield 构成一个「协程对」，通过 swtch 来回跳转，并用 p->lock 护住状态转换不变量。

***

## 7.4 Code: mycpu 和 myproc（当前 CPU/进程定位）

**要点：**

- 每个 CPU 有一个 struct cpu：包含当前运行的 proc 指针 c->proc、scheduler 上下文、嵌套关闭中断计数等。  
- RISC‑V 上，每个核有一个 hartid，xv6 把它存在 tp 寄存器里，mycpu 用 tp 作为索引从 cpu[] 数组中找到当前 CPU 的 struct cpu。 [user.it.uu](https://user.it.uu.se/~leomu203/osvt08/lec4.pdf)
- tp 的维护：  
  - 启动时 start 在 machine mode 设置 tp。  
  - usertrapret 保存/恢复 tp（因为用户态可能会用 tp）。  
- myproc：  
  - 关中断 → 调 mycpu→ 取 c->proc → 再开中断。  
  - 返回的 struct proc* 在被移动到其他 CPU 时仍然有效，所以在后续可以在开中断状态下用。  

**核心直觉：**  
> 多核下「全局变量 current」不再行，每个核要有自己的 per‑CPU 结构；用 tp 这类专用寄存器来快速定位「本核的状态」。

***

## 7.5–7.6 Sleep and wakeup（语义、lost wakeup、正确实现）

这两节是这一章里最容易考、也最容易把人搞晕的部分。

### 语义与问题（7.5）

- 场景：读 pipe 等数据、wait 等子进程退出、磁盘 I/O 完成等，都需要「某线程等待某个事件」。  
- 提供抽象：  
  - sleep(chan)：让当前线程睡在「等待通道」chan 上，并让出 CPU。  
  - wakeup(chan)：唤醒所有睡在 chan 上的线程。  
- 用 sleep/wakeup 实现高层同步（比如 semaphore 的 P/V），可以避免忙等（spin）。  

**lost wakeup 经典坑：**

- 伪代码（旧版 sleep 接口）：
  - P：检查 count == 0，若 0 就 sleep(s)；  
  - V：count++ 后 wakeup(s)。  
- Bug 场景：  
  - P 看见 count == 0，还没调用 sleep；  
  - 此时 V 执行，count++，调用 wakeup(s)，但此时没有人在睡；  
  - V 返回；  
  - P 接着调用 sleep(s)，睡死了，永远等不到未来的 V（已经发生过）。  
- 根因：  
  - 「只有当 count == 0 才 sleep」这一不变量被并发破坏，检查和 sleep 不是原子步骤。  

尝试用「锁包住 while + sleep」的错误修复：

- 在 P 中：  
  - acquire(lock)  
  - while(count == 0) sleep(s)  
  - …  
- 结果：死锁，因为 P 在 sleep 时持有锁，V 为了加 count 和 wakeup 也需要拿锁，永远拿不到。  

### 正确接口与实现（7.6）

**关键改动：sleep(chan, lk)**

- 调用 sleep 时必须传入「条件锁」lk：  
  - sleep 内部会：  
    - 先拿 p->lock（自己的进程锁）；  
    - 再释放 lk；  
    - 将自己标记为 SLEEPING，记录 chan，调用 sched；  
    - 之后在被唤醒后，重新 acquire(lk)，再返回给调用者。  
- wakeup(chan)：  
  - 遍历进程表，对每个进程先拿 p->lock；  
  - 找到 state == SLEEPING 且 chan 匹配的，就改为 RUNNABLE。  

**为什么不会丢唤醒？核心时序：**

- 睡眠线程：从检查条件开始到真正变成 SLEEPING，这段时间始终持有「条件锁 lk 或 p->lock 或两者之一」。  
- 唤醒线程：调用 wakeup 时，在自己的临界区内持有「条件锁 lk」，进入 wakeup 后再一个个拿各个进程的 p->lock。  
- 因此：  
  - 要么唤醒者在检查条件之前就把条件置真（然后 P 不会去 sleep）；  
  - 要么唤醒者在 P 已经标记为 SLEEPING 之后，再看到它并把它变成 RUNNABLE。  
- 只要 sleep 在「标记为 SLEEPING」之前不放掉 p->lock，就不会出现「已经 wakeup，却错过了正在入睡的进程」的情况。  

**使用约定：**

- sleep 一定在 while 循环里调用，醒来后重新检查条件，因为：  
  - wakeup(chan) 会唤醒所有睡在 chan 的进程，可能造成「虚假唤醒」；  
  - 多个线程竞争同一个资源时，只有第一个真正拿到资源，其它要重新 sleep。  

***

## 7.7 Pipes（用 sleep/wakeup 实现生产者–消费者）

**大纲：**

- struct pipe：  
  - 锁 + 环形 buffer（buf[] + nread + nwrite）。  
- pipewrite：  
  - 拿锁，循环写：  
    - 若缓冲满（nwrite == nread + PIPESIZE），说明没有空间：  
      - wakeup(&pi->nread) 通知读者有数据；  
      - 然后 sleep(&pi->nwrite, &pi->lock) 等读者腾空间；  
  - 写完后，wakeup(&pi->nread) 通知读者。  
- piperead：  
  - 拿锁，若空且没有 writer，直接返回；若空但有 writer，就 sleep(&pi->nread, &pi->lock) 等数据；  
  - 有数据就从 buf 里读，并增加 nread；  
  - 读完后 wakeup(&pi->nwrite) 通知写者有空间。  

**要点：**

- 读写双方通过不同的 chan（&pi->nread / &pi->nwrite）互相唤醒。  
- 都是在 while 循环里 sleep，处理「多个等待者 + 虚假唤醒」。  
- pipe 的实现直接用 sleep/wakeup + 锁，实现了一个典型的生产者–消费者队列。  

***

## 7.8 Wait, exit, kill（进程生命周期 & 同步）

**主线：**

- 目标：父进程 wait 能可靠地观察到子进程的 exit，不丢信息、不死锁；同时 kill 能请求其他进程退出，但真正的退出必须由目标进程自己在安全点完成。  

**状态流转：**

- 子进程 exit：  
  - 将自己标记为 ZOMBIE；  
  - 释放资源、把孩子 reparent 给 init；  
  - wakeup(parent) 通知父进程；  
  - 永久 yield，等父进程在 wait 中回收它。  
- 父进程 wait：  
  - 持有 wait_lock，遍历进程表找子进程：  
    - 若找到 ZOMBIE 子进程：清理、free struct proc、返回 pid；  
    - 若有子但都没死：sleep(&parent, &wait_lock)，等待 exit 的唤醒，然后再扫描。  

**锁设计：**

- wait_lock：  
  - 作为「条件锁」，配合 sleep/wakeup 避免 parent 丢失子进程的退出通知。  
  - 也用于序列化 parent/child 同时 exit，确保孤儿进程交给 init 时正确唤醒 init。  
- exit 同时持有 wait_lock 和 p->lock（自己的进程锁），顺序与 wait 一致，避免死锁。  

**kill：**

- 不直接杀死目标进程，而是：  
  - 设置 p->killed 标志；  
  - 若在 sleep，就 wakeup；  
- 真正退出发生在：  
  - 进程进入内核（系统调用或中断）时，在 usertrap/killed() 处看到 p->killed，然后走 exit 流程。  
- 某些 sleep 循环会检查 p->killed 并提前放弃当前操作（比如 pipe 读写），保证被 kill 后尽快返回用户态，再退出。  

***

## 7.9 Process Locking（p->lock 和 wait_lock 的角色整理）

这一节基本在总结前面所有用锁保护进程状态的场景。

**p->lock 的职责（可以背）：**

大致可以粗分为两类：

1. 保护 proc 结构的关键字段：  
   - p->state、p->chan、p->killed、p->xstate、p->pid 等。  
2. 保护「更高层的不变量」和流程：  
   - 进程分配/回收时，防止两个 CPU 同时碰同一个 proc 槽。  
   - 创建/销毁过程中，隐藏进程，使调度器和 wait 不会看到半初始化或半销毁状态。  
   - 防止 parent 的 wait 在子进程还没完全切到 ZOMBIE 并 swtch 之前就回收它。  
   - 确保：  
     - yielding 进程在 state 切到 RUNNABLE 但还没完全停止用内核栈时，不会又被另一个 CPU 调度起来。  
     - 只有一个调度器会选择某个 RUNNABLE 进程执行。  
     - 在 swtch 内不会被 timer 中断导致 yield 再次进入 swtch，打乱状态。  
   - 配合条件锁，确保 sleep 和 wakeup 在各种时序下不丢唤醒（特别是刚要睡或刚要醒的那一瞬间）。  
   - 给 kill 提供原子检查/修改：  
     - 防止在 kill 检查 p->pid 与设置 p->killed 之间，目标进程退出、被回收、再被复用。  
     - 让 kill 对 p->state 的检查与修改是原子的。  

**wait_lock：**

- 专门保护 parent/child 关系中的 p->parent，并作为 wait 的条件锁。  
- 保证：  
  - 退出的子进程在设置 ZOMBIE、唤醒父进程、yield 之间不会被父进程提前收割。  
  - 父子同时 exit 时，init 一定能被唤醒，负责收尸。  

***

如果你愿意，下一步我可以帮你做一份「超精简背诵版」：  
- 每小节压缩成 3～5 条「考点句子」；  
- 或者选其中某一块（比如 sleep/wakeup 或 p->lock）画「时序图/流程图式文字说明」，帮助你真正搞透。  

你目前是更想重点啃哪一块：上下文切换（swtch/sched）、sleep‑wakeup、还是 wait/exit/kill 这一组？
