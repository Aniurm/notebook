# Lab: Copy-on-Write Fork for xv6

!!! question "The Problem"

    The `fork()` system call in xv6 copies all of the parent process's user-space memory into the child. If the parent is large, **copying can take a long time.** Worse, the work is often largely wasted; for example, a `fork()` followed by `exec()` in the child will cause the child to discard the copied memory, probably without ever using most of it. On the other hand, if both parent and child use a page, and one or both writes it, **a copy is truly needed.**

!!! success "The solution"

    **The goal of copy-on-write (COW) `fork()` is to defer allocating and copying physical memory pages for the child until the copies are actually needed, if ever.**

    COW `fork()` creates just a pagetable for the child, with PTEs for user memory pointing to the parent's physical pages. COW `fork()` marks all the user PTEs in both parent and child as not writable. When either process tries to write one of these COW pages, the CPU will force a **page fault**. The kernel page-fault handler detects this case, allocates a page of physical memory for the faulting process, copies the original page into the new page, and modifies the relevant PTE in the faulting process to refer to the new page, this time with the PTE marked writeable. When the page fault handler returns, the user process will be able to write its copy of the page.

    COW `fork()` makes freeing of the physical pages that implement user memory a little trickier. **A given physical page may be referred to by multiple processes' page tables, and should be freed only when the last reference disappears.**

## Implement copy-on write

**Modify `uvmcopy()` to map the parent's physical pages into the child, instead of allocating new pages. Clear `PTE_W` in the PTEs of both child and parent.**

```c title="vm.c, uvmcopy()"
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    // clear PTE_W in both old and new page tables
    *pte = *pte & ~PTE_W;
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    // map the parent's physical pages into the child
    if(mappages(new, i, PGSIZE, pa, flags) != 0)
      goto err;
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}
```

**Modify `usertrap()` to recognize page faults. When a page-fault occurs on a COW page, allocate a new page with `kalloc()`, copy the old page to the new page, and install the new page in the PTE with `PTE_W` set.**

参考 lazy，用函数`needcow`判断是否符合 COW 的条件。

```c title="vm.c, needcow"
int
needcow(struct proc *p, uint64 va)
{
  // Kill a process if it page-faults on a virtual memory address
  // higher than any allocated with sbrk().
  if (va >= p->sz)
    return 0;

  // Handle faults on the invalid page below the user stack.
  // should not touch guard page below stack
  if (PGROUNDDOWN(va) <= p->trapframe->sp)
    return 0;

  pte_t *pte = walk(p->pagetable, va, 0);
  if (pte == 0 || (*pte & PTE_V) == 0)
    return 0;

  return (*pte & PTE_COW) != 0;
}
```

**Ensure that each physical page is freed when the last PTE reference to it goes away** -- but not before. A good way to do this is to keep, for each physical page, **a "reference count" of the number of user page tables that refer to that page.** Set a page's reference count to one when `kalloc()` allocates it. Increment a page's reference count when `fork` causes a child to share the page, and decrement a page's count each time any process drops the page from its page table. `kfree()` should only place a page back on the free list if its reference count is zero. It's OK to to keep these counts in a fixed-size array of integers. You'll have to **work out a scheme for how to index the array and how to choose its size.** For example, you could index the array with the page's physical address divided by 4096, and give the array a number of elements equal to highest physical address of any page placed on the free list by `kinit()` in `kalloc.c`.
