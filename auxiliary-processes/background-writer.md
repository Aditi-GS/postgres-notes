# 2. Background Writer:
<details>
    <summary>An auxiliary process that writes dirty data pages from shared memory to the file system. It wakes up periodically, but works only for a short period in order to distribute its expensive I/O activity over time to avoid generating larger I/O peaks which could block other processes.
    </summary>
<br>

When the number of clean shared buffers appears to be insufficient, the background writer writes some dirty buffers to the file system and marks them as clean. This reduces the load on checkpointer and also the likelihood that server processes handling user queries will be unable to find clean buffers and have to write dirty buffers themselves. However, the background writer does cause a net overall increase in I/O load, because while a repeatedly-dirtied page might otherwise be written only once per checkpoint interval, the background writer might write it several times as it is dirtied in the same interval. Unlike checkpointer which performs a linear scan to check for all dirty pages, bgwriter selects the dirty pages to write() based on clock-sweep algorithm - the same used for eviction as well. 

Similar to the checkpointer, bgwriter:
1. Acquires the `buffer header spinlock` to check the `state` of the buffer descriptor of a slot.
2. If `BM_DIRTY` is NOT set -> skip
   If `BM_IO_IN_PROGRESS` is set -> skip
   If `BM_DIRTY` set AND `BM_IO_IN_PROGRESS` clear -> Set it to `BM_IO_IN_PROGRESS`
3. Releases the spin lock 
4. Acquires the `content_lock` in shared mode to write() the contents of the dirty page
5. Clears the `BM_IO_IN_PROGRESS` state

Bgwriter never calls fsync(). It calls write() only - to push pages from shared buffers to OS page cache. The checkpointers is incharge of pushing pages from shared buffers to stable disk storage.

Both backends and the bgwriter forward their fsync() requests to the checkpointer via a shared-memory fsync requests queue. The checkpointer absorbs these periodically (every 1000 writes during the checkpoint loop, and continuously while idle) and issues the actual fsync() calls centrally. If the queue overflows, the caller falls back to doing its own fsync().

`bgwriter_delay` parameter specifies the delay between activity rounds for the background writer. In each round the writer issues writes for some number of dirty buffers - controlled by the another parameter `bgwriter_lru_maxpages`. Setting `bgwriter_lru_maxpages` to 0 disables background writing.

```sql
SHOW bgwriter_delay;
DEFAULT: 200 ms
```

```sql
SHOW bgwriter_lru_maxpages;
DEFAULT: 100
```

### More on Relevant Server Configured Parameters: 
<u>For detailed definitions</u>:
[Server Configurations - Background Writer](https://www.postgresql.org/docs/18/runtime-config-resource.html#RUNTIME-CONFIG-RESOURCE-BACKGROUND-WRITER)

> ***bgwriter_delay (integer)***

When there are no dirty buffers in the buffer pool, though, it goes into a longer sleep regardless of bgwriter_delay. If this value is specified without units, it is taken as milliseconds. Setting bgwriter_delay to a value that is not a multiple of 10 might have the same results as setting it to the next higher multiple of 10.

> **bgwriter_lru_multiplier floating point**

Multiplied with the average number of new buffers needed by server processes during recent rounds to estimate how many clean buffers will be required in the next round. The result is capped by bgwriter_lru_maxpages.

</details>

---