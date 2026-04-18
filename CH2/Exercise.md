#### 描述
Add a system call to xv6 that returns the amount of free memory available.

#### 实现
- 启动xv6 on mac:
  ```bash
  git clone git@github.com:mit-pdos/xv6-public.git
  brew install i686-elf-gcc # install cross compiler chain
  make
  make qemu
  ```
- 运行
  <img width="420" height="60" alt="image" src="https://github.com/user-attachments/assets/02b09ab6-a117-4a97-aa9a-2ce6f12c4338" />

  ```patch
  cat 0001-exercise-2.patch
  From 97153221b83c752031788004d368227a5db82190 Mon Sep 17 00:00:00 2001
  From: feng <feng@fengdeMacBook-Pro.local>
  Date: Sat, 18 Apr 2026 16:50:10 +0800
  Subject: [PATCH] exercise 2
  
  ---
   Makefile  | 11 ++++++-----
   defs.h    |  1 +
   kalloc.c  | 19 +++++++++++++++++++
   meminfo.c | 14 ++++++++++++++
   syscall.c |  2 ++
   syscall.h |  1 +
   sysproc.c |  7 +++++++
   user.h    |  1 +
   usys.S    |  1 +
   9 files changed, 52 insertions(+), 5 deletions(-)
   create mode 100644 meminfo.c
  
  diff --git a/Makefile b/Makefile
  index 09d790c..4b1b923 100644
  --- a/Makefile
  +++ b/Makefile
  @@ -29,7 +29,7 @@ OBJS = \
   	vm.o\
   
   # Cross-compiling (e.g., on Mac OS X)
  -# TOOLPREFIX = i386-jos-elf
  +TOOLPREFIX = i686-elf-
   
   # Using native tools (e.g., on X86 Linux)
   #TOOLPREFIX = 
  @@ -76,12 +76,12 @@ AS = $(TOOLPREFIX)gas
   LD = $(TOOLPREFIX)ld
   OBJCOPY = $(TOOLPREFIX)objcopy
   OBJDUMP = $(TOOLPREFIX)objdump
  -CFLAGS = -fno-pic -static -fno-builtin -fno-strict-aliasing -O2 -Wall -MD -ggdb -m32 -Werror -fno-omit-frame-pointer
  +CFLAGS = -fno-pic -static -fno-builtin -fno-strict-aliasing -O2  -MD -ggdb  -fno-omit-frame-pointer
   CFLAGS += $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
  -ASFLAGS = -m32 -gdwarf-2 -Wa,-divide
  +ASFLAGS = -gdwarf-2 -Wa,-divide
   # FreeBSD ld wants ``elf_i386_fbsd''
   LDFLAGS += -m $(shell $(LD) -V | grep elf_i386 2>/dev/null | head -n 1)
  -
  +CFLAGS += -Wno-int-conversion
   # Disable PIE when possible (for Ubuntu 16.10 toolchain)
   ifneq ($(shell $(CC) -dumpspecs 2>/dev/null | grep -e '[^f]no-pie'),)
   CFLAGS += -fno-pie -no-pie
  @@ -157,7 +157,7 @@ _forktest: forktest.o $(ULIB)
   	$(OBJDUMP) -S _forktest > forktest.asm
   
   mkfs: mkfs.c fs.h
  -	gcc -Werror -Wall -o mkfs mkfs.c
  +	gcc  -o mkfs mkfs.c
   
   # Prevent deletion of intermediate files, e.g. cat.o, after first build, so
   # that disk image changes after first build are persistent until clean.  More
  @@ -174,6 +174,7 @@ UPROGS=\
   	_kill\
   	_ln\
   	_ls\
  +	_meminfo\
   	_mkdir\
   	_rm\
   	_sh\
  diff --git a/defs.h b/defs.h
  index 82fb982..fa76e2b 100644
  --- a/defs.h
  +++ b/defs.h
  @@ -68,6 +68,7 @@ char*           kalloc(void);
   void            kfree(char*);
   void            kinit1(void*, void*);
   void            kinit2(void*, void*);
  +int             countfreemem(void);
   
   // kbd.c
   void            kbdintr(void);
  diff --git a/kalloc.c b/kalloc.c
  index 14cd4f4..e6b0b80 100644
  --- a/kalloc.c
  +++ b/kalloc.c
  @@ -94,3 +94,22 @@ kalloc(void)
     return (char*)r;
   }
   
  +// Return the amount of free memory in bytes.
  +int
  +countfreemem(void)
  +{
  +  struct run *r;
  +  int count = 0;
  +
  +  if(kmem.use_lock)
  +    acquire(&kmem.lock);
  +  r = kmem.freelist;
  +  while(r) {
  +    count++;
  +    r = r->next;
  +  }
  +  if(kmem.use_lock)
  +    release(&kmem.lock);
  +  return count * PGSIZE;
  +}
  +
  diff --git a/meminfo.c b/meminfo.c
  new file mode 100644
  index 0000000..f1a573d
  --- /dev/null
  +++ b/meminfo.c
  @@ -0,0 +1,14 @@
  +#include "types.h"
  +#include "stat.h"
  +#include "user.h"
  +
  +int
  +main(int argc, char *argv[])
  +{
  +  int free_bytes;
  +
  +  free_bytes = freemem();
  +  printf(1, "Free memory: %d bytes (%d KB)\n", free_bytes, free_bytes / 1024);
  +
  +  exit();
  +}
  diff --git a/syscall.c b/syscall.c
  index ee85261..38e4de0 100644
  --- a/syscall.c
  +++ b/syscall.c
  @@ -103,6 +103,7 @@ extern int sys_unlink(void);
   extern int sys_wait(void);
   extern int sys_write(void);
   extern int sys_uptime(void);
  +extern int sys_freemem(void);
   
   static int (*syscalls[])(void) = {
   [SYS_fork]    sys_fork,
  @@ -126,6 +127,7 @@ static int (*syscalls[])(void) = {
   [SYS_link]    sys_link,
   [SYS_mkdir]   sys_mkdir,
   [SYS_close]   sys_close,
  +[SYS_freemem] sys_freemem,
   };
   
   void
  diff --git a/syscall.h b/syscall.h
  index bc5f356..3776562 100644
  --- a/syscall.h
  +++ b/syscall.h
  @@ -20,3 +20,4 @@
   #define SYS_link   19
   #define SYS_mkdir  20
   #define SYS_close  21
  +#define SYS_freemem 22
  diff --git a/sysproc.c b/sysproc.c
  index 0686d29..806a7cd 100644
  --- a/sysproc.c
  +++ b/sysproc.c
  @@ -89,3 +89,10 @@ sys_uptime(void)
     release(&tickslock);
     return xticks;
   }
  +
  +// return the amount of free memory in bytes
  +int
  +sys_freemem(void)
  +{
  +  return countfreemem();
  +}
  diff --git a/user.h b/user.h
  index 4f99c52..2565461 100644
  --- a/user.h
  +++ b/user.h
  @@ -23,6 +23,7 @@ int getpid(void);
   char* sbrk(int);
   int sleep(int);
   int uptime(void);
  +int freemem(void);
   
   // ulib.c
   int stat(const char*, struct stat*);
  diff --git a/usys.S b/usys.S
  index 8bfd8a1..8a608bd 100644
  --- a/usys.S
  +++ b/usys.S
  @@ -29,3 +29,4 @@ SYSCALL(getpid)
   SYSCALL(sbrk)
   SYSCALL(sleep)
   SYSCALL(uptime)
  +SYSCALL(freemem)
  -- 
  2.50.1 (Apple Git-155)
  
  ```

#### further reading
- https://medium.com/@majumdersusanka068/running-xv6-on-apple-silicon-mac-3b7d06e6c9be
