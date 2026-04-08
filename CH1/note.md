- This interface has been so successful that modern operating systems—BSD, Linux, macOS, Solaris, and even, to a lesser extent, Microsoft Windows—have Unix-like interfaces. Understanding xv6 is a good start toward understanding any of these systems and many others.shows, xv6 takes the traditional form of a **single** kernel, a special program that provides
services to running programs.
  <img width="340" height="150" alt="image" src="https://github.com/user-attachments/assets/80248e26-4e6b-4ac3-88d9-dd63e26e0a82" />
- The shell is an ordinary program that reads commands from the user and executes them. The fact that the shell is a user program, and not part of the kernel, illustrates the power of the system call interface: there is nothing special about the shell.
- Xv6 time-shares processes: it transparently switches the available CPUs among the set of processes waiting to execute
- A process may create a new process using the fork system call. fork gives the new process an exact copy of the calling process’s memory, both instructions and data. fork returns in both the original and new processes
  <img width="700" height="500" alt="image" src="https://github.com/user-attachments/assets/2a74205a-d0ec-4009-b95a-fbbccd440d5c" />
- The exit system call causes the calling process to stop executing and to release resources such as memory and open files. Exit takes an integer status argument, conventionally 0 to indicate success and 1 to indicate failure. The wait system call returns the PID of an exited (or killed) child of the current process and copies the exit status of the child to the address passed to wait; if none of the caller’s children has exited, wait waits for one to do so. If the caller has no children, wait immediately returns -1. If the parent doesn’t care about the exit status of a child, it can pass a 0 address to wait.
- Although the child has the same memory contents as the parent initially, the parent and child are executing with separate memory and separate registers: changing a variable in one does not affect the other. For example, when the return value of wait is stored into pid in the parent process, it doesn’t change the variable pid in the child. The value of pid in the child will still be zero.
  ```
  int pid = fork();
  if(pid > 0){
    printf("parent: child=%d\n", pid);
    pid = wait((int *) 0);
    printf("child %d is done\n", pid);
  } else if(pid == 0){
    printf("child: exiting\n");
    exit(0);
  } else {
    printf("fork error\n");
  }
  ```
#### further reading
- https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L1
