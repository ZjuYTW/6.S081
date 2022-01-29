### Linux ex3fs crash recovery system

#### xv6 Design Defects

* Every block needs to be written twice(one for log and another for fs)
* Syscall needs wait for committing
* disk I/O is synchronized



#### ext3 Journal Design

ext3 could track on multiple transactions' status at one time to get more parallelism. Similar to xv6, ext3 has **write-back block cache** and maintains each transaction a **transaction info**, includes 

* Every transaction has a sequence number
* Revised block number by this tnx
* handles

On disk **circular log** has:

* **log super block**: recording *offset* of transaction with the lowest sequence number and its *sequence number*
* **descriptor block**: Every transaction's head block, recording seq# and home block#, and magic#
* **data block**
* **commit block**: Every transaction's tail block, has magic#.



When log is full or elapse times out, ext3 will write *log block* into *home disk* starts at the smallest seq# transaction.



#### Commit Transaction

* Temporarily block new syscalls
* Wait for *outstanding syscall* ends, because one transaction has a rather long time window, we need to wait for all syscalls in the window to finish.
* start a new transaction, unblock syscalls
* write block numbers into *descriptor block*
* write corresponding *data block*
* write *commit block*, after it is written, commit finishes.
* write to *home location*
* release  *log block*

#### Recovery Steps

* After rebooting, system first looks at super block and seeks for smallest valid seq#'s transaction
* Find the log's tail, if it missed a commit flag( by magic #) or encounter a false seq #, we just skip this.
* Write all valid commit log



#### Performance Analysis

*  Asynchronous disk update
  * syscall don't have to wait for disk I/O, instead it just modify buffer cache and different syscall's log could be `absorbed` for group commit
  * But we need to be careful with this **may not flushed** syscall. **Could use fsync(fd) to force flush**
* Batching
  * Group Commit
  * Amortize block seeking time
  * write absorption
  * disk scheduling: Write block in a ordered sequence instead of random I/O
* Concurrency
  * Log enables multiple transaction, each transaction may in different stage:
    * Open : Able to accept new syscall's write
    * Committing
    * Committed
    * Old: waiting to be freed 



#### Collision Handling 

Assume a scenario that when **committing** a transaction, because ext3 doesn't block syscall, if a new transaction needs to do modification basing on previous transaction's blocks. And we need to make sure current buffer cache won't be modified while committing, so we could grep a buffer cache's copy to the new transaction. 