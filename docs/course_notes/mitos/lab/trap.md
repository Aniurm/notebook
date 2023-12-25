# traps

## Backtrace

<aside> 📌 Backtrace: a list of the function calls on the stack above the point at which the error occurred.

</aside>

目的：当 kernel panic 的时候能够打印出 backtrace 信息

对于这个实验，很重要的一张图：

![Untitled](img/Untitled.png)

stack 从高地址向低地址增长，在这个图是从上到下的方向。而 backtrace 是从最 low 的 stack 开始（也就是最新的、刚刚发生 panic 的 stack），从下往上遍历所有 stack。

大概思路有了之后，跟着 hints 把实验做完

**Add the prototype for backtrace to kernel/defs.h so that you can invoke backtrace.**

```c
// defs.h
void            backtrace(void);
```

这里告诉我们怎么提取`ra`(return address)和上一个栈的`fp`

**Xv6 allocates one page for each stack in the xv6 kernel at PAGE-aligned address. You can compute the top and bottom address of the stack page by using PGROUNDDOWN(fp) and PGROUNDUP(fp). These number are helpful for backtrace to terminate its loop.**

这里告诉我们应该在什么时候停止遍历。

---

```c
void
backtrace(void)
{
	// 获取当前的fp
  uint64 fp = r_fp();
	// 因为栈是从高地址向低地址增长的，所以应该使用PGROUNDUP获得stack底部
  uint64 stack_page_bottom = PGROUNDUP(fp);
  while (fp != stack_page_bottom) {
		// return address
    printf("%p\n", *(uint64 *)(fp - 8));
		// previous frame pointer
    fp = *(uint64 *)(fp - 16);
  }
}
```

## Alarm

<aside>
📌 periodically alerts a process as it uses CPU time

</aside>

- add a new `sigalarm(interval, handler)` system call
- If an application calls `sigalarm(n, fn)`, then after every `n` "ticks" of CPU time that the program consumes, the kernel should cause application function `fn` to be called.
- When `fn` returns, the application should resume where it left off.

### **invoke handler**

**Update `user/usys.pl` (which generates `user/usys.S`), `kernel/syscall.h`, and `kernel/syscall.c` to allow `alarmtest` to invoke the `sigalarm` and `sigreturn` system calls**

- **`user/usys.pl`：**Generate usys.S, the stubs for syscalls. 有了 stub，用户程序的系统调用请求就可以被转发到内核，以便执行相应的系统调用

**Your `sys_sigalarm()` should store the alarm interval and the pointer to the handler function in new fields in the `proc` structure.**

**You'll need to keep track of how many ticks have passed since the last call to a process's alarm handler; you'll need a new field in struct `proc` for this too. You can initialize `proc` fields in `allocproc()` in `proc.c`.**

**When a trap on the RISC-V returns to user space, what determines the instruction address at which user-space code resumes execution?**

这是一个非常重要的提示：当 RISC-V 从 kernel 返回 user space 的时候

> …returning to user space…setting `sepc` to the previously saved user program counter

`usertrap`中的代码：

```c
// save user program counter.
p->trapframe->epc = r_sepc();
```

`usertrapret`中的代码：

```c
// set S Exception Program Counter to the saved user pc.
w_sepc(p->trapframe->epc);
```

所以只要修改`p->trapframe->epc`，就可以控制 user space 代码 resumes execution 的位置

---

最开始的时候，用户需要调用 system call -- sigalarm 来配置、激活 alarm。

没啥好说的，配置`struct proc`中 alarm 相关的 fields 即可

```c title="sysproc.c, sys_sigalarm"
uint64
sys_sigalarm(void)
{
  // get ticks in a0 and handler function in a1
  int ticks;
  if (argint(0, &ticks) < 0)
    return -1;

  uint64 handler;
  if (argaddr(1, &handler) < 0)
    return -1;

  struct proc *p = myproc();

  if (ticks == 0 && handler == 0) {
    // disable alarm
    p->alarm_interval = p->alarm_handler_addr = p->alarm_ticks_left = 0;
    return 0;
  }

  p->alarm_interval = ticks;
  p->alarm_handler_addr = handler;
  p->alarm_ticks_left = ticks;
  return 0;
}
```

当用户调用`sigalarm`之后，仅仅是把`struct proc`里面的几个 fields 的值改了一下而已。

所以还需要在发生 timer interrupt 的时候，根据该进程的 alarm 信息，进行对应的处理

```c title="kernel/trap.c, usertrap"
  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2) {
    if (p->alarm_interval > 0) {
      if (--p->alarm_ticks_left == 0 && p->handler_executing == 0) {
        p->alarm_ticks_left = p->alarm_interval;

        // save original trapframe
        memmove(p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));

        p->trapframe->epc = p->alarm_handler_addr;
        p->handler_executing = 1;
      }
    }
    yield();
  }
```

- 检查是否开启 alarm`if (p->alarm_interval > 0)`如果开启：
  - `--p->alarm_ticks_left`减小倒计时。如果倒计时变成 0：
    - reset 倒计时
    - `p->trapframe->epc = p->alarm_handler_addr;`这样返回到 user space 时的 PC 就是 alarm handler 的地址

**后续要求：when the alarm handler is done, control returns to the instruction at which the user program was originally interrupted by the timer interrupt**

接下来根据 hints 继续解释代码

**user alarm handlers are required to call the `sigreturn` system call when they have finished.**

**Have `usertrap` save enough state in `struct proc` when the timer goes off that `sigreturn` can correctly return to the interrupted user code.**

上面`usertrap`的代码中，`memmove(p->alarm_trapframe, p->trapframe, sizeof(struct trapframe));`把进程原本的`p->trapframe`保存到一个新增的 field：`alarm_trapframe`，备份了之后，我们就可以对`p->trapframe`为所欲为了，因为`p->trapframe`会在`sigreturn`中被恢复。

**Prevent re-entrant calls to the handler----if a handler hasn't returned yet, the kernel shouldn't call it again.**

加一个标志位即可：如果进入了 alarm handler，就把`p->handler_executing`设为 1，handler 执行结束后调用的`sigreturn`会把`p->handler_executing`重新置 0。每次 timer interrupt 都需要检查`p->handler_executing`是否为 0.

### resume interrupted code

```c
uint64
sys_sigreturn(void)
{
  struct proc *p = myproc();
  memmove(p->trapframe, p->alarm_trapframe, sizeof(struct trapframe));
  p->handler_executing = 0;
  return 0;
}
```

恢复之前备份好的数据，把标志位设为 0 即可。

---

![test](img/trap-res.png)
