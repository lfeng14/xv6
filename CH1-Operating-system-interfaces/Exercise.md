#### 描述
Write a program that uses UNIX system calls to “ping-pong” a byte between two processes over a pair of pipes, one for each direction. Measure the program’s performance, in exchanges per second.

#### 实现
- 首先管道是单向的，所以需要两个管道实现ping pong
- 一轮迭代：父进程 -> 子进程；子进程 -> 父进程；
- 构建：clang pingpong_linux.c -o pingpong -Wno-implicit-function-declaration
- 运行： ./pingpong
  ```
  $ ./pingpong
  Total exchanges: 1000
  Elapsed time: 53.154 ms (0.053154 sec)
  Performance: 18813.30 exchanges/second
  franz@ubuntu:~/src/xv6/CH1$ cat pingpong
  pingpong          pingpong_linux.c
  ```
- 源码：
  ```
  // pingpong_linux.c
  // 修改为可在 Linux 上运行的版本
  
  #include <stdio.h>      // printf
  #include <stdlib.h>     // exit
  #include <unistd.h>     // pipe, fork, read, write
  #include <sys/wait.h>   // wait
  #include <time.h>       // clock_gettime
  
  int main(int argc, char *argv[]) {
      int p1[2], p2[2];
      char buf = 'x';              // 要传输的字节
      int iterations = 1000;       // 循环次数
  
      // 1. 创建两根管道
      if (pipe(p1) < 0 || pipe(p2) < 0) {
          printf("pipe error\n");
          exit(1);
      }
  
      // 2. 记录开始时间（使用 Linux 的单调时钟，不受系统时间调整影响）
      struct timespec start, end;
      clock_gettime(CLOCK_MONOTONIC, &start);
  
      int pid = fork();
      if (pid < 0) {
          printf("fork error\n");
          exit(1);
      }
  
      if (pid == 0) {               // 子进程 (Pong)
          for (int i = 0; i < iterations; i++) {
              read(p1[0], &buf, 1);  // 等待父进程的“Ping”
              write(p2[1], &buf, 1); // 发送“Pong”回去
          }
          exit(0);
      } else {                      // 父进程 (Ping)
          for (int i = 0; i < iterations; i++) {
              write(p1[1], &buf, 1); // 发送“Ping”
              read(p2[0], &buf, 1);  // 等待子进程的“Pong”
          }
  
          // 3. 记录结束时间
          clock_gettime(CLOCK_MONOTONIC, &end);
  
          // 4. 计算耗时（单位：纳秒）
          long long elapsed_ns = (end.tv_sec - start.tv_sec) * 1000000000LL
                                 + (end.tv_nsec - start.tv_nsec);
  
          // 换算成秒和毫秒，方便展示
          double elapsed_sec = elapsed_ns / 1000000000.0;
          double elapsed_ms = elapsed_ns / 1000000.0;
  
          printf("Total exchanges: %d\n", iterations);
          printf("Elapsed time: %.3f ms (%.6f sec)\n", elapsed_ms, elapsed_sec);
  
          if (elapsed_sec > 0) {
              // 计算每秒交换次数 (Exchanges per second)
              double eps = iterations / elapsed_sec;
              printf("Performance: %.2f exchanges/second\n", eps);
          }
  
          wait(0);                  // 回收子进程
      }
  
      return 0;
  }
  ```
