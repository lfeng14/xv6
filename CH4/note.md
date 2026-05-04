- riscv硬件极简主义，复杂交给软件处理
- three kinds of event which cause the CPU to set aside ordinary execution of instructions
and force a transfer of control to special code that handles the event.
  - One situation is a system
call, when a user program executes the ecall instruction to ask the kernel to do something for
it.
- Another situation is an exception: an instruction (user or kernel) does something illegal, such as
divide by zero or use an invalid virtual address.
- The third situation is a device interrupt, when a
device signals that it needs attention, for example when the disk hardware finishes a read or write
request.
- CPU 会暂停常规指令执行、跳转至专用处理代码的事件分三类：
1. **系统调用**：**用户态**执行 ecall 指令，请求内核提供服务。
2. **异常**：用户态或内核态指令出现非法操作，如除零、访问无效虚拟地址。
3. **设备中断**：硬件设备发出请求信号，如磁盘完成读写。
- 本书将各类异常情形统称为**陷阱（trap）**。
  - 陷阱触发时正在执行的程序需**无感知恢复执行**，保持透明性，设备中断尤其需要这一特性。
  - 陷阱标准处理流程：
     - 陷阱触发，控制权转入内核
     - 内核保存寄存器及运行状态
     - 内核执行对应处理程序（系统调用、设备驱动等）
     - 内核恢复保存的状态、退出陷阱
     - 原程序从断点继续执行
- 这份逻辑是由CPU硬件设计和操作系统内核共同规定的。CPU硬件会在trap发生时，根据当前特权级自动切换到更高特权级，并跳转到内核预设的处理入口地址，这是硬件层面的固定行为。而内核会提前在这些入口地址放置对应的处理代码，比如保存现场、判断trap类型、执行具体处理逻辑，这部分是操作系统内核的设计。所以硬件负责“跳转触发”，内核负责“具体处理”，两者配合实现了trap的统一处理机制。
- trap 机制
  - CPU 陷入（trap）时仅保存**程序计数器pc**，**不切换内核页表、不切换内核栈、不保存其他寄存器**，这些工作由内核软件完成。
  - CPU 仅做最小操作是为赋予软件灵活性，部分系统会省略页表切换以提升陷入性能。
  - 为追求速度省略陷入流程步骤存在安全风险，贸然省略会破坏用户态与内核态隔离。
  - CPU 必须跳转到内核指定的**stvec**入口地址执行，保障系统隔离安全。
- 在RISC-V中，内核会把trap处理代码（trap handler）放在物理内存的固定低地址区域，并且这段区域在所有进程的页表中都被映射为“有效且可执行”，或者内核会在trap入口的前几条指令中，先手动将satp寄存器切换到内核自己的页表——而内核页表中肯定映射了trap handler的地址。这样一来，无论trap发生时CPU用的是哪个进程的页表，都能正确找到并执行trap处理代码，不需要提前“缓存”，而是靠页表映射保证地址可访问。
  - “映射好”指的是页表中已经建立了虚拟地址到物理地址的对应关系，也就是一级、二级、三级页表项里已经填好了trap处理代码所在的物理页地址，并且标记为有效、可执行。这样CPU通过页表翻译虚拟地址时，能直接找到物理内存中的指令，不需要提前缓存指令，缓存是CPU硬件自动做的，但前提是页表已经映射正确。
  ```
  一、整体路径概览
  
  从用户态陷入内核再返回，主要经历 4 个关键函数：
  
  进入内核：
  uservec (trampoline.S) → usertrap (trap.c)
  
  返回用户空间：
  usertrapret (trap.c) → userret (trampoline.S)
  
  前两个运行在 trampoline 页和内核页表下，后两个负责切回用户页表并恢复寄存器。
  二、核心设计约束（为什么要有 trampoline？）
  
  RISC-V 硬件在发生陷阱时 不切换页表。
  
  陷阱发生时，CPU 仍然使用 当前进程的用户页表。
  因此，stvec 寄存器的陷阱处理入口地址 必须在用户页表中有有效映射，否则无法取指。
  但处理代码很快就要切换到内核页表（修改 satp），切换后 同一段代码地址必须继续有效，所以内核页表也必须在同样虚拟地址映射这段代码。
  解决方案：
  一个专用的 trampoline 页，同时映射在：
  
  每个进程的用户页表（虚拟地址 TRAMPOLINE，位于用户地址空间最高处，不含 PTE_U 标志，保证 trap 时以 supervisor 模式执行）
  内核页表（同样虚拟地址 TRAMPOLINE）
  stvec 指向的就是这个 trampoline 页内的 uservec 函数。
  
  三、进入内核：uservec → usertrap
  
  1. uservec（汇编，trampoline.S）
  
  挑战： 刚进入时所有 32 个寄存器保存的是被中断的用户代码的值，还没有任何空闲寄存器来存数据。
  利用 sscratch 寄存器：
  
  内核在初始化时为每个进程在 sscratch 中放了一个“临时暂存值”。
  uservec 第一条指令 csrw sscratch, a0 实际上把 a0 和 sscratch 交换，从而 a0 变得可用。
  主要工作：
  
  用 a0 指向 trapframe（虚拟地址 TRAPFRAME，在每个进程用户页表中映射，紧挨在 trampoline 下面）。
  将全部 32 个用户寄存器保存到 trapframe 中（包括从 sscratch 恢复的用户 a0）。
  从 trapframe 中读出：
  
  内核栈地址
  当前 CPU 的 hartid
  usertrap 函数地址
  内核页表地址
  将 satp 切换为内核页表（此时仍能继续执行，因为 trampoline 页在内核页表中以相同虚拟地址映射）。
  调用 usertrap（进入 C 代码）。
  2. usertrap（C 语言，trap.c）
  
  修改 stvec：指向 kernelvec，这样如果在内核态再发生陷阱，由另一套处理逻辑负责。
  保存 sepc（用户程序计数器），因为后面可能调度到其他进程，会覆盖它。
  判断陷阱原因：
  
  系统调用（ecall） → 调用 syscall()，并将 sepc 加 4（因为 RISC-V 的 ecall 会停在当前指令，返回后需要执行下一条）。
  设备中断 → 调用 devintr()。
  异常 → 杀死进程。
  检查是否应该让出 CPU（如定时器中断）或进程被杀死。
  四、返回用户空间：usertrapret → userret
  
  1. usertrapret（trap.c）
  
  恢复 stvec 指向 uservec（为下一次用户态陷阱做准备）。
  重新填充 trapframe 中 uservec 需要的数据（如内核栈、页表等，因为可能换了进程）。
  将 sepc 设回之前保存的用户程序计数器。
  调用 userret（传入用户页表指针）。
  2. userret（汇编，trampoline.S）
  
  切换 satp 回用户页表（此时 trampoline 页在用户页表中有相同映射，所以代码继续执行）。
  从 trapframe 中恢复所有用户寄存器（通过 a0 指向的 trapframe 地址）。
  最后执行 sret 指令，返回用户空间，继续执行用户程序。
  五、关键图示理解
  
  虚拟地址	用户页表映射	内核页表映射	用途
  TRAMPOLINE	trampoline 页（无 PTE_U）	trampoline 页	存放 uservec, userret
  TRAPFRAME	该进程的 trapframe 页（无 PTE_U）	该进程的 trapframe 页（物理地址）	保存/恢复寄存器，状态交换
  两个地址分别位于用户地址空间的最高两个页，避免与用户程序自身堆栈冲突。
  六、一句话总结
  
  硬件不切页表 → 需要 trampoline 页同时映射在用户和内核空间 → stvec 指向 uservec（在 trampoline 页）→ 进入后先保存寄存器到 trapframe，再切换页表，最后调用 usertrap 处理实际逻辑；返回时通过 usertrapret 和 userret 对称地恢复并 sret。
  这个设计保证了从用户态陷入内核时，无论页表如何切换，陷阱处理代码都能连续执行，且所有用户寄存器都能被安全保存与恢复。
  
  如果你对其中某个步骤（比如 sscratch 的初始设置，或为什么 sepc 需要加 4）还想深入了解，可以继续问我。
  ```
- xv6 中一个系统调用（以 exec 为例）从用户空间到内核实现的完整路径
  ```
  一、用户空间的准备工作（以 initcode.S 为例）
  
  设置参数：将系统调用的参数放入寄存器
  
  a0、a1 等存放具体参数（exec 需要程序路径和参数数组）
  设置系统调用号：将 SYS_exec（定义在 kernel/syscall.h）放入 a7 寄存器
  执行 ecall 指令：触发陷阱，陷入内核
  系统调用号与内核中的 syscalls 函数指针数组的索引一一对应（kernel/syscall.c:107）。
  二、内核入口路径
  
  ecall 触发后，硬件按照之前梳理的陷阱流程执行：
  
  uservec（汇编） → usertrap（C） → syscall（C）
  
  uservec：保存用户寄存器，切换内核页表，调用 usertrap
  usertrap：判断陷阱原因，如果是 ecall，则调用 syscall
  三、syscall 函数分发执行
  
  c
  // kernel/syscall.c:132
  void syscall(void)
  {
      int num = p->trapframe->a7;   // 从 trapframe 中取出系统调用号
      if (num > 0 && num < NELEM(syscalls) && syscalls[num]) {
          p->trapframe->a0 = syscalls[num]();   // 调用具体函数，返回值保存在 a0
      } else {
          printf(...); p->trapframe->a0 = -1;   // 无效调用号返回 -1
      }
  }
  从 trapframe->a7 中取出用户空间的 a7 值（即系统调用号 SYS_exec）
  用该号索引 syscalls 数组，得到函数指针 sys_exec
  调用 sys_exec()，将其返回值存入 p->trapframe->a0
  四、返回值传递
  
  RISC-V 的 C 调用约定规定：返回值放在 a0 寄存器
  当内核完成系统调用，最终通过 userret 和 sret 返回用户空间时，会恢复所有寄存器，其中包括 trapframe->a0 被写回真实的 a0 寄存器
  因此，用户空间的 exec() 函数就能得到内核返回的值：
  
  成功：0 或正数
  失败：负数（通常 -1）
  五、整体流程图（文字版）
  
  text
  用户程序 (initcode.S)
    ① 设 a0, a1 = 参数
    ② 设 a7 = SYS_exec
    ③ ecall
          ↓
  内核 (trap路径)
    uservec → usertrap → syscall
          ↓
    syscall: num = trapframe->a7
          ↓
    sys_exec() 执行具体逻辑
          ↓
    返回值存入 trapframe->a0
          ↓
    usertrapret → userret → sret
          ↓
  用户程序
    恢复 a0 = 返回值
    exec() 调用完成
  关键点总结
  
  阶段	关键寄存器/变量	作用
  用户态	a7	存放系统调用号
  用户态	a0, a1...	存放参数
  陷入内核	trapframe->a7	保存的用户 a7
  系统调用分发	syscalls[num]	函数指针表
  返回值	trapframe->a0	内核将返回值放在这里
  返回用户态	a0	恢复后用户程序读到的返回值
  如果还需要了解 sys_exec 具体如何解析参数或 syscalls 数组的定义，我可以继续展开。
  什么是 ecall
  我们被问到“什么是 ecall”，这是 RISC-V 指令集中的一条指令。需要解释它在 xv6 上下文中作为系统调用触发指令的作用。回答应该清晰简洁。
  ecall 是 RISC-V 指令集中的一条环境调用指令（Environment Call）。它用于在用户态（U-mode） 主动陷入内核态（S-mode）发起服务请求，最常见的用途就是实现系统调用。
  
  在 xv6 中的具体作用
  
  当用户程序执行 ecall 时：
  
  触发异常：CPU 产生一个“环境调用”类型的陷阱（trap）。
  提高特权级：从用户模式（U-mode）切换到监督模式（S-mode）。
  跳转到陷阱处理程序：硬件会读取 stvec 寄存器中的地址，跳转到该地址执行（在 xv6 中就是 uservec 函数）。
  保存现场：ecall 之前的用户程序状态（包括返回地址 sepc）会被硬件/软件保存，以便日后恢复。
  为什么系统调用要用 ecall？
  
  安全性：用户程序不能直接访问内核数据或执行内核代码。ecall 提供了一种受控的入口，内核可以在陷阱处理程序中检查请求的合法性。
  参数传递约定：调用方将系统调用号放入 a7 寄存器，参数放入 a0、a1 等寄存器。内核从 trapframe 中读取这些值，分发到具体的系统调用函数（如 sys_exec）。
  其他架构的类比
  
  x86：int 0x80 或 syscall 指令
  ARM：svc（Supervisor Call）
  一句话总结
  
  ecall 是 RISC-V 中用户程序请求内核服务的指令，它主动触发一个陷阱，使 CPU 跳到内核预先设置好的入口（uservec），从而执行系统调用。

  ```
- trapframe不只是传递系统调用参数，它主要用来保存trap发生时的上下文，包括32个用户寄存器的值。当发生系统调用这种trap时，用户态传入的参数会存在用户寄存器里，而这些寄存器会被保存到trapframe中，内核从trapframe里读取寄存器值就能拿到系统调用参数。所以传递参数是它的一个功能，核心还是保存和恢复上下文。
- 这段内容解释了 xv6 中当陷阱（trap）发生在内核态时的处理流程。这与用户态陷阱有本质不同：内核已经运行在自己的地址空间和栈上，因此不需要切换页表，但需要保存被打断的内核线程的寄存器，并处理可能由定时器中断触发的调度。
  ```
  一、核心区别对比
  
  方面	用户态陷阱	内核态陷阱
  stvec 指向	uservec	kernelvec
  页表状态	用户页表 → 需切换到内核页表	已经是内核页表
  栈	没有内核栈 → 先保存到 trapframe，再切换内核栈	已有内核栈 → 直接压入当前内核栈
  寄存器保存位置	p->trapframe	当前内核线程的栈上
  返回路径	usertrapret + userret + sret	sret 直接返回
  能否发生异常	异常通常杀死进程	异常在内核中是致命错误 → panic
  二、入口：kernelvec（汇编，kernel/kernelvec.S:12）
  
  特点：当 stvec 指向 kernelvec 时，说明 CPU 已经在内核态运行。
  
  执行步骤：
  
  栈可用：sp 指向当前内核线程的有效内核栈（因为被打断的代码本身就是内核代码）。
  保存所有寄存器：将 32 个通用寄存器压入当前内核栈。
  
  为什么压到栈上而不是 trapframe？因为寄存器值属于该被打断的内核线程，保存在自己的栈上最自然，并且如果发生线程切换，原线程的寄存器会安全留在其栈上，新线程有自己的栈。
  调用 kerneltrap 进入 C 代码。
  三、处理函数：kerneltrap（kernel/trap.c:135）
  
  1. 保存关键控制寄存器
  
  保存 sepc（被打断内核代码的返回地址）和 sstatus（状态寄存器），因为后续可能调用 yield 修改它们。
  2. 区分陷阱类型
  
  设备中断：调用 devintr() 处理（如时钟中断、磁盘中断）。如果返回值非零，说明是设备中断。
  其他情况：如果 devintr 返回 0，说明不是设备中断，则必须是异常（如缺页、非法指令等）。在内核态发生异常是 xv6 无法恢复的严重错误 → 调用 panic() 并停机。
  3. 特殊处理：定时器中断导致调度
  
  如果当前陷阱是定时器中断，并且当前运行的是进程的内核线程（而非调度器线程 scheduler），则调用 yield() 让出 CPU，让其他线程有机会运行。
  这体现了 xv6 的抢占式调度：定时器中断强制当前线程让出 CPU。
  未来该线程恢复时，会继续从 yield 返回，然后执行 kerneltrap 的后续恢复代码。
  4. 恢复控制寄存器并返回
  
  恢复之前保存的 sepc 和 sstatus。
  返回到 kernelvec 的恢复代码。
  四、返回：kernelvec 恢复并 sret
  
  kerneltrap 返回到 kernelvec 中 call kerneltrap 之后的指令（kernel/kernelvec.S:50）：
  
  从栈上弹出之前保存的 32 个寄存器。
  执行 sret 指令：
  
  将 sepc 复制到 pc
  根据 sstatus 恢复特权级
  继续执行被打断的内核代码
  整个过程没有页表切换，因为始终在内核页表中。
  五、关键细节：stvec 的切换与中断窗口
  
  何时切换 stvec 到 kernelvec：在 usertrap 中（kernel/trap.c:29），从用户态进入内核后，立即将 stvec 设置为 kernelvec。
  危险窗口：内核刚进入，但 stvec 仍指向 uservec 的短暂时间内，如果发生设备中断，就会用 uservec 处理内核态中断，导致错误。
  RISC-V 的保护机制：硬件在开始处理陷阱时会自动禁用中断。xv6 在所有陷阱处理代码中，直到完成 stvec 切换之后才重新启用中断。因此，这个窗口期内不会有中断发生，保证了安全。
  六、整体流程图
  
  text
  内核代码执行
     ↓ 定时器中断 / 设备中断 / 异常
  硬件：stvec → kernelvec
     ↓
  kernelvec (汇编)：
    - 将所有寄存器压入当前内核栈
    - 调用 kerneltrap
     ↓
  kerneltrap (C)：
    - 保存 sepc, sstatus
    - if (devintr() == 0) → panic()  (异常)
    - else if (定时器中断 && 非调度器线程) → yield()
    - 恢复 sepc, sstatus
    - 返回
     ↓
  kernelvec 恢复：
    - 弹出寄存器
    - sret 返回被打断的内核代码
  七、一句话总结
  
  内核态陷阱使用专用的 kernelvec 入口，直接在当前内核栈上保存/恢复寄存器，由 kerneltrap 处理设备中断（可触发调度）并 panic 任何异常。stvec 在用户→内核切换时被更新，且利用硬件自动关中断的特性避免了窗口期错误。
  如果你希望进一步了解 yield 如何实现线程切换，或者 devintr 如何区分不同的设备中断，我可以继续帮你梳理。

  ```
- 这段内容详细介绍了 xv6 中对页错误（page fault）的基本处理方式，以及现代操作系统如何利用页错误实现更高级的内存管理功能。下面按层次梳理。
  ```
  一、xv6 的简单处理（对比现代 OS）
  
  用户态页错误：杀死该进程（因为无法恢复）。
  内核态页错误：直接 panic（内核 bug）。
  缺点：功能单一，没有利用页错误做优化。
  现代 OS 利用页错误 实现写时拷贝、惰性分配、请求调页、交换到磁盘等高级特性。
  二、页错误的基本概念
  
  1. 什么时候触发页错误？
  
  虚拟地址在页表中 没有映射（PTE_V = 0）
  或 PTE 权限位（PTE_R/PTE_W/PTE_X/PTE_U）禁止当前操作（例如写一个只读页）
  2. RISC-V 的三种页错误类型
  
  类型	说明
  Load page fault	加载指令无法转换地址
  Store page fault	存储指令无法转换地址
  Instruction page fault	PC 中的地址无法转换
  scause 寄存器：指示页错误类型
  stval 寄存器：包含出错的虚拟地址
  三、关键技术 1：写时拷贝（Copy-on-Write, COW）Fork
  
  传统 fork（xv6 的做法）
  
  uvmcopy 为子进程分配新物理页并复制父进程所有内存 → 开销大，尤其很多内存可能根本不会被写。
  COW fork 的思路
  
  父、子进程最初共享所有物理页。
  但页表项中 清除 PTE_W 标志（只读）。
  任何一方试图写共享页 → CPU 触发存储页错误。
  内核的陷阱处理程序：
  
  分配一个新物理页
  将故障地址对应的物理页内容复制到新页
  修改当前进程的 PTE，指向新页，并加上 PTE_W 标志
  重新执行导致故障的写指令（此时写入成功）
  引用计数：每个物理页记录被多少个页表引用，当计数降为 0 时才释放。
  优点
  
  fork 快速（无需复制内存）。
  实际只在写入时复制（通常大部分内存不会被复制，例如 fork 后立即 exec 的场景）。
  对应用程序完全透明。
  优化：如果写时发生页错误，但该物理页的引用计数为 1（仅本进程引用），则无需复制，直接改 PTE 为可写即可。
  四、关键技术 2：惰性分配（Lazy Allocation）
  
  动机
  
  应用程序通过 sbrk 请求内存时，往往申请的量远大于实际使用的量。
  如果立刻分配并清零，浪费时间和物理内存。
  
  实现两步骤
  
  sbrk 系统调用：只增加进程的堆大小（p->sz），不分配物理页，也不创建 PTE。
  页错误处理：
  
  当进程首次访问新范围内的某个虚拟地址时，触发页错误
  内核分配一个物理页，创建 PTE 映射到该地址，并填入页面（可能清零）
  重新执行指令
  优点
  
  避免分配永远不会使用的页面。
  大块内存申请（如 1GB）的成本分散到实际访问时，启动响应快。
  对应用透明。
  代价
  
  每次访问新页都需要一次页错误（内核/用户态切换开销）。
  优化：一次页错误批量分配多个连续页，或优化入口路径。
  五、关键技术 3：请求调页（Demand Paging）
  
  传统 exec（xv6）
  
  立即从文件系统加载整个可执行文件的 text 和 data 到内存 → 启动慢，尤其大程序。
  请求调页做法
  
  exec 创建页表，但将 text 和 data 段的 PTE 标记为无效（不分配物理页）。
  当 CPU 第一次取指或访问数据时，触发指令页错误或加载页错误。
  内核从磁盘（可执行文件）中读取对应的页内容到物理页，建立映射，然后恢复执行。
  优点
  
  启动快（按需加载）
  减少内存占用（只加载当前需要的部分）
  六、关键技术 4：交换到磁盘（Paging to Disk）
  
  场景
  
  物理内存不足，需要将一些不常用的页面换出到磁盘的交换区。
  
  机制
  
  内核在页表中将换出的页的 PTE 标记为无效（但保留元数据，如磁盘块号）。
  当进程访问该页 → 页错误 → 内核：
  
  若 RAM 有空闲，直接读回磁盘内容到新物理页
  若 RAM 满了，先换出另一个页面（写回磁盘），再换入当前需要的页
  更新 PTE 指向新的物理页，恢复执行。
  要求
  
  良好的引用局部性：工作集能放入 RAM，频繁换页（抖动）性能差。
  对应用透明。
  七、其他应用 & 总结
  
  页错误机制还可以实现：
  
  自动扩展栈（栈向下增长时触发页错误，分配新页）
  内存映射文件（mmap）：将文件内容映射到地址空间，按需通过页错误读入。
  核心思想归纳
  
  技术	触发时机	内核动作	优势
  COW fork	写共享只读页	复制页，改 PTE 为可写	快速 fork，节省内存
  惰性分配	访问未映射的堆区	分配物理页，创建映射	避免浪费，sbrk 快
  请求调页	访问未加载的代码/数据页	从文件读页，建映射	启动快，省内存
  交换到磁盘	访问已换出的页	换入页面，可能先换出	支持超过物理内存的大小
  所有这些技术都依赖于页表权限 + 页错误异常的组合，并且对用户程序透明。现代操作系统通过它们大幅提高了内存利用效率和响应速度。
  ```



- 根据 xv6 的实现，可以把陷阱（trap）分为三种场景：**来自用户空间的陷阱**（含系统调用、异常、中断）、**来自内核空间的异常**（非中断错误）、**来自内核空间的中断**（如定时器中断）。下面用表格清晰对比三者的处理流程差异：
  
  | 项目 | 用户态陷阱 (User Trap) | 内核态异常 (Kernel Exception) | 内核态中断 (Kernel Interrupt) |
  |------|------------------------|-------------------------------|-------------------------------|
  | **触发示例** | `ecall`（系统调用）、用户态缺页、非法指令、用户态运行时设备中断 | 内核代码执行时发生缺页、非法访问、非法指令等 | 内核运行时收到定时器中断、磁盘中断等 |
  | **当前特权级** | U-mode → S-mode | S-mode（已在内核） | S-mode（已在内核） |
  | **`stvec` 指向** | `uservec` | `kernelvec` | `kernelvec` |
  | **页表** | 开始为用户页表，很快切换到内核页表 | 已是内核页表，不变 | 已是内核页表，不变 |
  | **栈** | 起初无内核栈 → 从 `trapframe` 加载内核栈 | 当前内核线程的栈 | 当前内核线程的栈 |
  | **寄存器保存位置** | `p->trapframe` | 当前内核栈 | 当前内核栈 |
  | **汇编入口** | `uservec` (trampoline.S) | `kernelvec` (kernelvec.S) | `kernelvec` (kernelvec.S) |
  | **C 处理函数** | `usertrap` | `kerneltrap` | `kerneltrap` |
  | **对异常的反应** | 杀死用户进程 | `panic()`（致命错误，不返回） | N/A（中断不算异常） |
  | **对中断的反应** | 调用 `devintr()`，若是定时器中断可能调用 `yield()` 让出 CPU | N/A（不会发生中断？实际可能，但处理方式相同） | 调用 `devintr()`，若是定时器中断且非调度器线程则 `yield()` |
  | **是否可能发生调度** | 是（定时器中断或用户主动让出） | 否（直接 panic） | 是（定时器中断时） |
  | **返回路径** | `usertrapret` + `userret` + `sret` | 无（panic 后系统停机） | `kernelvec` 恢复寄存器 + `sret` |
  | **关键硬件/软件机制** | `sscratch` 交换、trampoline 页、`satp` 切换 | 直接使用内核栈 | 直接使用内核栈 |

- 补充说明
  1. **设备中断** 可在用户态或内核态触发，但 xv6 都通过 `devintr()` 统一处理。区别仅在于入口（`uservec` vs `kernelvec`）和返回后的行为（是否触发调度）。
  2. **内核态异常** 在 xv6 中被认为是无法恢复的 bug，因此直接 `panic`。生产操作系统会尝试恢复（如内核缺页可能通过 `copyout` 修复）。
  3. **内核态中断** 中最重要的是定时器中断，它实现了 xv6 的**抢占式调度**——即使内核正在执行进程的上下文（如系统调用中），也会被定时器中断打断并可能切换到其他线程。
