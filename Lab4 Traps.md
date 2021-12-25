## Lab4

### Part2 --Backtrace
backtrace is the method that prints the address of call stack. To implement backtrace, we should know some key points :  
* In RISC-V, stack grows from high to low.
* fp points to the start of a function(highest address), then `Return Address` (offset by -8), then previous frame pointer address(offset by -16)
* We should print the address of `Return Address`(means the address of code)
* Stack size are 1 PGSIZE and aligned.  

So we just need to fetch the frame pointer by the follower assembly code:
```c
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```
Then recursively fetch the previous frame pointer, note we should use check the boundary of one page.

### Part3 -- Alarm
In this part, the alarm are like a callback function but trigged by timer(Event driven).  
So `sys_sigalarm` are the setter of this scheme, first paramter is the `count down` number and second is the hanlder of callback function. 'sys_sigreturn' is the return function that reloads context of origin function(we will see in the lab)  
Firstly, we should register two functions in `/user` and make a syscall reference for them, then we go to `/kernel` to complete these two syscalls. In the `sys_sigalarm`, we should keep the tick intervals and the function pointer in the `struct proc` for later call. Then `sys_sigreturn`, for now it just return 0 is ok.  

Note that in the `/kernel/trap.c:usertrap()`, it handles the all exceptions (syscall, timer interupt...).  Once it is called by an interupt of timer, we should increase the tick number in the `struct proc` and once `ticks == interval`, go to handler.  

By modifing the address of `pc`, we could jump to the handler. But the way jump back to the origin code should be considered carefully -- The lab hints provide us a process to return, which is the `sys_sigreturn`.  We should reload all the origin(as many as possible to keep the context safy) in it then jump back to origin place(by `pc` register) 

#### Hints
* p->trapframe->epc points to the return address once the process jumps from kernel
* Record as many as possible register in the `proc` especially `pc, sp, ra` and so on.
* add backtrace() in `panic()` so that whenever a panic happens, it will print a backtrace information.
  
