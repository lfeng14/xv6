# Chapter 8: File system

## 概述

xv6文件系统提供类Unix的文件、目录和路径名。文件系统实现了多层抽象，每层都建立在下层之上。

## 8.1 概述

### 文件系统层次结构

```
文件描述符层 (File descriptor layer)
         ↓
路径名层 (Path name layer)
         ↓
目录层 (Directory layer)
         ↓
inode层 (Inode layer)
         ↓
日志层 (Logging layer)
         ↓
缓冲区缓存层 (Buffer cache layer)
         ↓
磁盘 (Disk)
```

### 各层职责

| 层 | 职责 |
|---|------|
| **文件描述符** | 统一文件、管道、设备的接口 |
| **路径名** | 解析路径名，查找目录 |
| **目录** | 管理目录内容，添加/删除条目 |
| **inode** | 管理单个文件的元数据和内容 |
| **日志** | 原子性多步更新，崩溃恢复 |
| **缓冲区缓存** | 缓存磁盘块，同步访问 |

### 磁盘布局

xv6磁盘分区布局：

```
[ 引导块 | superblock | 日志块 | inode块 | 位图块 | 数据块 ]
```

- **引导块**：包含引导加载程序
- **超级块（superblock）**：包含文件系统元数据
  - 总块数
  - 数据块数
  - inode数
  - 日志块数
- **日志块**：用于崩溃恢复
- **inode块**：存储inode
- **位图块**：跟踪哪些数据块已分配
- **数据块**：存储文件内容

### 超级块结构

```c
struct superblock {
  uint magic;       // 魔数
  uint size;        // 文件系统总块数
  uint nblocks;     // 数据块数
  uint ninodes;     // inode数
  uint nlog;        // 日志块数
  uint logstart;    // 第一个日志块
  uint inodestart;  // 第一个inode块
  uint bmapstart;   // 第一个位图块
};
```

## 8.2 缓冲区缓存层

### 缓冲区缓存的作用

1. **缓存**：缓存常用的磁盘块，减少磁盘I/O
2. **同步**：确保每个块在内存中只有一个副本，一次只有一个CPU使用

### 缓冲区结构

```c
struct buf {
  int valid;        // 数据是否已从磁盘读入
  int disk;         // 是否有未写入的更改
  uint dev;         // 设备号
  uint blockno;     // 块号
  struct sleeplock lock;  // 保护缓冲区内容
  uint refcnt;      // 引用计数
  struct buf *prev; // LRU链表
  struct buf *next;
  uchar data[BSIZE];  // 块数据
};
```

## 8.3 代码：缓冲区缓存

### 缓冲区缓存结构

```c
struct {
  struct spinlock lock;
  struct buf buf[NBUF];
  struct buf head;  // 链表头
} bcache;
```

### bget() - 获取缓冲区

```c
static struct buf* bget(uint dev, uint blockno) {
  struct buf *b;

  acquire(&bcache.lock);

  // 1. 缓冲区是否已缓存？
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // 2. 未缓存，回收未使用的缓冲区
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0 && b->disk == 0){
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}
```

### bread() - 读取块

```c
struct buf* bread(uint dev, uint blockno) {
  struct buf *b;

  b = bget(dev, blockno);
  if(!b->valid){
    iderw(b);  // 从磁盘读取
    b->valid = 1;
  }
  return b;
}
```

### bwrite() - 写入块

```c
void bwrite(struct buf *b) {
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  b->disk = 1;
  iderw(b);  // 写入磁盘
}
```

### brelse() - 释放缓冲区

```c
void brelse(struct buf *b) {
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  acquire(&bcache.lock);
  b->refcnt--;
  if(b->refcnt == 0){
    // 移到LRU链表头部
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
  release(&bcache.lock);
}
```

## 8.4 日志层

### 问题：崩溃恢复

考虑创建文件需要多个步骤：
1. 分配inode
2. 分配数据块
3. 写入inode指向数据块
4. 在目录中添加条目

如果系统在步骤2和3之间崩溃，文件系统会不一致。

### 解决方案：日志（Logging）

日志的思想：
1. 在写入实际块之前，先在日志中描述所有写入
2. 当日志中的所有写入都已记录后，标记为"提交"
3. 然后执行实际的写入
4. 写入完成后，清除日志

### 恢复

如果崩溃：
- 如果日志未提交：忽略日志，什么都不做
- 如果日志已提交：重新执行日志中的所有写入

## 8.5 日志设计

### 日志结构

日志位于磁盘的开头部分：
- **日志头块（header block）**：包含日志中块数和块号列表
- **日志数据块**：实际的块数据副本

### 日志写入规则

1. 调用`begin_op()`开始一组相关写入
2. 进行修改，调用`log_write()`而不是直接`bwrite()`
3. 调用`end_op()`提交修改

### 日志状态

1. **空闲**：日志头块的计数为0
2. **写入中**：系统调用正在写入日志块，日志头还未更新
3. **已提交**：日志头块的计数>0，表示可以将块复制到文件系统
4. **应用中**：正在从日志复制块到文件系统

## 8.6 代码：logging

### 日志超级块

```c
struct logheader {
  int n;
  int block[LOGSIZE];
};
```

### begin_op()

```c
void begin_op(void) {
  acquire(&log.lock);
  while(1){
    if(log.committing){
      sleep(&log, &log.lock);
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // 没有足够的空间，等待
      sleep(&log, &log.lock);
    } else {
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}
```

### log_write()

```c
void log_write(struct buf *b) {
  int i;

  acquire(&log.lock);
  if(log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  if(log.outstanding < 1)
    panic("log_write outside of trans");

  // 在日志中找一个槽
  for(i = 0; i < log.lh.n; i++){
    if(log.lh.block[i] == b->blockno)
      break;
  }
  log.lh.block[i] = b->blockno;
  if(i == log.lh.n)
    log.lh.n++;
  b->flags |= B_DIRTY;  // 防止eviction
  release(&log.lock);
}
```

### end_op()

```c
void end_op(void) {
  int commit = 0;

  acquire(&log.lock);
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    commit = 1;
    log.committing = 1;
  } else {
    // 还有未完成的操作，不提交
    wakeup(&log);
  }
  release(&log.lock);

  if(commit){
    commit_trans();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}
```

### commit_trans()

```c
static void commit_trans() {
  if(log.lh.n > 0){
    write_log();     // 将修改的块写入日志
    write_head();    // 写日志头（提交点）
    install_trans(); // 从日志读入并写入文件系统
    log.lh.n = 0;
    write_head();    // 清除日志
  }
}
```

## 8.7 代码：块分配器

### 位图（Bitmap）

xv6使用位图跟踪数据块分配：
- 每一位代表一个数据块
- 1 = 已分配，0 = 空闲

### balloc() - 分配块

```c
static uint balloc(uint dev) {
  int b, bi, m;
  struct buf *bp;

  for(b = 0; b < sb.size; b += BPB){
    bp = bread(dev, BBLOCK(b, sb));
    for(bi = 0; bi < BPB && b + bi < sb.size; bi++){
      m = 1 << (bi % 8);
      if((bp->data[bi/8] & m) == 0){
        // 找到空闲块
        bp->data[bi/8] |= m;
        log_write(bp);
        brelse(bp);
        bzero(dev, b + bi);
        return b + bi;
      }
    }
    brelse(bp);
  }
  panic("balloc: out of blocks");
}
```

### bfree() - 释放块

```c
static void bfree(uint dev, uint b) {
  struct buf *bp;
  int bi, m;

  bp = bread(dev, BBLOCK(b, sb));
  bi = b % BPB;
  m = 1 << (bi % 8);
  if((bp->data[bi/8] & m) == 0)
    panic("freeing free block");
  bp->data[bi/8] &= ~m;
  log_write(bp);
  brelse(bp);
}
```

## 8.8 Inode层

### Inode结构

```c
struct dinode {
  short type;           // 文件类型
  short major;          // 主设备号（仅设备文件）
  short minor;          // 次设备号（仅设备文件）
  short nlink;          // 链接数
  uint size;            // 文件大小（字节）
  uint addrs[NDIRECT+1];  // 数据块地址
};
```

### 文件类型

- `T_DIR`：目录
- `T_FILE`：普通文件
- `T_DEV`：设备文件

### 数据块地址

- `NDIRECT`（12）个直接块：直接指向数据块
- 1个间接块：指向一个包含256个块号的块

```c
#define NDIRECT 12
#define NINDIRECT (BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT)
```

### 内存中inode

```c
struct inode {
  uint dev;            // 设备号
  uint inum;           // inode号
  int ref;             // 引用计数
  struct sleeplock lock;
  int valid;           // inode是否已从磁盘读入
  short type;          // 文件类型
  short major;         // 主设备号
  short minor;         // 次设备号
  short nlink;         // 链接数
  uint size;           // 文件大小
  uint addrs[NDIRECT+1];  // 数据块地址
};
```

## 8.9 代码：Inodes

### iget() - 获取inode

```c
struct inode* iget(uint dev, uint inum) {
  struct inode *ip, *empty;

  acquire(&icache.lock);

  // 1. 检查是否已缓存
  empty = 0;
  for(ip = &icache.inode[0]; ip < &icache.inode[NINODE]; ip++){
    if(ip->ref > 0 && ip->dev == dev && ip->inum == inum){
      ip->ref++;
      release(&icache.lock);
      return ip;
    }
    if(empty == 0 && ip->ref == 0)
      empty = ip;
  }

  // 2. 分配新inode
  if(empty == 0)
    panic("iget: no inodes");

  ip = empty;
  ip->dev = dev;
  ip->inum = inum;
  ip->ref = 1;
  ip->valid = 0;
  release(&icache.lock);

  return ip;
}
```

### ilock() - 锁定inode

```c
void ilock(struct inode *ip) {
  struct buf *bp;
  struct dinode *dip;

  if(ip == 0 || ip->ref < 1)
    panic("ilock");

  acquiresleep(&ip->lock);

  if(ip->valid == 0){
    bp = bread(ip->dev, IBLOCK(ip->inum, sb));
    dip = (struct dinode*)bp->data + ip->inum%IPB;
    ip->type = dip->type;
    ip->major = dip->major;
    ip->minor = dip->minor;
    ip->nlink = dip->nlink;
    ip->size = dip->size;
    memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
    brelse(bp);
    ip->valid = 1;
    if(ip->type == 0)
      panic("ilock: no type");
  }
}
```

### iunlock() - 解锁inode

```c
void iunlock(struct inode *ip) {
  if(ip == 0 || !holdingsleep(&ip->lock) || ip->ref < 1)
    panic("iunlock");
  releasesleep(&ip->lock);
}
```

### iput() - 释放inode

```c
void iput(struct inode *ip) {
  acquire(&icache.lock);

  if(ip->ref == 1 && ip->valid && ip->nlink == 0){
    // 没有引用，没有链接，释放inode和块
    acquire(&ip->lock);
    release(&icache.lock);
    itrunc(ip);
    ip->type = 0;
    iupdate(ip);
    release(&ip->lock);
    acquire(&icache.lock);
    ip->valid = 0;
  }

  ip->ref--;
  release(&icache.lock);
}
```

## 8.10 代码：Inode内容

### bmap() - 查找文件块

```c
static uint bmap(struct inode *ip, uint bn) {
  uint addr, *a;
  struct buf *bp;

  // 直接块
  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  // 间接块
  if(bn < NINDIRECT){
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

### readi() - 读取inode

```c
int readi(struct inode *ip, char *dst, uint off, uint n) {
  uint tot, m;
  struct buf *bp;

  if(ip->type == T_DEV){
    if(ip->major < 0 || ip->major >= NDEV || !devsw[ip->major].read)
      return -1;
    return devsw[ip->major].read(ip, dst, n);
  }

  if(off > ip->size || off + n < off)
    return -1;
  if(off + n > ip->size)
    n = ip->size - off;

  for(tot = 0; tot < n; tot += m, off += m, dst += m){
    bp = bread(ip->dev, bmap(ip, off/BSIZE));
    m = min(n - tot, BSIZE - off%BSIZE);
    memmove(dst, bp->data + off%BSIZE, m);
    brelse(bp);
  }
  return n;
}
```

### writei() - 写入inode

```c
int writei(struct inode *ip, char *src, uint off, uint n) {
  uint tot, m;
  struct buf *bp;

  if(ip->type == T_DEV){
    if(ip->major < 0 || ip->major >= NDEV || !devsw[ip->major].write)
      return -1;
    return devsw[ip->major].write(ip, src, n);
  }

  if(off > ip->size || off + n < off)
    return -1;
  if(off + n > MAXFILE*BSIZE)
    return -1;

  for(tot = 0; tot < n; tot += m, off += m, src += m){
    bp = bread(ip->dev, bmap(ip, off/BSIZE));
    m = min(n - tot, BSIZE - off%BSIZE);
    memmove(bp->data + off%BSIZE, src, m);
    log_write(bp);
    brelse(bp);
  }

  if(n > 0 && off > ip->size){
    ip->size = off;
    iupdate(ip);
  }
  return n;
}
```

## 8.11 代码：目录层

### 目录条目

```c
struct dirent {
  ushort inum;
  char name[DIRSIZ];
};
```

### dirlookup() - 查找目录条目

```c
struct inode* dirlookup(struct inode *dp, char *name, uint *poff) {
  uint off, inum;
  struct dirent de;

  if(dp->type != T_DIR)
    panic("dirlookup not DIR");

  for(off = 0; off < dp->size; off += sizeof(de)){
    if(readi(dp, (char*)&de, off, sizeof(de)) != sizeof(de))
      panic("dirlookup read");
    if(de.inum == 0)
      continue;
    if(namecmp(name, de.name) == 0){
      if(poff) *poff = off;
      inum = de.inum;
      return iget(dp->dev, inum);
    }
  }
  return 0;
}
```

### dirlink() - 添加目录条目

```c
int dirlink(struct inode *dp, char *name, uint inum) {
  int off;
  struct dirent de;
  struct inode *ip;

  if(dp->type != T_DIR)
    panic("dirlink not DIR");

  if((ip = dirlookup(dp, name, 0)) != 0){
    iput(ip);
    return -1;
  }

  for(off = 0; off < dp->size; off += sizeof(de)){
    if(readi(dp, (char*)&de, off, sizeof(de)) != sizeof(de))
      panic("dirlink read");
    if(de.inum == 0)
      break;
  }

  strncpy(de.name, name, DIRSIZ);
  de.inum = inum;
  if(writei(dp, (char*)&de, off, sizeof(de)) != sizeof(de))
    panic("dirlink");
  return 0;
}
```

## 8.12 代码：路径名

### namex() - 解析路径名

```c
static struct inode* namex(char *path, int nameiparent, char *name) {
  struct inode *ip, *next;

  if(*path == '/')
    ip = iget(ROOTDEV, ROOTINO);
  else
    ip = idup(myproc()->cwd);

  while((path = skipelem(path, name)) != 0){
    ilock(ip);
    if(ip->type != T_DIR){
      iunlockput(ip);
      return 0;
    }
    if(nameiparent && *path == '\0'){
      iunlock(ip);
      return ip;
    }
    if((next = dirlookup(ip, name, 0)) == 0){
      iunlockput(ip);
      return 0;
    }
    iunlockput(ip);
    ip = next;
  }
  if(nameiparent){
    iput(ip);
    return 0;
  }
  return ip;
}
```

### namei() - 查找路径名

```c
struct inode* namei(char *path) {
  char name[DIRSIZ];
  return namex(path, 0, name);
}
```

### nameiparent() - 查找父目录

```c
struct inode* nameiparent(char *path, char *name) {
  return namex(path, 1, name);
}
```

## 8.13 文件描述符层

### 文件结构

```c
struct file {
  enum { FD_NONE, FD_PIPE, FD_INODE } type;
  int ref;              // 引用计数
  char readable;
  char writable;
  struct pipe *pipe;    // FD_PIPE
  struct inode *ip;     // FD_INODE
  uint off;             // FD_INODE
};
```

### 打开文件表

```c
struct {
  struct spinlock lock;
  struct file file[NFILE];
} ftable;
```

### 进程的文件描述符表

```c
struct proc {
  // ...
  struct file *ofile[NOFILE];  // 打开的文件
  // ...
};
```

## 8.14 代码：系统调用

### sys_open()

```c
int sys_open(void) {
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;

  if(argstr(0, path, MAXPATH) < 0 || argint(1, &omode) < 0)
    return -1;

  begin_op();

  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    if((ip = namei(path)) == 0){
      end_op();
      return -1;
    }
    ilock(ip);
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
  }

  if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
    if(f)
      fileclose(f);
    iunlockput(ip);
    end_op();
    return -1;
  }
  iunlock(ip);
  end_op();

  f->type = FD_INODE;
  f->ip = ip;
  f->off = 0;
  f->readable = !(omode & O_WRONLY);
  f->writable = (omode & O_WRONLY) || (omode & O_RDWR);
  return fd;
}
```

### sys_read()

```c
int sys_read(void) {
  struct file *f;
  int n;
  char *p;

  if(argfd(0, 0, &f) < 0 || argint(2, &n) < 0 || argptr(1, &p, n) < 0)
    return -1;
  return fileread(f, p, n);
}
```

### sys_write()

```c
int sys_write(void) {
  struct file *f;
  int n;
  char *p;

  if(argfd(0, 0, &f) < 0 || argint(2, &n) < 0 || argptr(1, &p, n) < 0)
    return -1;
  return filewrite(f, p, n);
}
```

### sys_close()

```c
int sys_close(void) {
  int fd;
  struct file *f;

  if(argfd(0, &fd, &f) < 0)
    return -1;
  myproc()->ofile[fd] = 0;
  fileclose(f);
  return 0;
}
```

### sys_fstat()

```c
int sys_fstat(void) {
  struct file *f;
  struct stat *st;

  if(argfd(0, 0, &f) < 0 || argptr(1, (void*)&st, sizeof(*st)) < 0)
    return -1;
  return filestat(f, st);
}
```

### sys_link()

```c
int sys_link(void) {
  char name1[MAXPATH], name2[MAXPATH];
  struct inode *ip, *dp;
  struct dirent de;

  if(argstr(0, name1, MAXPATH) < 0 || argstr(1, name2, MAXPATH) < 0)
    return -1;

  begin_op();
  if((ip = namei(name1)) == 0){
    end_op();
    return -1;
  }
  ilock(ip);
  if(ip->type == T_DIR){
    iunlockput(ip);
    end_op();
    return -1;
  }
  ip->nlink++;
  iupdate(ip);
  iunlock(ip);

  if((dp = nameiparent(name2, name1)) == 0){
    ilock(ip);
    ip->nlink--;
    iupdate(ip);
    iunlockput(ip);
    end_op();
    return -1;
  }
  ilock(dp);
  if(dp->dev != ip->dev || dirlink(dp, name1, ip->inum) < 0){
    iunlockput(dp);
    ilock(ip);
    ip->nlink--;
    iupdate(ip);
    iunlockput(ip);
    end_op();
    return -1;
  }
  iunlockput(dp);
  iput(ip);

  end_op();
  return 0;
}
```

### sys_unlink()

```c
int sys_unlink(void) {
  struct inode *ip, *dp;
  struct dirent de;
  char name[MAXPATH];
  uint off;

  if(argstr(0, name, MAXPATH) < 0)
    return -1;

  begin_op();
  if((dp = nameiparent(name, name)) == 0){
    end_op();
    return -1;
  }
  ilock(dp);
  if((ip = dirlookup(dp, name, &off)) == 0){
    iunlockput(dp);
    end_op();
    return -1;
  }
  ilock(ip);
  if(ip->nlink < 1)
    panic("unlink: nlink < 1");
  if(ip->type == T_DIR && !isdirempty(ip)){
    iunlockput(ip);
    iunlockput(dp);
    end_op();
    return -1;
  }

  memset(&de, 0, sizeof(de));
  if(writei(dp, (char*)&de, off, sizeof(de)) != sizeof(de))
    panic("unlink: writei");
  iunlockput(dp);
  ip->nlink--;
  iupdate(ip);
  iunlockput(ip);

  end_op();
  return 0;
}
```

## 8.15 现实世界

- 日志是一个通用的想法，用于数据库、文件系统等
- 现代文件系统（ext3/4, NTFS, HFS+）都使用日志
- 软更新（Soft updates）是日志的替代方案
- 写时复制（Copy-on-write）文件系统（ZFS, Btrfs）提供快照和更好的崩溃恢复
- 现代文件系统使用更大的块（4KB或更多）
- 扩展属性（Extended attributes）存储额外的元数据
- 日志结构化文件系统（Log-structured file systems）将所有写入追加到日志
- SSD优化的文件系统（F2FS）考虑闪存特性
- 目录索引（如B树或哈希）加速大目录查找

## 8.16 练习

（见书籍中的练习题目）
