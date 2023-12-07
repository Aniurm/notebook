# Lab: file system

In this lab you will add large files and symbolic links to the xv6 file system.

## Large files

!!! note "Task"
    * Modify `bmap()` so that it implements a doubly-indirect block, in addition to direct blocks and a singly-indirect block.
    * The first 11 elements of `ip->addrs[]` should be direct blocks; the 12th should be a singly-indirect block, the 13th should be your new doubly-indirect block.

首先，修改`inode`(disk and memory)的`addrs`，加入double indirect block。

```c title="fs.h"
#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
#define NDOUBLEINDIRECT (NINDIRECT * NINDIRECT)
#define INDEX_DOUBLEINDIRECT (NDIRECT + 1)
#define MAXFILE (NDIRECT + NINDIRECT + NDOUBLEINDIRECT)

// On-disk inode structure
struct dinode {
  /* other stuff... */

  uint addrs[NDIRECT+2];   // Data block addresses
};
```

```c title="file.h"
// in-memory copy of an inode
struct inode {
  /* other stuff... */

  uint addrs[NDIRECT+2];
};
```

接下来修改`bmap()`，实现double indirect block的分配和读取。

```c title="fs.c"
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){
    // Load direct block, allocating if necessary...
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary...
  }
  bn -= NINDIRECT;

  if (bn < NDOUBLEINDIRECT) {
    // Load double indirect block, allocating if necessary.
    if((addr = ip->addrs[INDEX_DOUBLEINDIRECT]) == 0)
      ip->addrs[INDEX_DOUBLEINDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    uint double_indirect_block = a[bn / NINDIRECT];
    if (double_indirect_block == 0) {
      a[bn / NINDIRECT] = double_indirect_block = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);

    // Load indirect block, allocating if necessary.
    bp = bread(ip->dev, double_indirect_block);
    a = (uint*)bp->data;
    if((addr = a[bn % NINDIRECT]) == 0){
      a[bn % NINDIRECT] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```

接着修改`itrunc()`，实现double indirect block的释放。

```c title="fs.c"
void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp;
  uint *a;

  for(i = 0; i < NDIRECT; i++){
    // Free direct blocks...
  }

  if(ip->addrs[NDIRECT]){
    // Free indirect blocks...
  }

  // Free double indirect blocks
  if (ip->addrs[INDEX_DOUBLEINDIRECT]) {
    bp = bread(ip->dev, ip->addrs[INDEX_DOUBLEINDIRECT]);
    a = (uint*)bp->data;
    for (i = 0; i < NINDIRECT; i++) {
      if (a[i]) {
        struct buf *bp2 = bread(ip->dev, a[i]);
        uint *a2 = (uint*)bp2->data;
        for (j = 0; j < NINDIRECT; j++) {
          if (a2[j]) {
            bfree(ip->dev, a2[j]);
          }
        }
        brelse(bp2);
        bfree(ip->dev, a[i]);
      }
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[INDEX_DOUBLEINDIRECT]);
    ip->addrs[INDEX_DOUBLEINDIRECT] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```

测试：在xv6中运行`bigfile`和`usertests`，通过✅。

## Symbolic links

!!! info

    **Symbolic links (or soft links) refer to a linked file by pathname**; when a symbolic link is opened, the kernel follows the link to the referred file. Symbolic links resembles hard links, but hard links are restricted to pointing to file on the same disk, while symbolic links can cross disk devices.

!!! note "Task"

    * Implement the `symlink(char *target, char *path)` system call, which creates a new symbolic link at path that refers to file named by target.

首先，更新`user/usys.pl, user/user.h, kernel/sysfile.c, Makefile`等文件，让xv6能够编译并运行`symlinktest`，这里不赘述。
只需要注意`O_NOFOLLOW`不要与其它flag冲突即可。

接下来，implement the `symlink(target, path)` system call to create a new symbolic link at path that refers to target. 细节：

* Note that target does not need to exist for the system call to succeed.
* `symlink` should return an integer representing success (0) or failure (-1).

```c title="sysfile.c"
uint64
sys_symlink(void)
{
  char target[MAXPATH], path[MAXPATH];
  if (argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0)
    return -1;

  begin_op();
  struct inode *ip = create(path, T_SYMLINK, 0, 0);
  if (ip == 0) {
    end_op();
    return -1;
  }

  if (writei(ip, 0, (uint64)target, 0, strlen(target)) != strlen(target)) {
    end_op();
    return -1;
  }
  iunlockput(ip);
  end_op();

  return 0;
}
```

在path创建一个inode，然后将target写入inode的data block中即可。

因为上文提到的细节，所以不需要检查target是否存在。

最后，modify the `open` system call to handle the case where the path refers to a symbolic link.

```c title="sysfile.c"
uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  if((n = argstr(0, path, MAXPATH)) < 0 || argint(1, &omode) < 0)
    return -1;

  begin_op();

  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op();
      return -1;
    }
  } else {
    if((ip = namei(path)) == 0){
      end_op();
      return -1;
    }
    ilock(ip);
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op();
      return -1;
    }
    if (ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
      if ((ip = followsymlink(ip, 0)) == 0) {
        end_op();
        return -1;
      }
    }
  }

  // other stuff...
}
```

只需要修改`omode & O_CREATE`为`false`，也就是要搜索path对应的inode的情况即可。最终目的是把symbol link的inode替换为target对应的inode。这样后续的代码就可以正常运行，不需要关心symbol link的存在。

`if`中的判断条件：

* `ip->type == T_SYMLINK`: path对应的inode是symbol link
* `!(omode & O_NOFOLLOW)`: open的调用者希望follow symbol link，得到target对应的inode

满足条件后，调用`followsymlink()`：

```c title="sysfile.c"
struct inode*
followsymlink(struct inode *ip, uint depth)
{
  // prevent symlink circle
  if (depth > 10) {
    iunlockput(ip);
    return 0;
  }

  char buf[MAXPATH];
  int n;
  struct inode *next;

  if (ip->type != T_SYMLINK)
    return ip;

  if ((n = readi(ip, 0, (uint64)buf, 0, MAXPATH)) < 0) {
    iunlockput(ip);
    return 0;
  }

  if ((next = namei(buf)) == 0) {
    iunlockput(ip);
    return 0;
  }

  iunlockput(ip);
  ilock(next);

  return followsymlink(next, depth + 1);
}
```

该函数递归直到找到target对应的inode（因为有可能出现symbol link chain），然后返回。为了防止symbol link circle，设置了一个`depth`的上限。

此外，需要非常注意lock。为了契合`sys_open`，`followsymlink`返回的inode是locked的，但是遍历symbol link chain的过程中需要将中间的inode unlock，否则有死锁的风险。各种edge case提前`return`的时候，到底应该怎么处理锁，需要仔细考虑。我个人花费了很多时间在锁的问题上。

最后，运行`make grade`，圆满结束。

```
== Test running bigfile == 
$ make qemu-gdb
running bigfile: OK (108.0s) 
== Test running symlinktest == 
$ make qemu-gdb
(0.8s) 
== Test   symlinktest: symlinks == 
  symlinktest: symlinks: OK 
== Test   symlinktest: concurrent symlinks == 
  symlinktest: concurrent symlinks: OK 
== Test usertests == 
$ make qemu-gdb
usertests: OK (176.4s) 
== Test time == 
time: OK
Score: 100/100
```


