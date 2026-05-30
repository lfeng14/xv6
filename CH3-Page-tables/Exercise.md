##### Description
Write a user program that grows its address space by one byte by calling sbrk(1). Run
the program and investigate the page table for the program before the call to sbrk and after
the call to sbrk. How much space has the kernel allocated? What does the PTE for the new
memory contain?

#### Analysis
- 执行现象
  <img width="430" height="250" alt="image" src="https://github.com/user-attachments/assets/22554606-fdd4-4402-ad99-c7005baf634b" />

- patch
  ```patch
  From f4c8c32ce231fc9b33fa6b0467b373f428238386 Mon Sep 17 00:00:00 2001
  From: feng <feng@fengdeMacBook-Pro.local>
  Date: Sat, 18 Apr 2026 16:54:48 +0800
  Subject: [PATCH] exercise 3
  
  ---
   Makefile |  1 +
   grow.c   | 23 +++++++++++++++++++++++
   2 files changed, 24 insertions(+)
   create mode 100644 grow.c
  
  diff --git a/Makefile b/Makefile
  index 4b1b923..48beff6 100644
  --- a/Makefile
  +++ b/Makefile
  @@ -181,6 +181,7 @@ UPROGS=\
    _stressfs\
    _usertests\
    _wc\
  +        _grow\
    _zombie\
   
   fs.img: mkfs README $(UPROGS)
  diff --git a/grow.c b/grow.c
  new file mode 100644
  index 0000000..82b592b
  --- /dev/null
  +++ b/grow.c
  @@ -0,0 +1,23 @@
  +#include "types.h"
  +#include "user.h"
  +
  +int stdout = 1;
  +
  +int main(int argc, char *argv[])
  +{
  +    // 1. 记录调用前的堆边界（可以使用一些内核打印或自定义系统调用查看页表）
  +    printf(stdout, "Before sbrk(1)...\n");
  +    
  +    char *a = sbrk(1);
  +    if(a == (char*)-1){
  +        printf(stdout, "sbrk failed\n");
  +        exit();
  +    }
  +
  +    printf(stdout, "After sbrk(1). Address allocated: %p\n", a);
  +
  +    // 为了观察页表，通常需要在这里触发某种内核打印
  +    // 或者在 qemu 中使用 ctrl-a c 进入监控模式查看
  +    
  +    exit();
  +}
  -- 
  2.50.1 (Apple Git-155)
  

  ```
