# locks

!!! danger "Lock contention"

    **A common symptom of poor parallelism on multi-core machines is high lock contention.** Improving parallelism often involves changing both data structures and locking strategies in order to reduce contention.

来自网上老哥的博客，写得很好：

!!! tip ""

    锁竞争优化一般有几个思路：

    * 只在必须共享的时候共享（对应为将资源从 CPU 共享拆分为每个 CPU 独立）
    * 必须共享时，尽量减少在关键区中停留的时间（对应“大锁化小锁”，降低锁的粒度）


## Memory allocator

The root cause of lock contention in kalloctest is that `kalloc()` has a single free list, protected by a single lock.

!!! note "Task"

    * Your job is to implement per-CPU freelists, and stealing when a CPU's free list is empty.
    * You must give all of your locks names that start with "kmem".

首先修改`kmem`，让每一个CPU都有自己的free list. 初始化时分配名字。

```c
struct kmem {
  struct spinlock lock;
  struct run *freelist;
  char lockname[8];
};

struct kmem kmems[NCPU];

void
kinit()
{
  for (int i = 0; i < NCPU; i++) {
    snprintf(kmems[i].lockname, sizeof(kmems[i].lockname), "kmem_%d", i);
    initlock(&kmems[i].lock, kmems[i].lockname);
  }
  freerange(end, (void*)PHYSTOP);
}
```

以下操作需要注意：在获取CPU ID时必须先关中断。

```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  push_off();
  struct kmem *kmem_ptr = &kmems[cpuid()];
  pop_off();
  acquire(&kmem_ptr->lock);
  r->next = kmem_ptr->freelist;
  kmem_ptr->freelist = r;
  release(&kmem_ptr->lock);
}
```

`freerange`使用了上面的`kfree`，这会导致一开始所有的free memory都在一个CPU（即调用`freerange`的CPU）的freelist中。

```c
void *
kalloc(void)
{
  struct run *r;

  push_off();
  int id = cpuid();
  struct kmem *kmem_ptr = &kmems[id];
  pop_off();
  acquire(&kmem_ptr->lock);

  // no free memory in current CPU's free list
  // steal 64 pages from other CPUs
  if (kmem_ptr->freelist == 0) {
    int steal_left = 64;
    for (int i = 0; i < NCPU; i++) {
      if (i == id) continue;
      acquire(&kmems[i].lock);
      while (kmems[i].freelist && steal_left > 0) {
        r = kmems[i].freelist;
        kmems[i].freelist = r->next;
        r->next = kmem_ptr->freelist;
        kmem_ptr->freelist = r;
        steal_left--;
      }
      release(&kmems[i].lock);
      if (steal_left == 0) break;
    }
  }

  r = kmem_ptr->freelist;
  if(r)
    kmem_ptr->freelist = r->next;
  release(&kmem_ptr->lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

如果当前CPU的freelist为空，就从其他CPU的freelist中偷64个pages。偷pages的合理数量应该由实验得出，如果太小，偷pages的次数会很多，导致频繁的锁操作。

## Buffer cache

If multiple processes use the file system intensively, they will likely contend for `bcache.lock`, which protects the disk block cache in `kernel/bio.c`.

!!! note "Task"

    * Modify the block cache to reduce contention.
    * Modify `bget` and `brelse` so that concurrent lookups and releases for different blocks that are in the bcache are unlikely to conflict on locks.
    * You must maintain the invariant that at most one copy of each block is cached.

旧版本的`bcache`
  
```c
struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head;
} bcache;
```

使用一个`spinlock`保护整个`bcache`，这会导致大量的锁竞争。buffer以双向链表的形式组织，最近使用的buffer在链表头部，最久未使用的在链表尾部，实现了LRU。

为了减少锁竞争，我们可以引入一个hash table，用`dev`和`blockno`计算出hash值。

```c
#define NBUCKET 13
#define HASH(dev, blockno) (((uint)dev ^ (uint)blockno) % NBUCKET)

struct {
  struct buf buf[NBUF];

  struct buf bufmap[NBUCKET];
  struct spinlock bufmap_lock[NBUCKET];
  struct spinlock evict_lock;
} bcache;

void
binit(void)
{
  struct buf *b;

  initlock(&bcache.evict_lock, "evict_lock");

  for (int i = 0; i < NBUCKET; i++) {
    bcache.bufmap[i].next = 0; 
    initlock(&bcache.bufmap_lock[i], "bufmap_lock");
  }

  // Put all buffers into bufmap's 0th bucket.
  for (b = bcache.buf; b < bcache.buf + NBUF; b++) {
    initsleeplock(&b->lock, "buffer");
    b->last_use = 0;
    b->next = bcache.bufmap[0].next;
    bcache.bufmap[0].next = b;
  }
}
```

`bufmap`的每一个bucket都指向一个单向的buffer linked list，每一个bucket都有一个`spinlock`保护。这相当于把原本的整个linked list拆分为了多个小的linked list，每个小的linked list存储在一个bucket中，每个bucket都有一个`spinlock`保护，这样就减少了锁竞争。

在`buf`中增加了`last_use`字段，与`trap.c`中的`ticks`结合使用，用来记录最近一次使用的时间。删去`prev`，不再使用双向链表。

```c
struct buf {
  int valid;   // has data been read from disk?
  int disk;    // does disk "own" buf?
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  uint last_use;
  struct buf *next;
  uchar data[BSIZE];
};
```

本次Lab的重头戏来了：`bget`

!!! danger "有几个非常重要的细节，需要谨慎地用lock来处理"
    * maintain the invariant that at most one copy of each block is cached
    * 死锁问题（进程A持有bucket1的锁，evict遍历时acquire bucket2的锁，进程B持有bucket2到锁，evict遍历时acquire bucket1的锁）
    * 得到lru_buffer后也需要继续持有对应bucket的锁，否则其他进程可能会增加对此buffer的引用计数，导致原本可驱逐的lru_buffer无法被驱逐

如果想看更详细的分析，可以看[这里](https://blog.miigon.net/posts/s081-lab8-locks/)，我的实现也是参考这篇博客的，这个哥们写得真的很好。

这里直接贴我的实现：

```c
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  int key = HASH(dev, blockno);
  acquire(&bcache.bufmap_lock[key]);

  // Is the block already cached?
  for (b = bcache.bufmap[key].next; b != 0; b = b->next) {
    if (b->dev == dev && b->blockno == blockno) {
      b->refcnt++;
      release(&bcache.bufmap_lock[key]);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Avoid deadlock
  release(&bcache.bufmap_lock[key]);

  acquire(&bcache.evict_lock);

  // Check Again:
  // Is the block already cached?
  for (b = bcache.bufmap[key].next; b != 0; b = b->next) {
    if (b->dev == dev && b->blockno == blockno) {
      b->refcnt++;
      release(&bcache.evict_lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Find a buffer to evict.
  // Trick: record the pointer before lru_buf for easier deletion.
  struct buf *before_lru_buf = 0;
  int holding_bucket = -1;
  for (int i = 0; i < NBUCKET; i++) {
    int new_found = 0;
    acquire(&bcache.bufmap_lock[i]);
    for (b = &bcache.bufmap[i]; b->next != 0; b = b->next) {
      if (b->next->refcnt == 0) {
        if (before_lru_buf == 0 || b->next->last_use < before_lru_buf->last_use) {
          before_lru_buf = b;
          new_found = 1;
        }
      }
    }
    if (new_found) {
      if (holding_bucket != -1)
        release(&bcache.bufmap_lock[holding_bucket]);
      // keep holding the lock where new lru_buf is found.
      holding_bucket = i;
    } else {
      release(&bcache.bufmap_lock[i]);
    }
  }

  if (before_lru_buf == 0) {
    panic("bget: no buffers");
  }

  b = before_lru_buf->next;
  if (holding_bucket != key) {
    // Remove lru_buf from original bucket
    before_lru_buf->next = b->next;
    // Add lru_buf to the head of the bucket
    acquire(&bcache.bufmap_lock[key]);
    b->next = bcache.bufmap[key].next;
    bcache.bufmap[key].next = b;
    release(&bcache.bufmap_lock[key]);
  }
  // Update buf info
  b->blockno = blockno;
  b->dev = dev;
  b->valid = 0;
  b->refcnt = 1;
  b->last_use = ticks;
  acquiresleep(&b->lock);
  release(&bcache.bufmap_lock[holding_bucket]);
  release(&bcache.evict_lock);
  
  return b;
}
```

1. 首先，如果block已经在cache中，直接返回。
2. 如果block不在cache中，需要从其他bucket中驱逐一个block，这里需要遍历所有的bucket，找到一个可驱逐的block。
      1. 遍历前，`release(&bcache.bufmap_lock[key])`，避免刚才提到的死锁。
      2. `acquire(&bcache.evict_lock)`，保证只有一个进程在执行evict操作，只有一个进程能够修改buf在bucket中的位置。
      3. Check Again：从`release(&bcache.bufmap_lock[key])`到`acquire(&bcache.evict_lock)`期间，可能有其他进程完成了evict操作，所以需要再次检查block是否已经在cache中。
3. 遍历过程中，用`holding_bucket`记录当前找到的lru_buffer所在的bucket，保证在遍历过程中不会释放这个bucket的锁。如果找到了一个新的lru_buffer，就释放之前的bucket的锁，保持对新的bucket的锁。这确保了将要被evict的lru_buffer不会被其他进程增加引用计数。
4. 终于遍历完了，找到了一个lru_buffer，将其从原来的bucket中删除，加入到新的bucket的头部，更新buf的信息，处理相关的锁，返回。

## 总结

这次Lab的重点是锁竞争优化，涉及到锁的粒度、锁的数量、锁的获取顺序等等。这些都是非常细节的东西，需要仔细地分析，才能写出正确的代码。


!!! tip "摘抄一段："

    don't share if you don't have to

    start with a few coarse-grained locks

    instrument your code -- which locks are preventing parallelism?

    use fine-grained locks only as needed for parallel performance

    use an automated race detector