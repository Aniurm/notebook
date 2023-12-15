# BUFFER POOL

## Overview

The buffer pool is responsible for **moving physical pages back and forth from main memory to disk**. It allows a DBMS to support databases that are larger than the amount of memory available to the system. The buffer pool's operations are **transparent to other parts** in the system. （这里的transparent应该理解为“未察觉的，未认识到的”）

**Your implementation will need to be thread-safe.**

## Task #1 - LRU-K Replacement Policy

借助GPT，我才勉强理解LRU-K。LRU-K算法是一种页面置换算法，用于计算机缓存中决定哪个页面应该被替换。在这个算法中，"K"代表一个数字，意味着算法会考虑到页面被访问的最后K次记录。**Backward k-distance是从现在开始往前数，第k次访问此页面点时间与现在的时间差。**

当一个页面的访问次数不足K次时，它的向后k距离就被认为是无限大（+inf）。这样做是因为算法至少需要K次访问记录来计算一个页面的向后k距离，不足K次就意味着没有足够的信息来决定它的重要程度。

当算法需要替换一个页面时，它会选择向后k距离最大的页面。如果有多个页面的向后k距离都是无限大（也就是说，这些页面的访问次数都不足K次），那么算法会选择最早被访问的页面（“最早被访问”指第一次被访问的时间最早）。

!!! note "LRU-K Replacement Policy"

    You will need to implement the following methods:

    * `Evict(frame_id_t* frame_id)` : Evict the frame with largest backward k-distance compared to all other evictable frames being tracked by the Replacer. 
    * `RecordAccess(frame_id_t frame_id)` : Record that given frame id is accessed at current timestamp. 
    * `Remove(frame_id_t frame_id)` : Clear all access history associated with a frame. 
    * `SetEvictable(frame_id_t frame_id, bool set_evictable)` : This method controls whether a frame is evictable or not. It also controls LRUKReplacer's size. 
    * `Size()` : This method returns the number of evictable frames that are currently in the LRUKReplacer.

我使用的数据结构：两个`std::list<frame_id_t>`，分别用于保存访问次数小于、大于K的frame_id。`std::unordered_map<frame_id_t, LRUKNode`，用于保存frame_id对应的LRUKNode，其中LRUKNode包含了`frame_id`对应的`history`, `is_evictable`, `k`. 从简单到复杂，实现几个Methods：

* `Size()`：返回`curr_size_`
* `SetEvictable`：修改`LRUKNode`的数据，维护两个`std::list<frame_id_t>`，修改`curr_size_`等信息。
* `Remove`：把`LRUKNode`中的history清空，维护两个`std::list<frame_id_t>`。
* `RecordAccess`：修改`LRUKNode`中的history，维护两个`std::list<frame_id_t>`。
* `Evict`：先删去访问次数小于K的list的第一个元素（FIFO），如果访问次数都大于K，那么遍历访问次数大于K的list，找到backward k-distance最大的frame_id，然后删除这个frame_id。时间复杂度$O(n)$。

好几个Method都需要根据`LRUKNode`的数据，调整两个`std::list<frame_id_t>`，所以可以把这个逻辑封装一下。

!!! failure "误区"
    看了下网上的一些实现，发现有些人也维护了两个`std::list<frame_id_t>`，但是他们对访问次数超过K的list使用了LRU算法。我认为这是不能等同于LRU-K的。举一个例子，假设K=2。现在对于frame 1, 访问时间是1, 9. 对于frame 2, 访问时间是4, 5. 现在的时间为10, 我需要Evict一个frame。如果按照LRU算法，那么frame 2会被替换，但是按照LRU-K算法，frame 1会被替换（因为frame 1的backward k-distance是10-1=9，frame 2的backward k-distance是10-4=6）所以这两个算法是不同的。我这边老老实实地$O(N)$遍历（曾经想用堆，但是维护堆的时间复杂度也差不多，实现起来还麻烦）

## Task #2 - Disk Scheduler

The disk scheduler can be used by other components to queue disk requests, represented by a `DiskRequest` struct. The disk scheduler will maintain a background worker thread which is responsible for processing scheduled requests.

The disk scheduler will utilize a shared queue to schedule and process the `DiskRequest`s. One thread will add 
requests to the queue, and the disk scheduler's background worker will process requests.

### Disk Manager

The Disk Manager class reads and writes the page data from and to the disk. Your disk scheduler will 
use `DiskManager::ReadPage()` and `DiskManager::WritePage()` when it is processing a read or write request.

!!! note "Task"
    * `Schedule(DiskRequest r)` : Schedules a request for the `DiskManager` to execute. The DiskRequest struct specifies whether the request is for a read/write, where the data should be written into/from, and the page ID for the operation. The `DiskRequest` also includes a `std::promise` whose value should be set to true once the request is processed.
    * `StartWorkerThread()` : Start method for the background worker thread which processes the scheduled requests. The worker thread is created in the DiskScheduler constructor and calls this method. This method is responsible for getting queued requests and dispatching them to the `DiskManager`. Remember to set the value on the `DiskRequest`'s callback to signal to the request issuer that the request has been completed. This should not return until the `DiskScheduler`'s destructor is called.

乍一看很高级，实际上很简单。`Schedule`方法就是把`DiskRequest`放到`std::queue<DiskRequest>`中，`StartWorkerThread`运行在
background worker thread中，死循环从`std::queue<DiskRequest>`中取出`DiskRequest`，然后调用`DiskManager::ReadPage()`和`DiskManager::WritePage()`，最后设置`DiskRequest`的`std::promise`的值。直到取出`std::nullopt`才退出。

## Task #3 - Buffer Pool Manager

**The `BufferPoolManager` is responsible for fetching database pages from disk with the `DiskScheduler` and storing them in memory. The `BufferPoolManager` can also schedule writes of dirty pages out to disk** when it is either explicitly instructed to do so or when it needs to evict a page to make space for a new page.

All in-memory pages in the system are represented by `Page` objects. It is important to understand that **`Page` objects are just containers for memory in the buffer pool and thus are not specific to a unique page.** The `Page` object's identifier (`page_id`) keeps track of what physical page it contains.
Each `Page` object also maintains a counter for the number of threads that have "pinned" that page.

!!! note "Task"
    * `FetchPage(page_id_t page_id)`
    * `UnpinPage(page_id_t page_id, bool is_dirty)`, the `is_dirty` parameter keeps track of whether a page was modified while it was pinned.
    * `FlushPage(page_id_t page_id)`: flush a page regardless of its pin status.
    * `NewPage(page_id_t* page_id)`: should call `AllocatePage`
    * `DeletePage(page_id_t page_id)`: should call `DeallocatePage`