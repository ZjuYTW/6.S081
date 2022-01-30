---
title: MIT6.S081 Lecture11 Machinery about synchronize
data: 2021/12/16
categories:
  - MIT6.S081
tags:
  - MIT6.S081
toc: true # 是否启用内容索引
---



## Lec10 Machinery about synchronize

Today's lecture talks about some synchronize mechanism, from lock, condition to wait to give a clear concept of what these mechanisms really effect.

### Diagram about switch
To avoid deadlock and repeat processes while doing switch, kernel needs a right order to lock and unlock.  

* We need to make sure no other locks during swtch.
  * Assume we have just 1 CPU and once p1 switch with 1 lock to p2, if p2 also tring to acquire the lock-> deadlock
  * And acquire() turns off interrupt to avoid another deadlock...(because in interrupt handler, it also needs lock)

### Coordination -- wake and sleep

To make thread wait on specific condition or event.
Given a simple example synchronize code:
```c
static int tx_done; // has the UART finished sending?
static int tx_chan; // &tx_chan is the "wait channel"

// transmit buf[].
void
uartwrite(char buf[], int n)
{
  acquire(&uart_tx_lock);

  int i = 0;
  while(i < n){
    while(tx_done == 0){
      // UART is busy sending a character.
      // wait for it to interrupt.
      sleep(&tx_chan, &uart_tx_lock);
    }
    
    WriteReg(THR, buf[i]);
    i += 1;
    tx_done = 0;
  }

  release(&uart_tx_lock);
}

// handle a uart interrupt, raised because input has
// arrived, or the uart is ready for more output, or
// both. called from trap.c.
void
uartintr(void)
{
  acquire(&uart_tx_lock);
  if(ReadReg(LSR) & LSR_TX_IDLE){
    // UART finished transmitting; wake up any sending thread.
    tx_done = 1;
    wakeup(&tx_chan);
  }
  release(&uart_tx_lock);

  // read and process incoming characters.
  while(1){
    int c = uartgetc();
    if(c == -1)
      break;
    consoleintr(c);
  }
}
```

UART driver use `uartwrite()` to actually write character, we could find in code that they proceed 1 character 1 time so it needs to yield cpu instead of spin.  

UART hardware will raise interrupt to `uartintr` then wakeup  `uartwrite()` to consume character.

**lost wakeup** is situation that one process sends wakeup signal but missed somehow by receiver. It is caused by mistakes on adding lock. If we replace sleep() by broken_sleep() which just takes one parameter indicating the sleep channel.
```c
void
broken_sleep(void *chan)
{
  struct proc *p = myproc();
  
  // Must acquire p->lock in order to
  // change p->state and then call sched.
  acquire(&p->lock);

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;
  release(&p->lock);
}
```
Previous snippet will like:
```c
while(tx_done == 0){
      //sleep(&tx_chan, &uart_tx_lock);
      
      release(&uart_tx_lock);
      //lose wakeup window here!!
      broken_sleep(&tx_chan);
      acquire(&uart_tx_lock);
    }
```

So we need to do this three line atomically in sleep().
```c
void
sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();
  
  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won't miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.
  if(lk != &p->lock){  //DOC: sleeplock0
    acquire(&p->lock);  //DOC: sleeplock1
    release(lk);
  }

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;

  // Reacquire original lock.
  if(lk != &p->lock){
    release(&p->lock);
    acquire(lk);
  }
}

void
wakeup(void *chan)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == SLEEPING && p->chan == chan) {
      p->state = RUNNABLE;
    }
    release(&p->lock);
  }
}
```

In the wakeup, we have to check proc table preceding with acquiring p's lock. So sleep with firstly get p's lock then release `lk` to make sure all steps are atomic.

* Actually, semaphore is a more easier understaning way to use. Because caller have no worry about lost wakeups. (But internal semaphore, it takes good care about it)

### exit and kill

As a process exits, we have to free memory, free pagetable and trapframe, clean up states, free stack...
* We cannot kill another thread directly, because it may in some critical area.
* In exit(), the process should reparent its children and set its state into ZOMBIE
* Parent should explicitly use wait() to reap zombie children.
```c
wait(uint64 addr){
    ...
    for(;;){
        // Scan through table looking for exited children.
        havekids = 0;
        for(np = proc; np < &proc[NPROC]; np++){
            // this code uses np->parent without holding np->lock.
            // acquiring the lock first would cause a deadlock,
            // since np might be an ancestor, and we already hold p->lock.
            if(np->parent == p){
                // np->parent can't change between the check and the acquire()
                // because only the parent changes it, and we're the parent.
                acquire(&np->lock);
                havekids = 1;
                if(np->state == ZOMBIE){
                    // Found one.
                    if(addr != 0 && copyout(p->pagetable, addr, (char *)&np->xstate,
                                            sizeof(np->xstate)) < 0) {
                        release(&np->lock);
                        release(&p->lock);
                        return -1;
                    }
                    freeproc(np);
                    release(&np->lock);
                    release(&p->lock);
                    return pid;
                }
                release(&np->lock);
            }
        }
    }
    ...
}
```