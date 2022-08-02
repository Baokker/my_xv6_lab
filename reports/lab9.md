

# Lab 9

## 实验目的

## 实验步骤

### Large files

就像课本里的inode一样，11个块，1个一级页表，1个二级页表

首先修改and增加几个宏定义，将原先直接访问的12个块改成11个，再定义二级页表，最大文件

```c
#define FSMAGIC 0x10203040

#define NDIRECT 11 // modify
#define NINDIRECT (BSIZE / sizeof(uint))
#define NDINDIRECT (NINDIRECT * NINDIRECT)
#define MAXFILE (NDIRECT + NINDIRECT + NDINDIRECT)
```

由于NDIRECT变了，再修改dinode和inode

```c
// On-disk inode structure
struct dinode {
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT + 2];   // Data block addresses (add one)
};
// ...
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?
  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT + 2];
};
```

最后修改bmap，仿照前面页表的寻找方式。记得bread完brelse

```c
  // second level
  bn -= NINDIRECT;
  if (bn < NDINDIRECT){
    if ((addr = ip->addrs[NDIRECT + 1]) == 0)
      ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if ((addr = a[bn / NINDIRECT]) == 0){
      a[bn / NINDIRECT] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);

    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if ((addr = a[bn % NINDIRECT]) == 0){
      a[bn % NINDIRECT] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

修改itrunc，确保释放包括两级页表在内的所有块

```c
  struct buf *bp2;
  uint *a2;

  if(ip->addrs[NDIRECT + 1]){
    bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j]){
        bp2 = bread(ip->dev, a[j]);
        a2 = (uint*)bp2->data;
        for (i = 0; i < NINDIRECT; i++){
          if (a2[i])
            bfree(ip->dev, a2[i]);
        }
        brelse(bp2);
        bfree(ip->dev, a[j]);
        a[j] = 0;
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
    ip->addrs[NDIRECT + 1] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```

### Symbolic links

首先，像lab2一样，给symlink添加系统调用

user/usys.pl

```perl
entry("sbrk");
entry("sleep");
entry("uptime");
entry("symlink");
```

kernel/syscall.h

```c
#define SYS_link   19
#define SYS_mkdir  20
#define SYS_close  21
#define SYS_symlink 22
```

kernel/syscall.c

```c
extern uint64 sys_wait(void);
extern uint64 sys_write(void);
extern uint64 sys_uptime(void);
extern uint64 sys_symlink(void);

static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
// ..
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
[SYS_symlink]   sys_symlink,
};
```

接下来，在kernel/stat.h中增加T_SYMLINK新类型

```c
#define T_DIR     1   // Directory
#define T_FILE    2   // File
#define T_DEVICE  3   // Device
#define T_SYMLINK 4   // symlink
```

kernel/fcntl.h中添加O_NOFOLLOW

```c
#define O_RDWR    0x002
#define O_CREATE  0x200
#define O_TRUNC   0x400
#define O_NOFOLLOW 0x800
```

修改sys_open，设置最大搜索深度为20

```c
      end_op();
      return -1;
    }
  } 
  else {
    int max_depth = 20, depth = 0;
    while (1) {
      if ((ip = namei(path)) == 0) {
        end_op();
        return -1;
      }
      ilock(ip);
      if (ip->type == T_SYMLINK && (omode & O_NOFOLLOW) == 0) {
        if (++depth > max_depth) {
          iunlockput(ip);
          end_op();
          return -1;
        }
        if (readi(ip, 0, (uint64)path, 0, MAXPATH) < MAXPATH) {
          iunlockput(ip);
          end_op();
          return -1;
        }
        iunlockput(ip);
      }
      else
        break;
    }
  }

  if(ip->type == T_DIR && omode != O_RDONLY){
    iunlockput(ip);
    end_op();
    return -1;
  }

  if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
    iunlockput(ip);
    end_op();
```

实现sym_link

```c
uint64 sys_symlink(void){
  char path[MAXPATH], target[MAXPATH];
  struct inode *ip;

  if (argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
    return -1;

  begin_op();
  if ((ip = create(path, T_SYMLINK, 0, 0)) == 0) {
    end_op();
    return -1;
  }

  if (writei(ip, 0, (uint64)target, 0, MAXPATH) < MAXPATH){
    iunlockput(ip);
    end_op();
    return -1;
  }

  iunlockput(ip);
  end_op();
  return 0;
} 
  1  
user/user.h
```



## 实验中遇到的问题及解决办法

![image-20220724100600976](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220724100600976.png)

## 实验心得