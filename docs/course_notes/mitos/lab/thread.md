# Multithreading

最简单的lab没有之一。

## Uthread: switching between threads

Uthread是一个用户态的线程库，由我们自己实现。

!!! note "Task"

    Your job is to come up with a plan to create threads and save/restore registers to switch between threads, and implement that plan.

### Thread creation

```c
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  t->context.ra = (uint64)func;
  t->context.sp = (uint64)(t->stack + STACK_SIZE);
}
```

> when `thread_schedule()` runs a given thread, the thread executes the function passed to `thread_create()`, on its own stack.

要做到这一点，需要把`ra`设置为`func`的地址，这样当scheduler切换到这个线程的时候，就会从`func`开始执行。

别忘了设置`sp`，让`sp`指向线程栈的顶部。

### Thread switching

仿照内核线程的切换，我们也需要保存和恢复context。

```c
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context context;       /* register context */
};
```

给`struct thread`添加`context`，用来保存寄存器的值。

这里可以观察到线程的stack只是`thread`结构体中的一个`char`数组，大小固定为`STACK_SIZE=8192 bytes`。

```c
extern void thread_switch(struct context*, struct context*);
```

`uthread_switch.S`可以直接照搬内核的`swtch.S`，都是一样的。

### Thread scheduling

```c
void 
thread_schedule(void)
{
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  /* ... */

  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    thread_switch(&t->context, &next_thread->context);
  } else
    next_thread = 0;
}
```

## Using threads

Explore parallel programming with UNIX `pthread` threading library using a hash table.

`notxv6/ph.c` contains a simple hash table that is correct if used from a single thread, but incorrect when used from multiple threads.

为什么多线程的情况下会有missing keys?

其实xv6 book中Lock的章节已经给出类似的例子，两个线程同时对一个链表执行插入node操作，会导致其中一个node丢失。结合代码画画图就能看出来。

!!! note "Task"

    Insert lock and unlock statements in `put` and `get` in `notxv6/ph.c` so that the number of keys missing is always 0 with multiple threads.

如果直接给put, get操作加锁，能得到正确的结果，但是性能反而比单线程的情况下差。因为每次只能有一个线程访问hash table，其他线程都在等待。这就跟单线程的情况下没有区别了。此外，锁操作（加锁、解锁、锁竞争）也会带来额外的开销，进一步降低性能。

因此，我们需要减少锁的粒度——每个bucket一个锁，让多个线程可以同时访问hash table。

```c
pthread_mutex_t locks[NBUCKET];
```

```c
static 
void put(int key, int value)
{
  int i = key % NBUCKET;
  pthread_mutex_lock(&locks[i]);

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&locks[i]);
}

static struct entry*
get(int key)
{
  int i = key % NBUCKET;
  pthread_mutex_lock(&locks[i]);

  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key) break;
  }
  pthread_mutex_unlock(&locks[i]);

  return e;
}
```

别忘了在`main`函数中初始化锁。
```c
int
main(int argc, char *argv[])
{
  pthread_t *tha;
  void *value;
  double t1, t0;

  for(int i = 0; i < NBUCKET; i++) {
    assert(pthread_mutex_init(&locks[i], NULL) == 0);
  }
  // ...
}
```

## Barrier

!!! note "Task"
    Implement a barrier: a point in an application at which all participating threads must wait until all other participating threads reach that point too.

先理解代码：

```c
static int nthread = 1;
static int round = 0;

struct barrier {
  pthread_mutex_t barrier_mutex;
  pthread_cond_t barrier_cond;
  int nthread;      // Number of threads that have reached this round of the barrier
  int round;     // Barrier round
} bstate;
```

可以看到全局变量`nthread`记录了程序有多少个线程。`bstate`记录了锁、条件变量、当前达到barrier的线程数、当前barrier的round。

```c
static void *
thread(void *xa)
{
  long n = (long) xa;
  long delay;
  int i;

  for (i = 0; i < 20000; i++) {
    int t = bstate.round;
    assert (i == t);
    barrier();
    usleep(random() % 100);
  }

  return 0;
}
```

每个线程都会执行20000次循环，每次循环中，检查当前的round是否等于循环次数。后调用`barrier()`，等待其他线程到达barrier。如果barrier正确同步了所有线程，就不会出现`i != t`的情况。

下面看看`barrier()`的实现：

```c
static void 
barrier()
{
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread++;
  if (bstate.nthread == nthread) {
    bstate.round++;
    bstate.nthread = 0;
    pthread_cond_broadcast(&bstate.barrier_cond);
  } else {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

首先，每个线程都会调用`barrier()`，因此需要加锁。然后，每个线程都会把`bstate.nthread`加1，表示当前线程已经到达barrier。如果当前线程是最后一个到达barrier的线程，那么就需要更新`bstate.round`，并且唤醒其他线程。否则，当前线程就需要等待其他线程到达barrier。