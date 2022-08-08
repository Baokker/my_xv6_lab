# Lab 6 cow

## 实验目的

这次跟上回的lazy策略挺像。主要是为了避免fork时子进程拷贝父进程，但是又不一定都用到，浪费了太多时间，因此采用COW（copy-on-write）策略，直到页面真的需要时再copy。

## 实验步骤

在进行修改之前，先进行一些准备工作

kernel/riscv.h模仿`PTE_W`等，定义一个`PTE_RSW`，表示RISC-V中的RSW（reserved for software）位

```c
#define PTE_X (1L << 3)
#define PTE_U (1L << 4) // 1 -> user can access

#define PTE_RSW (1L << 8) // RSW
```

接下来在kernel/kalloc.c中定义INDEX宏，用物理地址除4096（即右移12位）实现定位

```c
#define INDEX(pa) (((char*)pa - (char*)PGROUNDUP((uint64)end)) >> 12)
```

然后修改kmem结构，增加计数

```c
struct {
  struct spinlock lock;
  struct run *freelist;
  // add ref
  struct spinlock ref_lock;
  uint *ref_count;
} kmem;
```

最后定义一些def.h中声明所需函数，这些函数的主要目的是抽象计数的相关操场

```c
void*           kalloc(void);
void            kfree(void *);
void            kinit(void);
int             get_kmem_ref(void *pa);
void            add_kmem_ref(void *pa);
void            acquire_ref_lock();
void            release_ref_lock();
```

并中在kernel/kalloc.c实现它们

```c
int get_kmem_ref(void *pa){
  return kmem.ref_count[INDEX(pa)];
}

void add_kmem_ref(void *pa){
  kmem.ref_count[INDEX(pa)]++;
}

void acquire_ref_lock(){
  acquire(&kmem.ref_lock);
}

void release_ref_lock(){
  release(&kmem.ref_lock);
}
```

准备完成后，开始修改。首先是kernel/vm.c中的uvmcopy，注释掉释放页面的代码，清除PTE_W位，同时给ref计数+1。

```c
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    // set unwrite and set rsw
    *pte = ((*pte) & (~PTE_W)) | PTE_RSW;
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    // stop copy
    // if((mem = kalloc()) == 0)
    //   goto err;
    // memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, pa, flags) != 0){
      // kfree(mem);
      goto err;
    }
    add_kmem_ref((void*)pa);
  }
  return 0;
```

接下来修改usertrap。由于cow中只有写页面错误，只需要考虑对应rscause=15的情况。当cow页面中出现page fault，释放新页面，复制旧页面到新页面中，并且设新的PTE的PTE_W位为true。

```c
 } 
  else if (r_scause() == 15) { // write page fault
    uint64 va = PGROUNDDOWN(r_stval());
    pte_t *pte;
    if (va > p->sz || (pte = walk(p->pagetable, va, 0)) == 0){
      p->killed = 1;
      goto end;
    }

    if (((*pte) & PTE_RSW) == 0 || ((*pte) & PTE_V) == 0 || ((*pte) & PTE_U) == 0){
      p->killed = 1;
      goto end;
    }

    uint64 pa = PTE2PA(*pte);
    acquire_ref_lock();
    uint ref = get_kmem_ref((void*)pa);
    if (ref == 1){
      *pte = ((*pte) & (~PTE_RSW)) | PTE_W;
    }
    else {
      char* mem = kalloc();
      if (mem == 0){
        p->killed = 1;
        release_ref_lock();
        goto end;
      }

      memmove(mem, (char*)pa, PGSIZE);
      uint flag = (PTE_FLAGS(*pte) | PTE_W) & (~PTE_RSW);
      if (mappages(p->pagetable, va, PGSIZE, (uint64)mem,flag) != 0){
        kfree(mem);
        p->killed = 1;
        release_ref_lock();
        goto end;
      }
      kfree((void*)pa);
    }
    release_ref_lock();
  }
  else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }
```

walk是外部函数，因此还需要在kernel/trap.c中添加外部声明

```c
#include "proc.h"
#include "defs.h"

extern pte_t* walk(pagetable_t, uint64, int);

struct spinlock tickslock;
```

修改mappages，避免其在PTE_V非法时panic

```c
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    // if(*pte & PTE_V)
    //   panic("remap");
    *pte = PA2PTE(pa) | perm | PTE_V;
    if(a == last)
      break;
```

接下来先修改kinit：

```c
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&kmem.ref_lock, "ref");

  //total physical pages
  uint64 physical_pages = ((PHYSTOP - (uint64)end) >> 12) + 1;
  physical_pages = ((physical_pages * sizeof(uint)) >> 12) + 1;
  kmem.ref_count = (uint*) end;
  uint64 offset = physical_pages << 12;
  freerange(end + offset, (void*)PHYSTOP);
}
```

修改freerange，初始化计数

```c
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE){
    kmem.ref_count[INDEX((void*)p)] = 1;
    kfree(p);
  }
}
```

最后修改kfree，计数为0后才释放

```c
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // check if the number reaches 0
  acquire(&kmem.lock);
  if (--kmem.ref_count[INDEX(pa)]){
    release(&kmem.lock);
    return;
  }
  release(&kmem.lock);

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);
```

修改kalloc，计数设为1

```c
  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r){
    kmem.freelist = r->next;
    kmem.ref_count[INDEX((void*)r)] = 1;
  }
  release(&kmem.lock);
```

最后修改copyout，操作类似usertraps

```c
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;
  pte_t* pte;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    if(va0 >= MAXVA)  
      return -1;
    if((pte = walk(pagetable, va0, 0)) == 0)
      return -1;
    if (((*pte & PTE_V) == 0) || ((*pte & PTE_U)) == 0) 
      return -1;

    pa0 = PTE2PA(*pte);
    if (((*pte & PTE_W) == 0) && (*pte & PTE_RSW)){
      acquire_ref_lock();
      if (get_kmem_ref((void*)pa0) == 1) {
          *pte = (*pte | PTE_W) & (~PTE_RSW);
      }
      else {
        char* mem = kalloc();
        if (mem == 0){
          release_ref_lock();
          return -1;
        }
        memmove(mem, (char*)pa0, PGSIZE);
        uint new_flags = (PTE_FLAGS(*pte) | PTE_RSW) & (~PTE_W);
        if (mappages(pagetable, va0, PGSIZE, (uint64)mem, new_flags) != 0){
          kfree(mem);
          release_ref_lock();
          return -1;
        }
        kfree((void*)pa0);
      }
      release_ref_lock();
    }

    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
```

## 实验中遇到的问题及解决办法

位操作的运算比划了半天，简简单单几个& ~还是太精妙了

还有就是<<12实现计算页数真的很精妙。

## 实验心得

![image-20220731162838248](img\image-20220731162838248.png)

