- This interface has been so successful that modern operating systems—BSD, Linux, macOS, Solaris, and even, to a lesser extent, Microsoft Windows—have Unix-like interfaces. Understanding xv6 is a good start toward understanding any of these systems and many others.shows, xv6 takes the traditional form of a **single** kernel, a special program that provides
services to running programs.
  <img width="686" height="298" alt="image" src="https://github.com/user-attachments/assets/80248e26-4e6b-4ac3-88d9-dd63e26e0a82" />
- The shell is an ordinary program that reads commands from the user and executes them. The fact that the shell is a user program, and not part of the kernel, illustrates the power of the system call interface: there is nothing special about the shell.
- Xv6 time-shares processes: it transparently switches the available CPUs among the set of processes waiting to execute
- A process may create a new process using the fork system call. fork gives the new process an exact copy of the calling process’s memory, both instructions and data. fork returns in both the original and new processes
  <img width="1416" height="1008" alt="image" src="https://github.com/user-attachments/assets/2a74205a-d0ec-4009-b95a-fbbccd440d5c" />

#### further reading
- https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L1
