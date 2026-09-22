## Despite `background writer` and `checkpointer`, in scenarios with heavy write workload - these processes fall behind in performance. 

Under a high write traffic, the rate at which the pages get dirtied can outpace both, leaving the shared buffers saturated with dirty pages. 
The bg writer wouldn't have been able to proactively clean enough buffer slots ahead of demand and the checkpointer wouldn't have yet arrived to flush all dirty pages. 
In such cases - when the backend is in need of a clean buffer slot and none are available - it is forced to write a dirty page from shared buffer to the disk based on `clock-sweep (LRU) eviction` that frees a slot. This has to happen synchronously before the backend can reuse the slot for a new page it wants to read - introducing disk I/O wait time.
If this is occuring often then tuning these parameters might show improvement: 

1. Many small I/O bursts 

- Increasing `bgwriter_lru_maxpages`        (more pages cleaned)
- Decreasing `bgwriter_delay`               (trigger frequent flushes)
- Increasing `checkpoint_completion_target` (avoid I/O spikes)
- Decreasing `checkpoint_timeout`           (triggers frequent flushes)

OR 

2. One paced flush with interval 

- Increasing `bgwriter_lru_maxpages`
- Decreasing `bgwriter_delay`               
- Increasing `checkpoint_completion_target`
- Increasing `checkpoint_timeout`

Despite having a lower latency, the first strategy introduces a drawback of ***WAL amplification*** due to frequent `FPWs` (Full Page Writes). Decreasing `checkpoint_timeout` results in 
more frequent checkpoints -> more pages eligible for FPWs (hot pages) -> More FPIs -> More WAL generated. 
The checkpointer may enter a self-reinforcing loop if the WAL volume increases due to FPIs exceeding `max_wal_size`.

> What are Full Page Writes and their purpose ?
- When `full_page_writes` parameter is on, the backend which makes the first modification to page(s) after a checkpoint, writes the entire content of each of the affected pages to WAL.
- This is needed because if a page write is interupted due to OS crash or system failure, the content of the page will be half old and half new - resulting in a `torn page`.
- Normal WAL records can't fix torn pages during post crash recovery because they only store change data. This might lead to either unrecoverable data 
corruption or silent data corruption after the system failure.

Storing the full page image (FPI) guarantees that the page can be correctly restored, but at the price of increasing the amount of data that must be written to WAL. 
It is sufficient to write this only during the first change of each page after
a checkpoint. One way to reduce the cost of full page writes is to increase the checkpoint interval parameters => 
fewer checkpoints -> fewer FPWs -> lesser WAL Volume. 
This is mainly applicable to "hot" pages - pages that are refered to and modified often.

---