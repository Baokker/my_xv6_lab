# Lab 4

## 实验目的

## 实验步骤

### RISC-V assembly 

1. 运行make fs.img
   ![image-20220715213128807](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220715213128807.png)

回答：

1. a0~a7储存函数参数，例如`printf`的13寄存在a2寄存器中

2. ```assembly
   void main(void) {
     1c:	1141                	addi	sp,sp,-16
     1e:	e406                	sd	ra,8(sp)
     20:	e022                	sd	s0,0(sp)
     22:	0800                	addi	s0,sp,16
     printf("%d %d\n", f(8)+1, 13);
     24:	4635                	li	a2,13
     26:	45b1                	li	a1,12 # key sentence
     28:	00000517          	auipc	a0,0x0
   ```

   可以看到，编译器直接计算出了结果12

3. ![image-20220731133518705](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220731133518705.png)
   printf的地址为638

4. ra=pc+4=0x34+4=0x38

5. 输出：HE110 World

6. y输出的是a2的内容
   ![image-20220731141133622](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220731141133622.png)

   ![image-20220731141110370](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220731141110370.png)

## backtrace

加入到def.h声明

```c
void            printf(char*, ...);
void            panic(char*) __attribute__((noreturn));
void            printfinit(void);
void            backtrace(void);
```

在kernel/riscv.h中加入如下函数，以便backtrace能返回当前页指针

```c
// add for backtrace
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```

在kernel/printf.c中实现backtrace

```c
void backtrace(void){
  printf("backtrace:\n");
  uint64 fp = r_fp();
  while (PGROUNDDOWN(fp) < PGROUNDUP(fp)){
    printf("%p\n", *(uint64*)(fp - 8));
    fp = *(uint64*)(fp - 16);
  }
}
```

最后加入到kernel/sysproc.c中的sys_sleep即可

```c
  int n;
  uint ticks0;

  backtrace();

  if(argint(0, &n) < 0)
    return -1;
  acquire(&tickslock);
```

### Alarm

首先是第一部分，即test0

修改makefile，加入alarmtest

```c
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_alarmtest\
```

再在user/user.h里加声明，hints里已经给出

```c
int sleep(int);
int uptime(void);

// add new alarm
int sigalarm(int ticks, void (*handler)());
int sigreturn(void);

// ulib.c
int stat(const char*, struct stat*);
char* strcpy(char*, const char*);
```

接下来这部分与Lab2添加系统调用大同小异

user/usys.pl增加

```perl
entry("sbrk");
entry("sleep");
entry("uptime");
entry("sigalarm");
entry("sigreturn");
```

kernel/syscall.h增加

```c
#define SYS_link   19
#define SYS_mkdir  20
#define SYS_close  21
#define SYS_sigalarm 22
#define SYS_sigreturn 23
```

kernel/syscall.c

```
extern uint64 sys_wait(void);
extern uint64 sys_write(void);
extern uint64 sys_uptime(void);
extern uint64 sys_sigalarm(void);
extern uint64 sys_sigreturn(void);

static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
...
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
[SYS_sigalarm] sys_sigalarm,
[SYS_sigreturn] sys_sigreturn,
};
```

接下来，需要让`sys_sigalarm`记录时钟间隔，为此，先在kernel/proc.h的proc中增添属性：

alarm_trapframe是为了test1/2中的临时保存寄存器。

```c
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)

  // add for alarm
  int alarm_interval;
  int alarm_ticks;
  uint64 alarm_handler;
  struct trapframe alarm_trapframe;
};
```

并在kernel/proc.c的allocproc中初始化这些属性

```c
  return 0;

found:
  p->pid = allocpid(); 

  // set alarm related parameters
  p->alarm_interval = 0;
  p->alarm_ticks = 0;
  p->alarm_handler = 0;

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
```

在kernel/trap.c的usertrap里处理interrupt

```c
  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2){
    if (p->alarm_interval){
      if (++p->alarm_ticks == p->alarm_interval){
        memmove(&(p->alarm_trapframe), p->trapframe, sizeof(*(p->trapframe)));
        p->trapframe->epc = p->alarm_handler;
      }
    }
    yield();
  }

  usertrapret();
}
```

最后在kernel/sysproc.c中完善相关函数

```c
uint64 sys_sigalarm(void){
  int interval;
  uint64 handler;

  if (argint(0, &interval) < 0)
    return -1;
  if (argaddr(1, &handler) < 0)
    return -1;

  myproc()->alarm_interval = interval;
  myproc()->alarm_handler = handler;
  return 0;
}

uint64 sys_sigreturn(void){
  memmove(myproc()->trapframe, &(myproc()->alarm_trapframe), sizeof(struct trapframe));
  myproc()->alarm_ticks = 0;
  return 0;
}
```

## 实验中遇到的问题及解决办法

## 实验心得

![image-20220731162859572](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220731162859572.png)