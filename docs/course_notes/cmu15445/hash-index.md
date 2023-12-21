# EXTENDIBLE HASH INDEX

## Overview

In this programming project you will implement **disk-backed hash index** in your database system. You will be using a variant of [extendible hashing](https://en.wikipedia.org/wiki/Extendible_hashing) as the hashing scheme.
We added a non-resizable header page on top of the directory pages so that the hash table can hold more values and potentially achieve better multi-thread performance.

<figure markdown>
![img](img/extendible-htable-structure.svg){width="400"}
</figure>

Your implementation should support thread-safe search, insertion, and deletion (including growing/shrinking the directory and splitting/merging buckets).

## Task #1 - Read/Write Page Guards

In the Buffer Pool Manager, `FetchPage` and `NewPage` functions return pointers to pages that are already pinned. To indicate that the page is no longer needed in memory, the programmer has to manually call `UnpinPage`. If the programmer forgets to call `UnpinPage`, the page will never be evicted out of the buffer pool. Not only the performance takes a hit, the bug is also difficult to be detected.

!!! note "Task"
    * You will  implement `BasicPageGuard` which store the pointers to `BufferPoolManager` and `Page` objects. A page guard ensures that `UnpinPage` is called on the corresponding `Page` object as soon as it goes out of scope.
    * You will also implement `ReadPageGuard` and `WritePageGuard` which automatically unlatch the pages as soon as they go out of scope.

## Task #2 - Extendible Hash Table Pages

### Hash Table Header Page

The header page sits the at the **first level** of our disk-based extendible hash table, 
and there is only one header page for a hash table. It **stores the logical child pointers 
to the directory pages. **
The header page has the following fields:

| Variable Name         | Description                                    |
| --------------------- | ---------------------------------------------- |
| `directory_page_ids_` | An array of directory page ids                 |
| `max_depth_`          | The maximum depth the header page could handle |

### Hash Table Directory Page

Directory pages sit at the **second level** of our disk-based extendible hash table. 
Each of them **stores the logical child pointers to the bucket pages**, 
as well as **metadata** for handling bucket mapping and dynamic directory growing and 
shrinking. The directory page has the following fields:

| Variable Name      | Description                                    |
| ------------------ | ---------------------------------------------- |
| `max_depth_`       | The maximum depth the header page could handle |
| `global_depth_`    | The current directory global depth             |
| `local_depths_`    | An array of bucket page local depths           |
| `bucket_page_ids_` | An array of bucket page ids                    |

### Hash Table Bucket Page

Bucket pages sit at the **third level** of our disk-based extendible hash table. 
They are the ones that are actually **storing the key-value pairs. **
The bucket page has the following fields:

| Variable Name | Description                                                 |
| ------------- | ----------------------------------------------------------- |
| `size`        | The number of key-value pairs the bucket is holding         |
| `max_size`    | The maximum number of key-value pairs the bucket can handle |
| `array_`      | An array of bucket page local depths                        |

## Task #3 - Extendible Hashing Implementation

Your implementation needs to support insertions, point search and deletions. 
Your only strict API requirement is adhering to `Insert`, `GetValue`, and `Remove`.

!!! note "Note"

    You should use the page classes you implemented in Task #2 to store the key-value pairs as well as the metadata to maintain the hash table (page ids, global/local depths).

The extendible hash table is parameterized on arbitrary key, value, and key comparator types.

This project requires you to implement:

* Bucket splitting and merging
* Directory growing and shrinking