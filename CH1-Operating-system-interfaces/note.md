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
- The xv6 shell uses the above calls to run programs on behalf of users. The main structure of the shell is simple; see main (user/sh.c:146). The main loop reads a line of input from the user with getcmd. Then it calls fork, which creates a copy of the shell process. The parent calls wait, while the child runs the command. For example, if the user had typed “echo hello” to the shell, runcmd would have been called with “echo hello” as the argument. runcmd (user/sh.c:55) runs the actual command. For “echo hello”, it would call exec (user/sh.c:79). If exec succeeds then the child will execute instructions from echo instead of runcmd. At some point echo will call exit, which will cause the parent to return from wait in main (user/sh.c:146).
- You might wonder why fork and exec are not combined in a single call; we will see later that the shell exploits the separation in its implementation of I/O redirection.
- the file descriptor interface abstracts away the differences between files, pipes, and devices, making them all look like streams of bytes. We’ll refer to input and output as I/O.
- Internally, the xv6 kernel uses the file descriptor as an index into a per-process table, so that every process has a private space of file descriptors starting at zero. By convention, a process reads from file descriptor 0 (standard input), writes output to file descriptor 1 (standard output), and writes error messages to file descriptor 2 (standard error).
```
char *argv[2];
argv[0] = "cat";
argv[1] = 0;
if(fork() == 0) {
  close(0); // 子进程关闭标准输入文件描述后，下回打开的文件就会占用标准输入
  open("input.txt", O_RDONLY);
  exec("cat", argv);
}
```
- 等价命令
```
cat < input.txt
```
- Now it should be clear why it is helpful that fork and exec are separate calls: between the two, the shell has a chance to redirect the child’s I/O without disturbing the I/O setup of the main shell。
  ```
  if(fork() == 0) {
    write(1, "hello ", 6);
    exit(0);
  } else {
    wait(0);
    write(1, "world\n", 6);
  }
  ```
  - 等价命令
  ```
  (echo hello; echo world) >output.txt
  ```
  - 等价命令：dup复制的文件描述符共享offset，fork也有相同作用共享offset；
  - The dup system call duplicates an existing file descriptor, returning a new one that refers to the same underlying I/O object
  ```
  fd = dup(1);
  write(1, "hello ", 6);
  write(fd, "world\n", 6);
  ```
  - ls existing-file non-existing-file > tmp1 2>&1. The 2>&1 tells the shell to give the command a file descriptor 2 that is a duplicate of descriptor 1。所以先close(2); dup(1)
  - File descriptors are a powerful abstraction, because they hide the details of what they are connected to: a process writing to file descriptor 1 may be writing to a file, to a device like the console, or to a pipe.
- pipe: p0是单向 读 文件描述符，p1是单项 写 文件描述符；阻塞 直至**所有**写端关闭；
  ```
  cat test.c
  #include <stdio.h>
  #include <unistd.h>
  #include <stdlib.h>
  #include <sys/wait.h>
  
  int main() {
      int p[2];
      char *argv[2];
      argv[0] = "wc";
      argv[1] = NULL;  // 必须以 NULL 结束
  
      // 创建管道
      pipe(p);
  
      if(fork() == 0) {
          // 子进程：把管道读端 → 标准输入 0
          close(0);
          dup(p[0]);      // 重定向！
  
          close(p[0]);
          close(p[1]); // 注意两个写端，需要先关闭子进程写端，这样父进程写端关闭后，管道读端才不回阻塞；
  
          // ✅ 这里必须用 execvp，不能用 exec！
          execvp("wc", argv);
  
          // 如果执行失败
          perror("exec failed");
          exit(1);
      } else {
          // 父进程：写入数据
          close(p[0]);
          write(p[1], "hello world\n", 12);
          close(p[1]);
  
          // 等待子进程结束
          wait(NULL);
      }
  
      return 0;
  }
  franz@ubuntu:~/src/leetcode$ ./test 
        1       2      12  # 1行 2个单词 12个字符 等价echo -n -e "hello world\n" | wc
  ```
  ```
  Pipes may seem no more powerful than temporary files: the pipeline
    echo hello world | wc
  could be implemented without pipes as
    echo hello world >/tmp/xyz; wc </tmp/xyz
  ``
- 如果是级连管道，shell会解析成树，通过管道连接不同的叶子节点数据流；这里可以挖一挖代码来看看是怎么处理的；虽然会创建管理者用来等待子进程结束，但是数据流是边运行边留给下一个叶子节点；
  ```
        [Shell] (原始 Shell 进程，不在树内，它只等待前台作业)
           |
           | fork 一个子进程来管理整个 a|b|c
           v
      [管理者 P2] (内部节点，负责管道2，连接 P1 和 c)
           |
           |----------------------------------|
           |                                  |
    fork 左子 (P1)                       fork 右子 (c 命令)
           |                                  |
           v                                  v
   [管理者 P1] (内部节点，负责管道1)      [c 命令进程] (叶子节点)
           |
           |------------------|
           |                  |
      fork 左子 (a)      fork 右子 (b)
           |                  |
           v                  v
   [a 命令进程] (叶子)    [b 命令进程] (叶子)
  ```
- File system：The xv6 file system provides data files, which contain uninterpreted byte arrays：A file’s name is distinct from the file itself; the same underlying file, called an inode, can have multiple names, called links. Each link consists of an entry in a directory; the entry contains a file name and a reference to an inode. An inode holds metadata about a file, including its type (file or directory or device), its length, the location of the file’s content on disk, and the number of links to a file.
- The link system call creates another file system name referring to the **same inode** as an existing file. This fragment creates a new file named both a and b：open("a", O_CREATE|O_WRONLY); link("a", "b");
- inode表示文件的元数据，如果实际文件、软链接 通过fstat获取的stat struct中的字段inode都是同一个：
  ```
  struct stat {
    int dev; // File system’s disk device
    uint ino; // Inode number
    short type; // Type of file
    short nlink; // Number of links to file
    uint64 size; // Size of file in bytes
  };
  ```
- 普通外置命令是通过fork一个子进程来执行，父进程wait子进程结束；但是cd命令却不是，因为cd是shell内置命令；如果cd是外置命令，那么仅改变了子进程的工作目录；
- The Unix system call interface has been standardized through the Portable Operating System Interface (POSIX) standard. Xv6 is not POSIX compliant: it is missing many system calls (including basic ones such as lseek), and many of the system calls it does provide differ from the standard. Our main goals for xv6 are simplicity and clarity while providing a simple UNIX-like system-call interface. 
#### further reading
- https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L1
