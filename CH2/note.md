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

#### further reading
- The RISC-V Reader: An Open Architecture Atlas
- xv6代码导读：https://www.bilibili.com/video/BV1DY4y1a7YD/?vd_source=2211521a84d324c18aba00755ad3bcec
- The UNMO Time-Sharing System Dennis M. Ritchie and Ken Thompson：https://dl.acm.org/doi/epdf/10.1145/357980.358014
- https://jyywiki.cn/pages/OS/manuals/unix-v6-book.pdf
