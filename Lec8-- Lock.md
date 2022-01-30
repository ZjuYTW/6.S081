---
title: MIT6.S081 Lecture9 Lock
data: 2021/12/10
categories:
  - MIT6.S081
tags:
  - MIT6.S081
toc: true # 是否启用内容索引
---



## Locks

This lecture talks about locks, I believe many people already have the concept of locks. So I just skip the very beginning of the lecture.

### Basic lock guideline
A consecuative rule: 2 processes accessing a shared data structure.
If one is a writer=> lock data structure is needed.  
* Too strict: there is a style called lock-free programming. Isn't all situation needed lock
* Too loose: Some situation are no shared memory but still need lock

### Some problems
* Deadlock
  * When two process trying to acquire the lock holded by other, deadlock may happen (Not strict definition)
* Lock vs Modularity
  * Lock ordering needes the lock are global.
  * If exists m1.g() calls m2.f() which uses lock. Then f()'s lock need to be visible to m1. Violates abstract principle.
  
* Lock vs performance
  * Need to split up data structure
  * Best split is a challenge
  * A better way to find the practical lock granularity.
    * Start with coarse-grained locks 
    * Meassure the performance
    * If multiple thread are trying to get lock -> serialized -> need redesign

### The implementation of lock
**Spinlock**  
Need hw support, test and set instruction  
test_and_set(addr, r1, r2):  
lock()  
&ensp;&ensp;&ensp;tmp <- [addr]  
&ensp;&ensp;&ensp;[arr] <- r1  
&ensp;&ensp;&ensp;r2   <- tmp  
unlock()  
return r2  
c std function -> __sync_lock_test_and_set(void*, int)

### Memory layout
Due to compiler optimzation, some instructions could be resorted and exectued in different sequential than original written one.  
The spinlock could use __sync_synchronize() function to denote between two `barrier`, compiler shouldn't rearrange any instruction order.

