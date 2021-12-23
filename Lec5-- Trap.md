## Lec6- Isolation and syscall
### Supervisor registers
* stap -- Store the page table's base address
* stvec -- Store the address of trap program -- `trampolines`, note that this address is mapped on user's page table but without `PTE_U` which means the user can't modify it.
* sepc -- Store the address in the user mode when ecall happens
* sstrach -- Store the address of `trapframe` which stores a frame useds to temporarily keep the user's registers.
    * See the usage of `csrrw a0, sstrach, a0` in the trampoline.S for example.