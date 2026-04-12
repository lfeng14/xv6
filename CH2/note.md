- Thus an operating system must fulfill three requirements: multiplexing, isolation, and interaction.
- To achieve strong isolation it’s helpful to forbid applications from directly accessing sensitive
hardware resources, and instead to abstract the resources into services. For example, Unix applica-
tions interact with storage only through the file system’s open, read, write, and close system
calls, instead of reading and writing the disk directly. This provides the application with the con-
venience of pathnames, and it allows the operating system (as the implementer of the interface)
to manage the disk. Even if isolation is not a concern, programs that interact intentionally (or just
wish to keep out of each other’s way) are likely to find a file system a more convenient abstraction
than direct use of the disk
- Similarly, Unix transparently switches hardware CPUs among processes, saving and restor-
ing register state as necessary, so that applications don’t have to be aware of time sharing. This
transparency allows the operating system to share CPUs even if some applications are in infinite
loops.
- 抽象和隔离问题：The system-call interface in Figure 1.2 is carefully designed to provide both programmer con-
venience and the possibility of strong isolation. The Unix interface is not the only way to abstract
resources, but it has proven to be a very good one.
  <img width="360" height="150" alt="image" src="https://github.com/user-attachments/assets/534b6319-d991-4740-98ed-8c805aa6f809" />
  <img width="480" height="600" alt="image" src="https://github.com/user-attachments/assets/0f915363-b964-437c-aaa7-5bde8d61bfee" />

- Strong isolation requires a hard boundary between applications and the operating system. If the
application makes a mistake, we don’t want the operating system to fail or other applications to
fail. Instead, the operating system should be able to clean up the failed application and continue
running other applications. To achieve strong isolation, the operating system must arrange that
applications cannot modify (or even read) the operating system’s data structures and instructions
and that applications cannot access other processes’ memory
- In supervisor mode the CPU is allowed to execute privileged instructions: for example, en-
abling and disabling interrupts, reading and writing the register that holds the address of a page
table, etc. If an application in user mode attempts to execute a privileged instruction, then the CPU
doesn’t execute the instruction, but switches to supervisor mode so that supervisor-mode code can
terminate the application, because it did something it shouldn’t be doing. Figure 1.1 in Chapter 1
illustrates this organization. An application can execute only user-mode instructions (e.g., adding
numbers, etc.) and is said to be running in user space, while the software in supervisor mode can
also execute privileged instructions and is said to be running in kernel space. The software running
in kernel space (or in supervisor mode) is called the kernel
- An application that wants to invoke a kernel function (e.g., the read system call in xv6) must
transition to the kernel; an application cannot invoke a kernel function directly. CPUs provide a
special instruction that switches the CPU from user mode to supervisor mode and enters the kernel
at an entry point specified by the kernel. (RISC-V provides the ecall instruction for this purpose.)
Once the CPU has switched to supervisor mode, the kernel can then validate the arguments of the
system call (e.g., check if the address passed to the system call is part of the application’s memory),
decide whether the application is allowed to perform the requested operation (e.g., check if the
application is allowed to write the specified file), and then deny it or execute it. It is important that
the kernel control the entry point for transitions to supervisor mode; if the application could decide
the kernel entry point, a malicious application could, for example, enter the kernel at a point where
the validation of arguments is skipped.
- 也就是user space --syscall->kernel space(supervisor space)硬件寄存器上是有适配的，寄存器状态变化；enters the kernel at an entry point specified by the kernel.
- A key design question is what part of the operating system should run in supervisor mode. One
possibility is that the entire operating system resides in the kernel, so that the implementations of
all system calls run in supervisor mode. This organization is called a monolithic kernel.
- A downside of the monolithic organization is that the interfaces between different parts of the
operating system are often complex (as we will see in the rest of this text), and therefore it is
easy for an operating system developer to make a mistake. In a monolithic kernel, a mistake is
fatal, because an error in supervisor mode will often cause the kernel to fail. If the kernel fails,
the computer stops working, and thus all applications fail too. The computer must reboot to start
again.
  To reduce the risk of mistakes in the kernel, OS designers can minimize the amount of operating
system code that runs in supervisor mode, and execute the bulk of the operating system in user
mode. This kernel organization is called a microkernel.
- 微内核优势内核相对简单，但是频繁的用户、内核切换开销会比较大；In the real world, both monolithic kernels and microkernels are popular. Many Unix kernels
are monolithic. For example, Linux has a monolithic kernel, although some OS functions run as
user-level servers (e.g., the windowing system). Linux delivers high performance to OS-intensive
applications, partially because the subsystems of the kernel can be tightly integrated.
- Operating systems such as Minix, L4, and QNX are organized as a microkernel with servers,
and have seen wide deployment in embedded settings. A variant of L4, seL4, is small enough that
it has been verified for memory safety and other security properties [8].
There is much debate among developers of operating systems about which organization is
better, and there is no conclusive evidence one way or the other. Furthermore, it depends much on
what “better” means: **faster performance, smaller code size, reliability of the kernel, reliability **of
the complete operating system (including user-level services), etc
- 微内核、红内核各有优劣，比如微内核将file server移到用户态可能增加了用户内核切换，但是提升这种file server可靠性；
- From this book’s perspective, microkernel and monolithic operating systems share many key
ideas. They implement system calls, they use page tables, they handle interrupts, they support processes, they use locks for concurrency control, they implement a file system, etc. This book
focuses on these core ideas.
**Xv6 is implemented as a monolithic kernel**, like most Unix operating systems. Thus, the xv6
kernel interface corresponds to the operating system interface, and the kernel implements the com-
plete operating system. Since xv6 doesn’t provide many services, its kernel is smaller than some
microkernels, but conceptually xv6 is monolithic.
<img width="460" height="600" alt="image" src="https://github.com/user-attachments/assets/fa800360-90c3-4b36-b262-42105b521836" />
- 早期xv6支持多进程，不支持多线程，请问为什么 ？
  - 这是一个非常经典的问题，触及了操作系统设计的核心权衡。简单来说，xv6 不支持多线程（同一个进程内有多个执行流）并不是因为它做不到，而是因为其**教学目标**和**设计哲学**决定的。xv6 的存在是为了让学生在最短的时间内理解操作系统的核心概念，而“线程”带来的复杂性往往会掩盖这些核心原理。以下是具体的几个原因：
  - 教学的简约性 (Pedagogical Simplicity)。xv6 的首要目标是**易读性**。目前 xv6 的源代码只有几千行，一个学期就能通读。
    * **进程模型简单清晰**：在 xv6 中，`struct proc` 既是资源分配的单位（内存、文件描述符），也是调度的单位。这种“一对一”的关系非常直观。
    * **线程的复杂性**：如果引入多线程，内核需要区分 **进程控制块 (PCB)** 和 **线程控制块 (TCB)**。你需要处理一个进程内多个线程共享地址空间、共享文件描述符的情况，这会增加大量的记账代码（Bookkeeping code）。
    
  - 地址空间管理的复杂化。这是技术上最直观的障碍。
    * **单一进程模型**：在 xv6 中，每个进程都有自己独立的页表（Page Table）。内核切换进程时，直接切换页表寄存器（如 RISC-V 的 `satp`）。
    * **多线程模型**：如果一个进程有多个线程，它们必须**共享同一个页表**。
        * 这意味着你需要为每个线程分配独立的**用户栈 (User Stack)** 和**内核栈 (Kernel Stack)**。
        * 在同一个地址空间里管理多个栈，需要处理栈溢出保护、栈的动态增长等极其琐碎的问题。
  - 同步与锁的噩梦。多线程会极大地增加内核与用户态的同步难度：
    * **用户态同步**：为了让多线程有用，内核必须实现复杂的同步原语（如 `futex` 或更高级的条件变量），否则用户没法安全地访问共享内存。
    * **内核态挑战**：当多个线程同时通过系统调用陷入内核时，内核代码必须是**重入安全**的。虽然 xv6 的内核已经是并行的（支持多 CPU 核心），但多线程会引入更多细粒度的锁竞争，这对于初学者来说是理解上的灾难。
  - 历史渊源 (Unix V6 的传承)。xv6 是对 **Unix V6** 的现代重写。
    * 在 1970 年代，Unix V6 本身就只有进程的概念。
    * 早期的 Unix 哲学倾向于“进程间通信 (IPC)”，而不是“共享内存线程”。如果你需要并发，传统的做法是 `fork()` 一个子进程。
    
  - 语义定义的模糊性。实现线程会带来一系列令人头疼的语义问题，xv6 的设计者显然想避开这些：
    * 如果一个线程调用了 `exit()`，是整个进程退出还是只有该线程退出？
    * 如果一个线程调用了 `exec()`，其他线程该怎么办？
    * 信号（Signal）应该发送给哪个线程？
  - 总结：xv6 支持**多进程**是因为它要演示如何利用多核 CPU 和硬件隔离（虚拟内存）；它不支持**多线程**是因为多线程引入的边角情况（Edge Cases）太多，会显著增加代码量，降低系统的教学透明度。如果你对在 xv6 中实现线程感兴趣，这通常是很多操作系统课程（如 MIT 6.S081）的**进阶实验题目**（例如 `uthread` 或 `kthread` 实验）。
  - **简单来说：xv6 是为了让你“看清”系统，而不是为了让你“高效”地开发应用。**
- 如果我想在xv6上增加多线程，我应该思考哪些 增加哪些 ？
  - 系统调用：增加线程创建等
  - 调度
  - 内存布局：栈布局
 
- 当然，在现实世界中情况并非如此简单。我们很难阻止精巧的用户代码通过消耗受内核保护的资源——如磁盘空间、CPU时间、进程表项等——导致系统无法使用（或引发系统崩溃）。编写无缺陷的代码或设计无漏洞的硬件通常是不可能的；如果恶意用户代码的开发者知晓内核或硬件漏洞，他们就会加以利用。即便在成熟且应用广泛的内核（如Linux）中，人们也在不断发现新的安全漏洞[1]。在内核中设计防护机制以应对其自身存在漏洞的可能性是很有必要的，例如断言检查、类型校验、栈保护页等。最后，用户代码与内核代码之间的界限有时会变得模糊：部分拥有特权的用户级进程可能提供核心服务，实际上成为操作系统的一部分；而在某些操作系统中，特权用户代码还可以向内核注入新代码，比如Linux的可加载内核模块。

- xv6也有虚拟的进程地址空间，地址从0开始；struct proc
  ```
  // Per-process state
  struct proc {
    struct spinlock lock;
  
    // p->lock must be held when using these:
    enum procstate state;        // Process state
    void *chan;                  // If non-zero, sleeping on chan
    int killed;                  // If non-zero, have been killed
    int xstate;                  // Exit status to be returned to parent's wait
    int pid;                     // Process ID
  
    // wait_lock must be held when using this:
    struct proc *parent;         // Parent process
  
    // these are private to the process, so p->lock need not be held.
    uint64 kstack;               // Virtual address of kernel stack
    uint64 sz;                   // Size of process memory (bytes)
    pagetable_t pagetable;       // User page table   地址空间在这里
    struct trapframe *trapframe; // data page for trampoline.S
    struct context context;      // swtch() here to run process
    struct file *ofile[NOFILE];  // Open files
    struct inode *cwd;           // Current directory
    char name[16];               // Process name (debugging)
  };
  ```
- Each process has a thread of execution (or thread for short) that executes the process’s instruc-
tions. A thread can be suspended and later resumed. To switch transparently between processes,
the kernel suspends the currently running thread and resumes another process’s thread. Much of
the state of a thread (local variables, function call return addresses) is stored on the thread’s stacks.
Each process has two stacks: a user stack and a kernel stack (p->kstack).
- A process can make a system call by executing the RISC-V ecall instruction. This instruction
raises the **hardware privilege level** and changes the program counter to a kernel-defined entry point.
- p->state indicates whether the process is allocated, ready to run, running, waiting for I/O, or
exiting.
- p->pagetable holds the process’s page table, in the format that the RISC-V hardware ex-
pects. Xv6 causes the **paging hardware** to use a process’s p->pagetable when executing that
process in user space.
- 幻相：In summary, a process bundles two design ideas: an address space to give a process the illusion
of its own memory, and, a thread, to give the process the illusion of its own CPU. In xv6, a process
consists of one address space and one thread. In real operating systems a process may have more
than one thread to take advantage of multiple CPUs.
- The loader loads the xv6 kernel into memory at physical address 0x80000000. The reason it
places the kernel at 0x80000000 rather than 0x0 is because the address range 0x0:0x80000000
contains I/O devices.
The instructions at _entry set up a stack so that xv6 can run C code. Xv6 declares space
for an initial stack, stack0, in the file start.c (kernel/start.c:11). The code at _entry loads the
stack pointer register sp with the address stack0+4096, the top of the stack, because the stack
on RISC-V grows down. Now that the kernel has a stack, _entry calls into C code at start

- 启动过程：
  - RISC-V 上电后由引导程序加载 xv6 内核至物理地址 0x80000000，在机器模式下从 _entry 启动并初始化栈，进而转入 C 代码 start 函数。
  - start 完成机器模式下的配置后，通过 mret 指令切换到监管模式，跳转到 main 函数完成设备与子系统初始化，并创建首个用户进程。
  - 首个进程通过 exec 系统调用启动 /init 程序，最终 init 进程打开控制台并启动 shell，系统正式运行。
  <img width="650" height="700" alt="image" src="https://github.com/user-attachments/assets/286afed7-0e7c-43b9-b4c4-2dd565c33761" />

#### further reading
- The RISC-V Reader: An Open Architecture Atlas
- xv6代码导读：https://www.bilibili.com/video/BV1DY4y1a7YD/?vd_source=2211521a84d324c18aba00755ad3bcec
- The UNMO Time-Sharing System Dennis M. Ritchie and Ken Thompson：https://dl.acm.org/doi/epdf/10.1145/357980.358014
- https://jyywiki.cn/pages/OS/manuals/unix-v6-book.pdf
- process max: https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/riscv.h#L363
- entry.S：https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/entry.S#L7
- main.c https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/main.c#L11
