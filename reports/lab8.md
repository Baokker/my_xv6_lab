# Lab 8 Locks

## 实验目的

## 实验步骤

首先在kernel/kalloc.c中，为每个CPU分配kmem

```c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU]; // each cpu has one 
```

修改kinit

```c
void
kinit()
{
  char lockname[10];
  for (int i = 0; i < NCPU; i++){
    initlock(&kmem[i].lock, "kmem");
  }
  freerange(end, (void*)PHYSTOP);
}
```

修改kfree。记得查看cpuid前先关中断。

```c
  r = (struct run*)pa;

  push_off(); // need tu interrupt before get cpuid()
  int id = cpuid();
  acquire(&kmem[id].lock);
  r->next = kmem[id].freelist;
  kmem[id].freelist = r;
  release(&kmem[id].lock);
  pop_off();
}
```

修改kalloc，偷取。记得查看cpuid前先关中断。

```c
{
  struct run *r;

  push_off();
  int id = cpuid();
  acquire(&kmem[id].lock);
  r = kmem[id].freelist;
  if(r)
    kmem[id].freelist = r->next;
  else {
    int new_id;
    for (new_id = 0; new_id < NCPU; ++new_id){
      if (new_id == id)
        continue;
      acquire(&kmem[new_id].lock);
      r = kmem[new_id].freelist;
      if (r){
        kmem[new_id].freelist = r->next;
        release(&kmem[new_id].lock);
        break;
      }
      release(&kmem[new_id].lock);
    }
  }
  release(&kmem[id].lock);
  pop_off();

  if(r)
```

### Buffer cache

先定义buckets数量（在kernel/bio.c），并添加到bcache中。同时实现一个简单的hash

```c
#define NBUCKETS 13

struct {
  struct spinlock lock[NBUCKETS];
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head[NBUCKETS];
} bcache;

int hash(uint blockno){
  return blockno % NBUCKETS;
}
```

记得在def.h中声明hash函数

```c
void            bpin(struct buf*);
void            bunpin(struct buf*);

int             hash(uint id);
```

接下来修改bcache相关函数

binit

```c
void
binit(void)
{
  struct buf *b;

  for (int i = 0; i < NBUCKETS; i++){
    initlock(&bcache.lock[i], "bcache"); 
  }

  // Create linked list of buffers
  for (int i = 0; i < NBUCKETS; i++){
    bcache.head[i].prev = &bcache.head[i];
    bcache.head[i].next = &bcache.head[i];
  }
  for(b = bcache.buf; b < bcache.buf+NBUF; b++){
    b->next = bcache.head[0].next;
    b->prev = &bcache.head[0];
    initsleeplock(&b->lock, "buffer");
    bcache.head[0].next->prev = b;
    bcache.head[0].next = b;
  }
}
```

bget

```c
{
  struct buf *b;

  int id = hash(blockno);

  acquire(&bcache.lock[id]);

  // Is the block already cached?
  for(b = bcache.head[id].next; b != &bcache.head[id]; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock[id]);
      acquiresleep(&b->lock);
      return b;
    }
  }
    
    // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  int i = id;
  while (1){
    i = (i + 1) % NBUCKETS;
    if (i == id) continue;
    acquire(&bcache.lock[i]);
    for(b = bcache.head[i].prev; b != &bcache.head[i]; b = b->prev){
      if(b->refcnt == 0) {
        b->dev = dev;
        b->blockno = blockno;
        b->valid = 0;
        b->refcnt = 1;

        // disconnect
        b->prev->next = b->next;
        b->next->prev = b->prev;

        release(&bcache.lock[i]);

        b->prev = &bcache.head[id];
        b->next = bcache.head[id].next;
        b->next->prev = b;
        b->prev->next = b;
        release(&bcache.lock[id]);
        acquiresleep(&b->lock);
        return b;
      }
    }
    release(&bcache.lock[i]);
  }
  panic("bget: no buffers");
}
```

链表就是在这里派上用场的...

brelse

```c
  if(!holdingsleep(&b->lock))
    panic("brelse");

  int id = hash(b->blockno);

  releasesleep(&b->lock);

  acquire(&bcache.lock[id]);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head[id].next;
    b->prev = &bcache.head[id];
    bcache.head[id].next->prev = b;
    bcache.head[id].next = b;
  }

  release(&bcache.lock[id]);
}
```

bpin和bunpin

```c
void
bpin(struct buf *b) {
  int id = hash(b->blockno);
  acquire(&bcache.lock[id]);
  b->refcnt++;
  release(&bcache.lock[id]);
}

void
bunpin(struct buf *b) {
  int id = hash(b->blockno);
  acquire(&bcache.lock[id]);
  b->refcnt--;
  release(&bcache.lock[id]);
}
```

## 实验中遇到的问题及解决办法

## 实验心得

![image-20220731164001211](D:\GitHub_Desktop\my_xv6_lab\reports\img\image-20220731164001211.png)