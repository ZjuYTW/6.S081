---
title: MIT6.S081 Lab9
data: 2022/01/5
categories:
  - MIT6.S081
tags:
  - MIT6.S081
toc: true # 是否启用内容索引
---



## Lab9 -- Multithreading

### User thread implementation
In this section, we will implement a user-level thread. Like many user-level thread libraries do, it should have its own 
`pc, reregister and stack`. For each thread, lab has already written the storage area in the `struct thread`.  

But because we still needs to save / restore the register through `thread_switch`, we also need a `struct thread_context` to store register status.
```c
struct thread_context{
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```
Like what kernel does in kernel/swtch.S(), we copy the context switch assembly code to save `ra, sp and callee saved registers`.  

In our implementation, `ra` is used to store the return address of thread_switch.S namely the last switch out point's address. Cooperates with ra, `sp` provides stack pointer points to stack which stores more variables on stack.

Finally, we have to define where `ra` and `sp` starts.  
D note `sp` grows from high to low, so we need to set `sp = &state` and `ra` as the address of `func`

### Using thread
Last two sections require us to do coding on pthread. Some main API are following:
```cpp
pthread_mutex_t lock;            // declare a lock
pthread_mutex_init(&lock, NULL); // initialize the lock
pthread_mutex_lock(&lock);       // acquire lock
pthread_mutex_unlock(&lock);     // release lock
pthread_cond_wait(&cond, &mutex);  // go to sleep on cond, releasing lock mutex, acquiring upon wake up
pthread_cond_broadcast(&cond);     // wake up every thread sleeping on cond
```

Actually I don't want talk much about these because they are so easy for me? Maybe `Barrier`  is worthy a concise description.

To make all threads block till all of them synchronize at one point, we need `pthread_cond_wait()` and `pthread_cond_broadcase()`.  

In each round, if reached thread number are not enough, then they call `cond_wait()` to release the lock and sleep. Till the last thread reaches, it increment round and set 0 the thread counter then `cond_broadcast()`.

Note, don't forget to unlock properly.
