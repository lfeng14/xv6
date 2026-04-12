#### 描述
Add a system call to xv6 that returns the amount of free memory available.

#### 实现
- 启动xv6 on mac:
  ```
  git clone git@github.com:mit-pdos/xv6-public.git
  brew install i686-elf-gcc # install cross compiler chain
  make
  make qemu
  ```
  ```
  cat > qemu.patch << EOF
  diff --git a/Makefile b/Makefile
  index 09d790c..b45a12b 100644
  --- a/Makefile
  +++ b/Makefile
  @@ -29,7 +29,7 @@ OBJS = \
          vm.o\
   
   # Cross-compiling (e.g., on Mac OS X)
  -# TOOLPREFIX = i386-jos-elf
  +TOOLPREFIX = i686-elf-
   
   # Using native tools (e.g., on X86 Linux)
   #TOOLPREFIX = 
  @@ -76,9 +76,9 @@ AS = $(TOOLPREFIX)gas
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
   
  @@ -157,7 +157,7 @@ _forktest: forktest.o $(ULIB)
          $(OBJDUMP) -S _forktest > forktest.asm
   
   mkfs: mkfs.c fs.h
  -       gcc -Werror -Wall -o mkfs mkfs.c
  +       gcc  -o mkfs mkfs.c
   
   # Prevent deletion of intermediate files, e.g. cat.o, after first build, so
   # that disk image changes after first build are persistent until clean.  More
  EOF
  ```

#### further reading
- https://medium.com/@majumdersusanka068/running-xv6-on-apple-silicon-mac-3b7d06e6c9be
