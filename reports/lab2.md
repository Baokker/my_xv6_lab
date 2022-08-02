# Lab 2 system calls

## 实验目的

这部分的主要目的在于实现一些系统调用，而由于系统调用必须处于内核态中，故在实验过程中也对xv6的内核进一步了解

## 实验步骤

### trace

1. 将`$U/_trace`添加到UPROGS中

   ```makefile
   UPROGS=\
    	$U/_grind\
    	$U/_wc\
    	$U/_zombie\
   +	$U/_trace\
   ```

   

2. 将`trace`加入到系统调用中（后续所有实验都采取如是方式）

   ```c
   --- a/user/user.h
   +++ b/user/user.h
   @@ -23,6 +23,7 @@ int getpid(void);
    char* sbrk(int);
    int sleep(int);
    int uptime(void);
   +int trace(int mask); // add trace
   
   --- a/user/usys.pl
   +++ b/user/usys.pl
   @@ -36,3 +36,4 @@ entry("getpid");
    entry("sbrk");
    entry("sleep");
    entry("uptime");
   +entry("trace")
   
   --- a/kernel/syscall.h
   +++ b/kernel/syscall.h
   @@ -20,3 +20,4 @@
    #define SYS_link   19
    #define SYS_mkdir  20
    #define SYS_close  21
   +#define SYS_trace  22
   ```

   

3. 测试原先的`trace`函数，发现调用失败，需要对内核进行修改，实现内核系统调用的功能

   ```bash
   make qemu
   trace
   ```

   ![2-1](D:\GitHub_Desktop\my_xv6_lab\reports\img\2-1.png)

4. 包括

   ```c
   # 在proc.h中，为proc增加一个mask变量
   --- a/kernel/proc.h
   +++ b/kernel/proc.h
   @@ -103,4 +103,6 @@ struct proc {
      struct file *ofile[NOFILE];  // Open files
      struct inode *cwd;           // Current directory
      char name[16];               // Process name (debugging)
   +
   +  int mask;
    };
    
   # 在sysproc中增加一个sys_trace的函数
   --- a/kernel/sysproc.c
   +++ b/kernel/sysproc.c
   @@ -95,3 +95,8 @@ sys_uptime(void)
      release(&tickslock);
      return xticks;
    }
   
   # 获取系统调用的参数
   +uint64 sys_trace(void){
   +  argint(0,&(myproc()->mask));
   +  return 0;
   +}
   
   # 给fork做修改，使其在copy时将mask传递
   --- a/kernel/proc.c
   +++ b/kernel/proc.c
   @@ -268,6 +268,10 @@ fork(void)
      }
    
      // Copy user memory from parent to child.
   +
   +  // add mask
   +  np->mask = p->mask;
   +
      if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
        freeproc(np);
   ```

   

5. 修改syscall.c，使其打印trace输出
   注意追踪时需要用`&`符号判断，使得题目中`trace 2147483647 grep hello README`能够追踪所有的系统调用。
   
   ```c
   # 首先增加syscall外部声明
   --- a/kernel/syscall.c
   +++ b/kernel/syscall.c
   @@ -104,6 +104,7 @@ extern uint64 sys_unlink(void);
    extern uint64 sys_wait(void);
    extern uint64 sys_write(void);
    extern uint64 sys_uptime(void);
   +extern uint64 sys_trace(void);
   
   # 再是增加一个syscallname数组，以便输出调用函数名
   +// extra array for the name of syscall
   +static char *syscallname[] = {
   +[SYS_fork]    "fork",
   +[SYS_exit]    "exit",
   +[SYS_wait]    "wait",
   ...省略
   +[SYS_mkdir]   "mkdir",
   +[SYS_close]   "close",
   +[SYS_trace]   "trace",
    };
   
   # 最后修改syscall函数，打印输出
      num = p->trapframe->a7;
      if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        p->trapframe->a0 = syscalls[num]();
   +
   +    // if fetch..
   +    // use & so that every syscall that has lower bit
   +    // will log
   +    if (p->mask & (1 << num)){ 
   +      printf("%d: syscall %s -> %d\n", p->pid, syscallname[num], p->trapframe->a0);
   +    }
      } else {
        printf("%d %s: unknown sys call %d\n",
                p->pid, p->name, num);
   ```

  ### Sysinfo

1. 在UPROG中增加sysinfotest

   ```bash
   --- a/Makefile
   +++ b/Makefile
   @@ -150,7 +150,7 @@ UPROGS=\
    	$U/_wc\
    	$U/_zombie\
    	$U/_trace\
   +	$U/_sysinfotest
   ```
   
2. 在user.h中提前声明sysinfo这个struct

   ```c
   --- a/user/user.h
   +++ b/user/user.h
   @@ -25,6 +25,10 @@ int sleep(int);
    int uptime(void);
    int trace(int mask); // add trace
    
   +// add sysinfo
   +struct sysinfo;
   +int sysinfo(struct sysinfo *);
   +
   ```

3. 接下来实现这个系统调用

   ```
   # 增加系统调用的步骤同trace大同小异
   --- a/user/usys.pl
   +++ b/user/usys.pl
   @@ -36,4 +36,6 @@ entry("getpid");
    entry("sbrk");
    entry("sleep");
    entry("uptime");
   -entry("trace")
   +entry("trace");
   +entry("sysinfo");
   
   --- a/kernel/syscall.c
   +++ b/kernel/syscall.c
   @@ -105,6 +105,7 @@ extern uint64 sys_wait(void);
    extern uint64 sys_write(void);
    extern uint64 sys_uptime(void);
    extern uint64 sys_trace(void);
   +extern uint64 sys_sysinfo(void);
   ...
    // extra array for the name of syscall
   @@ -155,6 +157,7 @@ static char *syscallname[] = {
    [SYS_mkdir]   "mkdir",
    [SYS_close]   "close",
    [SYS_trace]   "trace",
   +[SYS_sysinfo] "sysinfo",
    };
    
    --- a/kernel/syscall.h
   +++ b/kernel/syscall.h
   @@ -21,3 +21,4 @@
    #define SYS_mkdir  20
    #define SYS_close  21
    #define SYS_trace  22
   +#define SYS_sysinfo  23
	```
4. 提前在def.h中声明所需函数
    ```c
       --- a/kernel/defs.h
       +++ b/kernel/defs.h
        void*           kalloc(void);
        void            kfree(void *);
        void            kinit(void);
       +// add freememory
       +void            freememory(uint64* result);
       ...
        int             either_copyout(int user_dst, uint64 dst, void *src, uint64 len);
        int             either_copyin(void *dst, int user_src, uint64 src, uint64 len);
        void            procdump(void);
       +// add procnum
       +void            procnum(uint64* result);
    ```

5. 遍历链表以获取所有的空闲内存
   
    ```c
       --- a/kernel/kalloc.c
       +++ b/kernel/kalloc.c
       @@ -39,6 +39,17 @@ freerange(void *pa_start, void *pa_end)
            kfree(p);
        }
    
       +// add function to get free memory
       +void freememory(uint64* result){
       +  *result = 0;
       +  struct run *cursor = kmem.freelist;
       +  acquire(&kmem.lock); // lock it
       +  while(cursor){
       +    cursor=cursor->next;
       +    (*result) += PGSIZE;
       +  }
       +  release(&kmem.lock);
       +}
    ```
6. 参考scheduler，获取活动进程数
   
   
    ```c
    --- a/kernel/proc.c
    +++ b/kernel/proc.c
    @@ -494,6 +494,16 @@ scheduler(void)
    }
    }
    
    +void procnum(uint64* result){
    +  *result = 0;
    +  // must use 'struct' in c
    +  for (struct proc *p = proc;p < &proc[NPROC];p++){
    +    if (p->state != UNUSED){
    +      (*result)++;
    +    }
    +  }
    +}
    +
    ```
7. 实现sys_info
    ```c
    # 因为需要用到myproc，所以包含proc相关头文件
    --- a/kernel/sysproc.c
    +++ b/kernel/sysproc.c
    @@ -7,6 +7,9 @@
    #include "spinlock.h"
    #include "proc.h"
    
    +// sysinfo struct
    +#include "sysinfo.h"
    
    # ..
    +
    +uint64 sys_sysinfo(void){
    +  // destination
    +  uint64 addr;
    +  if (argaddr(0, &addr) < 0){
    +    return -1;
    +  }
    +
    +  struct sysinfo info;
    +  freememory(&info.freemem);
    +  procnum(&info.nproc);
    +
    +
    +  // copyout from kernel to user
    +  if (copyout(myproc()->pagetable, addr, (char *)&info, sizeof info) < 0)
    +    return -1;
    +
    +  return 0;
    }
    ```

## 实验中遇到的问题及解决办法

- 忘记在def.h中添加声明
- entry里忘加了分号，实属粗心

## 实验心得

![2-2](img\2-2.png)