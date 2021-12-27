## Lab 5 Lazy Allocation

In this lab, we gonna implement `lazy allocation` which means delay the true memory allocation till we actually needs it.
### lazy sbrk()


To start with, we should modify the sbrk() from eager allocation to lazy one. 
But we should note that the parameter `n` can be arbitary number, either `+ or -` so once the number is positve do the lazy allocation, but if the number is negative, we should do the deallocation immediately.

### Lazy Allocation

The time to truely allocate a page, is when kernel receives a `No.13 or 15` exception, which tells the kernel a non-PTE_V page is being accessed. If the page is lazy allocated but not been truely allocated yet, we should do the allocation and return to the same instruction in the user mode.

We should check the boundary case, that if  
* va $\in$ [0, p->sz) (Exclusively contain! Important)
* va $\notin$ guard page range  


Then the virtual address is valid to allocate a new physical page for it. In this part, we could refer to the implementation of `uvmalloc`
```cpp
void allocate_page(uint64 va){
  pagetable_t pagetable = myproc()->pagetable;
  va = PGROUNDDOWN(va);
  void* mem = kalloc();
  if(mem == 0){
    printf("usertrap(): scause %p pid=%d\n", r_scause(), myproc()->pid);
    printf("Out of memory!\n");
    myproc()->killed = 1;
    exit(-1);
  }
  memset(mem, 0, PGSIZE);
  if(mappages(pagetable, va, PGSIZE, (uint64)mem, PTE_W|PTE_R|PTE_U) != 0){
    printf("usertrap(): scause %p pid=%d\n", r_scause(), myproc()->pid);
    printf("Can't allocate the page table\n");
    kfree(mem);
    uvmdealloc(pagetable, PGROUNDUP(va), PGROUNDDOWN(va));
    myproc()->killed = 1;
    exit(-1);
  }
}
```

Note that in the original implementation of xv6, we do the eager allocation, which means whenever we visit a virtual address, the physical page is mapped. So there are many `panic` check in the `uvmunmap, walk, uvmcopy...` should be canceled, because now many ptes are `lazy` allocated but not been mapped yet. So just leave to `usertrap()` to decide whether page should be accessed.

Last, in the `sbrkarg` test, it tests the copyin and copyout functions.
Because in the kernel mode, kernel simulates the dereference process by `walkaddr`, it won't cause a exception. So we should manually modify the function and add a allocation to it.
```c
va0 = PGROUNDDOWN(dstva);
if(check_valid(va0) && ((pte = walk(pagetable, va0, 0)) == 0 || (*pte & PTE_V) == 0)){
    allocate_page(va0);
}
```


#### Some hints
* When encounter a bug, check the backtrace in the gdb.
* The way to decide whether a `va` is in the guard page, I use `if(va < PGROUNDDOWN(p->trapframe->sp))`. 
  * I saw some other's blog says could use `r_sp()`, but it just return the kernel stack pointer...
* Don't forget to add codes on copyin, copyout and copyinstr.
 
