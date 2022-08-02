# Lab 10 mmap

## 实验目的

## 实验步骤

先在UPROGS中加入mmaptest

```makefile
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_mmaptest\
```

和sys_mmap，sys_munmap的系统调用

kernel/syscall.c

```c
extern uint64 sys_wait(void);
extern uint64 sys_write(void);
extern uint64 sys_uptime(void);
extern uint64 sys_mmap(void);
extern uint64 sys_munmap(void);

static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
//..
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
[SYS_mmap]    sys_mmap,
[SYS_munmap]  sys_munmap,
};
```

user/usys.pl

```perl
entry("sbrk");
entry("sleep");
entry("uptime");
entry("mmap");
entry("munmap");
```

user/user.h

```c
char* sbrk(int);
int sleep(int);
int uptime(void);
void* mmap(void *, int, int, int, int, uint);
int munmap(void *, int);
```

定义一个vma结构在kernel/proc.h中

```c
#define VMASIZE 16
struct vma
{
  struct file *file;
  int fd;
  int used;
  uint64 addr;
  int length;
  int prot;
  int flags;
  int offset;
};
```

并加到proc中

```c
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  struct vma vma[VMASIZE];     // vma
};
```

接下来修改usertrap，处理page fault（提前声明头文件）

```c
#include "fcntl.h"
#include "fs.h"
#include "sleeplock.h"
#include "file.h"

//...

} else if((which_dev = devintr()) != 0){
    // ok
  } 
  else if (r_scause() == 13 || r_scause() == 15) {
    uint64 va = r_stval();
    if(va >= p->sz || va > MAXVA || PGROUNDUP(va) == PGROUNDDOWN(p->trapframe->sp))
      p->killed = 1;
	  else {
      struct vma *vma = 0;
      for (int i = 0; i < VMASIZE; i++) {
        if (p->vma[i].used == 1 && va >= p->vma[i].addr && 
            va < p->vma[i].addr + p->vma[i].length) {
          vma = &p->vma[i];
          break;
        }
      }

      if(vma) {
        va = PGROUNDDOWN(va);
        uint64 offset = va - vma->addr;
        uint64 mem = (uint64)kalloc();
        if(mem == 0) {
          p->killed = 1;
        } 
        else {
          memset((void*)mem, 0, PGSIZE);
		      ilock(vma->file->ip);
          readi(vma->file->ip, 0, mem, offset, PGSIZE);
          iunlock(vma->file->ip);
          int flag = PTE_U;
          if(vma->prot & PROT_READ) flag |= PTE_R;
          if(vma->prot & PROT_WRITE) flag |= PTE_W;
          if(vma->prot & PROT_EXEC) flag |= PTE_X;
          if(mappages(p->pagetable, va, PGSIZE, mem, flag) != 0) {
            kfree((void*)mem);
            p->killed = 1;
          }
        }
      }
    }
  }
  else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
```

在kernel/sysfile.c中实现这两个函数

```c
uint64 sys_mmap(void) {
  uint64 addr;
  int length, prot, flags, fd, offset;
  struct proc *p = myproc();
  struct file *file;
  if(argaddr(0, &addr) || argint(1, &length) || argint(2, &prot) ||
    argint(3, &flags) || argfd(4, &fd, &file) || argint(5, &offset)) 
    return -1;

  if(!file->writable && (prot & PROT_WRITE) && flags == MAP_SHARED)
    return -1;

  length = PGROUNDUP(length);
  if(p->sz > MAXVA - length)
    return -1;

  for(int i = 0; i < VMASIZE; i++) {
    if(p->vma[i].used == 0) {
      p->vma[i].used = 1;
      p->vma[i].addr = p->sz;
      p->vma[i].length = length;
      p->vma[i].prot = prot;
      p->vma[i].flags = flags;
      p->vma[i].fd = fd;
      p->vma[i].file = file;
      p->vma[i].offset = offset;
      filedup(file);
      p->sz += length;
      return p->vma[i].addr;
    }
  }
  return -1;
}

uint64
sys_munmap(void)
{
  uint64 addr;
  int length;
  struct proc *p = myproc();
  struct vma *vma = 0;
  if(argaddr(0, &addr) || argint(1, &length))
    return -1;
  addr = PGROUNDDOWN(addr);
  length = PGROUNDUP(length);
  for(int i = 0; i < VMASIZE; i++) {
    if (addr >= p->vma[i].addr || addr < p->vma[i].addr + p->vma[i].length) {
      vma = &p->vma[i];
      break;
    }
  }
  if(vma == 0) 
    return 0;
  if(vma->addr == addr) {
    vma->addr += length;
    vma->length -= length;
    if(vma->flags & MAP_SHARED)
      filewrite(vma->file, addr, length);
    uvmunmap(p->pagetable, addr, length/PGSIZE, 1);
    if(vma->length == 0) {
      fileclose(vma->file);
      vma->used = 0;
    }
  }
  return 0;
}
```

修改uvmcopy和uvmunmap，避免因不合法而panic

uvmunmap

```c
if((*pte & PTE_V) == 0)
      // panic("uvmunmap: not mapped");
      continue;
```

uvmcopy

```c
    if((*pte & PTE_V) == 0)
      // panic("uvmcopy: page not present");
      continue;
```

最后修改exit和fork

```c
  for(int i = 0; i < VMASIZE; i++) {
    if(p->vma[i].used) {
      if(p->vma[i].flags & MAP_SHARED)
        filewrite(p->vma[i].file, p->vma[i].addr, p->vma[i].length);
      fileclose(p->vma[i].file);
      uvmunmap(p->pagetable, p->vma[i].addr, p->vma[i].length/PGSIZE, 1);
      p->vma[i].used = 0;
    }
  }

  begin_op();
  iput(p->cwd);
  end_op();
```

fork

```c
  np->state = RUNNABLE;

  for(int i = 0; i < VMASIZE; i++) {
    if(p->vma[i].used){
      memmove(&(np->vma[i]), &(p->vma[i]), sizeof(p->vma[i]));
      filedup(p->vma[i].file);
    }
  }

  release(&np->lock);

  return pid;
```

## 实验中遇到的问题及解决办法

![image-20220724105722937](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220724105722937.png)

## 实验心得