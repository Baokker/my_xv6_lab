# Lab 3

## 实验目的

最难的一个实验！太痛苦了...

## 实验步骤

### Print a page table

设计一个vmprint函数，打印页表内容

首先在def.h中声明，这样可以在exec.c中调用

```c
// add vmprint
void            vmprint(pagetable_t pagetable);
```

再在kernel/vm.c中实现vmprint函数（参考`freewalk`函数，递归实现）

```c
// simulate freewalk to print vm
void printwalk(pagetable_t pagetable, int depth)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(pte & PTE_V){
      for (int j = 0;j < depth;j++){
        printf("..");
        if (j != depth - 1)
          printf(" ");
      }
      // this PTE points to a lower-level page table.
      // type cast
      uint64 child = PTE2PA(pte);
      printf("%d: pte %p pa %p\n", i, pte, child);

      if ((pte & (PTE_R|PTE_W|PTE_X)) == 0){
        printwalk((pagetable_t)child, depth + 1);
      }
    }
  }
}

void vmprint(pagetable_t pagetable){
  printf("page table %p\n", pagetable);
  printwalk(pagetable, 1);
}
```

PTE_*位实际上就是riscv的地址的某一特定位，如PTE_W就表示是否允许写，PTE_V表示是否有效。

![image-20220731105259736](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220731105259736.png)

最后在kernel/exec.c中加入pid=1时的判断，使其在初始时能打印

```c
 p->trapframe->sp = sp; // initial stack pointer
  proc_freepagetable(oldpagetable, oldsz);

  // add vmprint
  if(p->pid==1) vmprint(p->pagetable);

  return argc; // this ends up in a0, the first argument to main(argc, argv)
```

对实验中问题的解答：

![image-20220731131044825](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220731131044825.png)

![image-20220731131101427](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220731131101427.png)

![image-20220731131203381](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220731131203381.png)

可知，page0对应程序的代码段和数据段，page2则对应用户栈，中间的page1是guard page，因此也不能用于映射。

### A kernel page table per process

这个实验的要求是给每个进程配备一个内核页表。xv6原本只有一个内核页表，当处理user进程传入的指针时，只能先处理为物理地址（physical address）再处理。

首先在kernel/proc.h中，给proc增加内核页表

```c
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)

  pagetable_t kernelpgtbl;     // add kernel page table
};
```

修改kvminit函数。原先的函数逻辑是将全局变量kernel_pagetable（也就是xv6原先唯一的内核页表）初始化并map。这里将这个功能抽出来，先写一个kvm_init_one，表示初始化一个内核页表，再在kvminit里调用此函数，赋值给kernel_pagetable。

先在def.h中声明

```c
// add new function for kernel pg in each process
void            kvmmap_with_certain_page(pagetable_t pg, uint64 va, uint64 pa, uint64 sz, int perm);
void            kvmmap_with_certain_page(pagetable_t pg, uint64 va, uint64 pa, uint64 sz, int perm);
pagetable_t     kvm_init_one();

pte_t *         walk(pagetable_t pagetable, uint64 va, int alloc);
```

再在kernel/vm.c里完成（记得包含对应头文件）

```c
#include "spinlock.h"
#include "proc.h"

//..

/*
* init a new kernel virtual map
*/
pagetable_t kvm_init_one(){
  pagetable_t newpg = uvmcreate();
  // copy but forget to modify the first argument. so sad
  // uart registers
  kvmmap_with_certain_page(newpg, UART0, UART0, PGSIZE, PTE_R | PTE_W);

  // virtio mmio disk interface
  kvmmap_with_certain_page(newpg, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);

  // CLINT
  kvmmap_with_certain_page(newpg, CLINT, CLINT, 0x10000, PTE_R | PTE_W);

  // PLIC
  kvmmap_with_certain_page(newpg, PLIC, PLIC, 0x400000, PTE_R | PTE_W);

  // map kernel text executable and read-only.
  kvmmap_with_certain_page(newpg, KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);

  // map kernel data and the physical RAM we'll make use of.
  kvmmap_with_certain_page(newpg, (uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);

  // map the trampoline for trap entry/exit to
  // the highest virtual address in the kernel.
  kvmmap_with_certain_page(newpg, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);

  return newpg;
}

/*
 * create a direct-map page table for the kernel.
 */
void
kvminit()
{
  kernel_pagetable = kvm_init_one();
}

// simulate kvmmap
void kvmmap_with_certain_page(pagetable_t pg, uint64 va, uint64 pa, uint64 sz, int perm)
{
  if(mappages(pg, va, sz, pa, perm) != 0)
    panic("kvmmap with certain page");
}
```

在原先xv6中，所有内核栈均设置在`procinit`函数中初始化，为实现本实验功能，需将初始化移动到`allocproc`中，并调用刚才写的函数。

在kernel/proc.c中

```c
 initlock(&pid_lock, "nextpid");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");
  }
  kvminithart();
```

再在allocproc中增加

```c
  // add kernel page table
  p->kernelpgtbl = kvm_init_one();
  if (p->kernelpgtbl == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // init in allocproc

  // Allocate a page for the process's kernel stack.
  // Map it high in memory, followed by an invalid
  // guard page.
  char *pa = kalloc();
  if(pa == 0)
    panic("kalloc");
  uint64 va = KSTACK((int)(p - proc));
  kvmmap_with_certain_page(p->kernelpgtbl, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
  p->kstack = va;
```

接下来修改scheduler，使其在切换进程的同时切换内核页表（记得刷新TLB！），在该进程结束后（and没有运行进程的时候）记得换回kernel_pagetable

```c
 // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;

        // Switch to the independent kernel page table
        w_satp(MAKE_SATP(p->kernelpgtbl));
        // flush TLB
        sfence_vma();

        swtch(&c->context, &p->context);

        // Switch back to global kernel page table
        kvminithart();

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
```

接下来修改freeproc函数，使其能正确释放内核页表和内核栈

```c
  // free kernel stack in process
  // void *kstack_pa = (void*)kvmpa(p->kstack);
  // kfree(kstack_pa);
  // p->kstack = 0;

  if (p->kstack){
    pte_t* pte = walk(p->kernelpgtbl, p->kstack, 0);
    if (pte == 0)
      panic("freeproc: kstack");
    kfree((void*)PTE2PA(*pte));
  }
  p->kstack = 0;

  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);

  p->pagetable = 0;

  if (p->kernelpgtbl){
    kvm_free_pgtbl(p->kernelpgtbl);
  }
  p->kernelpgtbl = 0;

// free pg recursively
void kvm_free_pgtbl(pagetable_t pg){
  for (int i = 0; i < 512; i++){
    pte_t pte = pg[i];

    // copy wrong!!
    // if((pte & PTE_V) && (PTE_R|PTE_W|PTE_X) == 0){
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      uint64 child = PTE2PA(pte);
      kvm_free_pgtbl((pagetable_t)child);
      pg[i] = 0;
    } 
  }
  kfree((void*)pg);
}
```

最后记得修改kvmpa，此函数默认转换kernel_pagetable，改成正在运行的进程，即myproc()。

```c
  pte_t *pte;
  uint64 pa;

  pte = walk(myproc()->kernelpgtbl, va, 0);

  if(pte == 0)
    panic("kvmpa");
```

### simplify copyin/copyinstr

用`copyin[str]_new`代替原先的`copyin[str]`函数，同时修改user mapping，使用户地址能map到每个进程的内核页表中。

先在kernel/defs.h中声明

```c
// copy user pagetable to kernel page table
void            uvm2kvm(pagetable_t u, pagetable_t k, uint64 from, uint64 to);

// copy new
int             copyin_new(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len);
int             copyinstr_new(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max);
```

再在kernel/vm.c中实现`uvm2kvm`函数，将用户页表转换到内核页表。注意PLIC限制，同时将PTE_U设为0。

```c
void uvm2kvm(pagetable_t userpgtbl, pagetable_t kernelpgtbl, uint64 from, uint64 to)
{
  if (from > PLIC) // PLIC limit
    panic("uvm2kvm: from larger than PLIC");
  from = PGROUNDDOWN(from); // align
  for (uint64 i = from; i < to; i += PGSIZE)
  {
    pte_t *pte_user = walk(userpgtbl, i, 0); 
    pte_t *pte_kernel = walk(kernelpgtbl, i, 1); 
    if (pte_kernel == 0)
      panic("uvm2kvm: allocating kernel pagetable fails");
    *pte_kernel = *pte_user;
    *pte_kernel &= ~PTE_U;
  }
}
```

某些系统调用（fork(), exec(), sbrk()）会改变user mapping，需要对此也做相应更改。

fork

```c
  np->sz = p->sz;

  uvm2kvm(np->pagetable, np->kernelpgtbl, 0, np->sz);

  np->parent = p;
```

exec

```c
  // add vmprint
  if(p->pid==1) vmprint(p->pagetable);

  uvm2kvm(p->pagetable, p->kernelpgtbl, 0, p->sz);

  return argc; // this ends up in a0, the first argument to main(argc, argv)
```

（没给sbrk做更改，但貌似没报错（））

userinit

```c
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  uvm2kvm(p->pagetable, p->kernelpgtbl, 0, p->sz); // copy from user to kernel

  // prepare for the very first "return" from kernel to user.
  p->trapframe->epc = 0;      // user program counter
```

growproc

```c
  } else if(n < 0){
    sz = uvmdealloc(p->pagetable, sz, sz + n);
  }
  uvm2kvm(p->pagetable, p->kernelpgtbl, sz - n, sz);
  p->sz = sz;
  return 0;
}
```

最后替换copyin[str]

```c
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  return copyinstr_new(pagetable, dst, srcva, max);
}

int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  return copyin_new(pagetable, dst, srcva, len);
}
```

这里解释一下为什么`srcva + len < srcva`是必要的。例如，srcva=0x10，len为0xffff...ffff时，满足srcva >= p->sz，srcva+len >= p->sz，但srcva+len溢出，小于srcva，便可以检测到溢出

## 实验中遇到的问题及解决办法

- 使用`%p`在printf中输出64位的pte

- c不支持重载，一开始照着cpp的写法写成重载了

- 在调试的时候学会了用gdb调试

  - 先在一个命令行中`make qemu`
  - 再在另外一个命令行中`riscv64-unknown-elf-gdb kernel/kernel`，然后`target remote localhost:26000`
  - 接下来就是gdb的常规操作，如b，s，n等
    - `b xxx`添加断点
    - `c` continue
    - `s` step，进入函数
    - `n`next，但不进入函数
    - `d`删除所有断点
    - `

- usertests 在 reparent2 的时候出现了 panic: kvmmap，最后发现是释放页表的时候出现了错误
  ```c
  void kvm_free_pgtbl(pagetable_t pg){
    for (int i = 0; i < 512; i++){
      pte_t pte = pg[i];
      // copy wrong!!
  	// if((pte & PTE_V) && (PTE_R|PTE_W|PTE_X) == 0){
  	if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
  ```

  没有正确释放内存，导致大量内存泄漏，kvmmap 分配页表项所需内存失败。在这里卡了很久，最后发现是这里写错了，很难过。

## 实验心得

![image-20220731110821337](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220731110821337.png)