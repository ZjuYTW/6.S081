## Lab6 -- Multithreading

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
Like the kernel does in kernel/swtch.S(), we copy the context switch assembly code just to save `ra, sp and callee saved registers`.
In our implementation, `ra` is used to store the return address of thread_switch.S namely the last switch out point's address. Cooperates with ra, `sp` provides stack pointer points to stack which stores more variables on stack.

So