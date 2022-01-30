---
title: MIT6.S081 Lecture8 Interrupt
data: 2021/12/05
categories:
  - MIT6.S081
tags:
  - MIT6.S081
toc: true # 是否启用内容索引
---



## Interrupt

### RISCV interrupt-related registers
SIE -- supervisor interrupt enabled register, 3 bits for software int, external int and timer int.
SSTATUS -- supervior status register, one bit to enable interrupts.
SIP -- supervisor interrupt pending register
SCAUSE -- supervisor cause register
STVEC -- supervisor trap vector register
MDELEG -- machine delegate register

### Use of PLIC
PLIC, platform-level interrupt controller, passes interrupt on to a CPU. It is the handler of interrupts, that distributes interrupts among cores. If no CPU claims the interrupt, the interrupt stays pending till eventually be delivered to some core.

### Concurrency external device

In the RISCV xv6, external devices are running parallelly with kernel and controlled by `driver`. The way device used to communicate with kernel, is interrupt which has three properties:
* Asynchronous
  * interrupts running process
  * interrupt handler may not run in context of process who caused interrupt.
* Concurrency
  * devices and process run in parallel
* Programming devices
  * device can be difficult to program
  

Refer to xv6-book, we have the definition of driver:
> Many device drivers execute code in two contexts: a top half that runs in a process’s kernel
thread, and a bottom half that executes at interrupt time. The top half is called via system calls
such as read and write that want the device to perform I/O. This code may ask the hardware
to start an operation (e.g., ask the disk to read a block); then the code waits for the operation
to complete. Eventually the device completes the operation and raises an interrupt. The driver’s
interrupt handler, acting as the bottom half, figures out what operation has completed, wakes up a
waiting process if appropriate, and tells the hardware to start work on any waiting next operation.

In the lecture, the professor uses one case to walk through the workflow of `UART`, and the xv6-book also writes the detailed explanation of how kernel interacts with devices.
I just want to write down some ideas attracts me most during the lecture.

* Each device is mapped to a physical memory address. And there are handful of control register on it, like`RHR`(receive holding register) and `THR`(transmit holding register)...
* Both user type a byte input and UART completes sending a byte both raise a interrupt.
* To display a character, driver puts character into UART's send FIFO then interrupt when character has been sent
* To receive a keyboard hit, user hits key then returns in UART interrupt. Driver gets character from UART's receive FIFO

### polling

Consider the fact that nowadays some devices generate events faster than one per microsecond, eg. gigabit ethernet can deliver 1.5 million small packets / second.

The way to deal with those huge amount of interrupts is polling -- processor spins until device wants attention.
* Wastes processor cycles if device is slow
* But inexpensive if device is fast
  * No saving of regiser, etc