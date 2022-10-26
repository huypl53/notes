# Buffer Manager

The buffer manager (primarily configured by shared_buffers) is the part of Postgres that caches on-disk data in memory.

The PostgreSQL buffer manager comprises a buffer table, buffer descriptors, and buffer pool,

## Buffer Tag

In PostgreSQL, each page of all data files can be assigned a unique tag, i.e. a buffer tag. When the buffer manager receives a request, PostgreSQL uses the buffer_tag of the desired page.

The buffer_tag comprises three values:

- The RelFileNode

```cpp
typedef struct RelFileNode
{
    Oid         spcNode;        /* tablespace */
    Oid         dbNode;         /* database */
    Oid         relNode;        /* relation */
} RelFileNode;
```

- The fork number of the relation to which its page belongs

The fork numbers of tables, freespace maps and visibility maps are defined in 0, 1 and 2, respectively.

- The block number of its page.

For example, the buffer_tag `{(16821, 16384, 37721), 0, 7}` identifies the page that is in the `seventh` block, fork number = `0`, tablespace's OID = `16821`, database's OID = `16384` and relation's OID = `37721`

## How a Backend Process Reads Pages

![](https://user-images.githubusercontent.com/17776979/198048769-8f02d7a6-b107-4eba-982c-9b1e4354a6a8.png)

1. When reading a table or index page, a backend process sends a request that includes the page's `buffer_tag` to the buffer manager
2. The buffer manager returns the buffer_ID of the slot that stores the requested page. If the requested page is not stored in the buffer pool, the buffer manager loads the page from persistent storage to one of the buffer pool slots and then returns the buffer_ID's slot.
3. The backend process accesses the buffer_ID's slot (to read the desired page).

## Page Replacement Algorithm

When all buffer pool slots are occupied but the requested page is not stored, the buffer manager must select one page in the buffer pool that will be replaced by the requested page.

Since version 8.1, PostgreSQL has used **clock sweep** algorithm because it is simpler and more efficient than the LRU algorithm used in previous versions.

## Buffer Manager Structure

![](https://user-images.githubusercontent.com/17776979/198048832-d293601b-2839-4663-bba7-8b2cd03ff9bd.png)

The PostgreSQL buffer manager comprises three layers:

- The **buffer pool** is an array. Each slot stores a data file pages. The indices of the array slots are referred to as `buffer_ids`.
- The **buffer descriptors** layer is an array of buffer descriptors. Each descriptor has one-to-one correspondence to a buffer pool slot and holds metadata of the stored page in the corresponding slot.
- The **buffer table** is a hash table that stores the relations between the buffer_tags of stored pages and the buffer_ids of the descriptors that hold the stored pages' respective metadata.

### Buffer Table

A buffer table can be logically divided into three parts: a hash function, hash bucket slots, and data entries.

A data entry comprises two values: the buffer_tag of a page, and the buffer_id of the descriptor that holds the page's metadata. For example, a data entry `Tag_A, id=1` means that the buffer descriptor with buffer_id `1` stores metadata of the page tagged with `Tag_A`.

![](https://user-images.githubusercontent.com/17776979/198048896-a90e7ca1-7f06-426e-ac93-0db67e8a42f4.png)

### Buffer Descriptor

Buffer descriptor holds the metadata of the stored page in the corresponding buffer pool slot. The buffer descriptor structure is defined by the structure BufferDesc.

The structure BufferDesc is defined in [src/include/storage/buf_internals.h](https://github.com/postgres/postgres/blob/master/src/include/storage/buf_internals.h).

```cpp
/*
 * Flags for buffer descriptors
 *
 * Note: TAG_VALID essentially means that there is a buffer hashtable
 * entry associated with the buffer's tag.
 */
#define BM_DIRTY                (1 << 0)    /* data needs writing */
#define BM_VALID                (1 << 1)    /* data is valid */
#define BM_TAG_VALID            (1 << 2)    /* tag is assigned */
#define BM_IO_IN_PROGRESS       (1 << 3)    /* read or write in progress */
#define BM_IO_ERROR             (1 << 4)    /* previous I/O failed */
#define BM_JUST_DIRTIED         (1 << 5)    /* dirtied since write started */
#define BM_PIN_COUNT_WAITER     (1 << 6)    /* have waiter for sole pin */
#define BM_CHECKPOINT_NEEDED    (1 << 7)    /* must write for checkpoint */
#define BM_PERMANENT            (1 << 8)    /* permanent relation (not unlogged) */

src/include/storage/buf_internals.h
typedef struct sbufdesc
{
   BufferTag    tag;                 /* ID of page contained in buffer */
   BufFlags     flags;               /* see bit definitions above */
   uint16       usage_count;         /* usage counter for clock sweep code */
   unsigned     refcount;            /* # of backends holding pins on buffer */
   int          wait_backend_pid;    /* backend PID of pin-count waiter */
   slock_t      buf_hdr_lock;        /* protects the above fields */
   int          buf_id;              /* buffer's index number (from 0) */
   int          freeNext;            /* link in freelist chain */

   LWLockId     io_in_progress_lock; /* to wait for I/O to complete */
   LWLockId     content_lock;        /* to lock access to buffer contents */
} BufferDesc;
```

To simplify the following descriptions, three descriptor states are defined:

- **Empty** When the corresponding buffer pool slot does not store a page (i.e. refcount and usage_count are 0)
- **Pinned** When the corresponding buffer pool slot stores a page and any PostgreSQL processes are accessing the page (i.e. refcount and usage_count are greater than or equal to 1)
- **Unpinned** When the corresponding buffer pool slot stores a page but no PostgreSQL processes are accessing the page (i.e. usage_count is greater than or equal to 1, but refcount is 0)

When the PostgreSQL server starts, the state of all buffer descriptors is empty. In PostgreSQL, those descriptors comprise a linked list called **freelist**.

## Buffer Pool

The buffer pool is a simple array that stores data file pages, such as tables and indexes. Indices of the buffer pool array are referred to as buffer_ids. The buffer pool slot size is 8 KB, which is equal to the size of a page. Thus, each slot can store an entire page.

## How the Buffer Manager Works

### Accessing a Page Stored in the Buffer Pool

1. Create the buffer_tag of the desired page and compute the hash bucket slot
2. Acquire the BufMappingLock partition that covers the obtained hash bucket slot in shared mode (this lock will be released in step (5)).
3. Look up the entry whose tag is 'Tag_C' and obtain the buffer_id from the entry. In this example, the buffer_id is 2.
4. Pin the buffer descriptor for buffer_id 2, i.e. the refcount and usage_count of the descriptor are increased by 1
5. Release the BufMappingLock.
6. Access the buffer pool slot with buffer_id 2.

![](https://user-images.githubusercontent.com/17776979/198057798-5e64cba3-b5f2-4ca0-8a27-35ecf475e065.png)

Then, when reading rows from the page in the buffer pool slot, the PostgreSQL process acquires the `shared content_lock` of the corresponding buffer descriptor. Thus, buffer pool slots can be read by multiple processes simultaneously.

When inserting (and updating or deleting) rows to the page, a Postgres process acquires the `exclusive content_lock` of the corresponding buffer descriptor (note that the dirty bit of the page must be set to '1').

After accessing the pages, the refcount values of the corresponding buffer descriptors are decreased by 1.

### Loading a Page from Storage to Empty Slot

1. Look up the buffer table (we assume it is not found).

```
- Create the buffer_tag of the desired page (in this example, the buffer_tag is 'Tag_E') and compute the hash bucket slot.
- Acquire the BufMappingLock partition in shared mode.
- Look up the buffer table (not found according to the assumption).
- Release the BufMappingLock.
```

2. Obtain the empty buffer descriptor from the freelist, and pin it. In this example, the buffer_id of the obtained descriptor is 4.
3. Acquire the BufMappingLock partition in exclusive mode (this lock will be released in step (6)).
4. Create a new data entry that comprises the buffer_tag 'Tag_E' and buffer_id 4; insert the created entry to the buffer table.
5. Load the desired page data from storage to the buffer pool slot with buffer_id 4 as follows:

```
- Acquire the exclusive `io_in_progress_lock` of the corresponding descriptor.
- Set the `io_in_progress` bit of the corresponding descriptor to '1 to prevent access by other processes.
- Load the desired page data from storage to the buffer pool slot
- Change the states of the corresponding descriptor; the `io_in_progress` bit is set to `0`, and the valid bit is set to `1`.
- Release the `io_in_progress_lock`.
```

6. Release the BufMappingLock.
7. Access the buffer pool slot with buffer_id 4.

![](https://user-images.githubusercontent.com/17776979/198059184-54397a09-0f49-46be-9edc-f5066008d688.png)

## Dirty Pages

When data is modified in memory, the page in which this data is stored is called a `dirty page`, as long as the modifications are not written to disk. So if there is a page that is different in the shared buffer and the disk, it is called a dirty page. The buffer manager flushes the dirty pages to storage with the assistance of two subsystems called – **checkpointer** and **background writer**.
