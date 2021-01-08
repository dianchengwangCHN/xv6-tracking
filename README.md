# xv6-tracking
Repository to record the progress for self-learning MIT 6.S081. Out of the respect to MIT for making all the great course resource public, the code implementation will only be kept in a separate private repository.
- [Util Lab](#util-lab)
    - [Lab Result](#lab-result)
- [Syscall Lab](#syscall-lab)
    - [Lab Result](#lab-result-1)
- [Page Table Lab](#page-table-lab)
    - [Lab Result](#lab-result-2)
- [Trap Lab](#trap-lab)
    - [Lab Result](#lab-result-3)
- [Lazy Lab](#lazy-lab)
    - [Lab Result](#lab-result-4)
- [Copy-on-Write Lab](#copy-on-write-lab)
    - [Lab Result](#lab-result-5)

## Util Lab
This lab requires to write some basic user space functions utilizing some library functions and system call functions.

Most of the problems in this lab are not very hard to implement. The most interesting one is the `primes.c` program. The tricky part is every process will have two pipes to communicate with (one to communicate with its parent, one to communicate with its child), so how to use the `fork` and `pipe` to create the process? The way I approach this problem is:
- For a process, it will have two sets of file descriptors (corresponding to two pipes). It also keeps a `flag` to indicate which pipes is the one it should use to communicate with its parent.
- Main process will initialize a pipe and set the `flag` to correspond pipe. Then after the `fork`, the child process will get the same memory image as the parent. Since the `flag` will be the same as its parent, it knows which set of file descriptors to use to read the input, and the other one will be the one it uses to communicate with its child. Then it can `fork` to create the child, write values to the child pipe, and `wait` for child to return.
- For a child process:
    - If `read` returns 0, it can `exit`.
    - Otherwise, it will print the value and be responsible for filter out all the multiples of this value. It will initialize a pipe with another file descriptors set to write output to its child. Note here the `flag` will need to be change to correspond pipe.
    - `fork` to create the child, write filtered value to its child pipe, and then `wait` for child.

### Lab Result
```sh
== Test sleep, no arguments ==
$ make qemu-gdb
sleep, no arguments: OK (5.6s)
== Test sleep, returns ==
$ make qemu-gdb
sleep, returns: OK (1.8s)
== Test sleep, makes syscall ==
$ make qemu-gdb
sleep, makes syscall: OK (1.6s)
== Test pingpong ==
$ make qemu-gdb
pingpong: OK (1.0s)
== Test primes ==
$ make qemu-gdb
primes: OK (2.3s)
== Test find, in current directory ==
$ make qemu-gdb
find, in current directory: OK (1.4s)
== Test find, recursive ==
$ make qemu-gdb
find, recursive: OK (1.1s)
== Test xargs ==
$ make qemu-gdb
xargs: OK (2.2s)
== Test time ==
time: OK
Score: 100/100
```

## Syscall Lab

### Lab Result
```sh
== Test trace 32 grep ==
$ make qemu-gdb
trace 32 grep: OK (4.8s)
== Test trace all grep ==
$ make qemu-gdb
trace all grep: OK (1.7s)
== Test trace nothing ==
$ make qemu-gdb
trace nothing: OK (1.0s)
== Test trace children ==
$ make qemu-gdb
trace children: OK (11.5s)
== Test sysinfotest ==
$ make qemu-gdb
sysinfotest: OK (2.1s)
== Test time ==
time: OK
Score: 35/35
```

## Page Table Lab
This is the first lab involving much kernel programming. To get the kernel page mapping correct is tricky. The lab Q&A lecture is pretty helpful, which eventually saves me after I spent a few days of debugging.

**A few things to note:**
- kernel page table and user page table are completely separate. `walk` is the bridge to get kernel space and user space work together. Before adding virtual address of user space to kernel page table, kernel mode and user mode are essentially using physical address to communicate.
- I eventually followed the approach from the lab Q&A lecture to finish this lab. While to create a new kernel page for each process in `allocproc` is not impossible, the approach may cause some performance issue as you will need to free all the 3-level page tables when the process is done. I also tried to reuse everything in global kernel page table except allocating each kernel stack dynamically, but I was not able to get this working.
- In the third problem, I was also thinking about to add user virtual address mappings only to the level 2 kernel page table such that level 1 and level 0 user page table can be reused. However, the problem here is kernel cannot access PTE with `PTE_U` flag, so I eventually gave up this approach.

### Lab Result
```sh
== Test pte printout ==
$ make qemu-gdb
pte printout: OK (5.2s)
== Test answers-pgtbl.txt == answers-pgtbl.txt: OK
== Test count copyin ==
$ make qemu-gdb
count copyin: OK (1.3s)
== Test usertests ==
$ make qemu-gdb
(230.3s)
== Test   usertests: copyin ==
  usertests: copyin: OK
== Test   usertests: copyinstr1 ==
  usertests: copyinstr1: OK
== Test   usertests: copyinstr2 ==
  usertests: copyinstr2: OK
== Test   usertests: copyinstr3 ==
  usertests: copyinstr3: OK
== Test   usertests: sbrkmuch ==
  usertests: sbrkmuch: OK
== Test   usertests: all tests ==
  usertests: all tests: OK
== Test time ==
time: OK
Score: 66/66
```


## Trap Lab
Overall for this lab, it is harder to understand how trap works than to implement lab solution.

The high level workflow of trap is:
- Some syscall is issued from user space and it triggers `ecall`.
- program will be redirected to address in `stvec`, which is `trampoline`.
- `uservec` in `trampoline` will save the registers' state and swap the user page table with kernel page table. Note here since `trampoline` and `trapframe` apply the same mapping in kernel page table and user page table, so page table swap won't affect the program execution. Then it will redirect program to `usertrap`.
- `usertrap` will handle the trap according to certain logic. Then it will set some states in trapframe and some system hardware, including `sepc`, which is the program counter that user space program will start to run after the trap. After setting these states, `usertrap` will jump to `userret`, which will restore the registers' state stored in previous `uservec`. Then program will return to user space.

The problems in the lab are not hard to implement. `backtrace` is simply tracking the frame pointer and print the return address all the way to top of the page of the kernel stack. `alarm` basically needs to set the `epc` in `usertrap` and store the user registers' state when timer interrupt occurred, so it can restore the state when `sys_sigreturn` is triggered.

### Lab Result
```sh
== Test answers-traps.txt == answers-traps.txt: OK
== Test backtrace test == backtrace test: OK (5.4s)
== Test running alarmtest == (4.0s)
== Test   alarmtest: test0 ==
  alarmtest: test0: OK
== Test   alarmtest: test1 ==
  alarmtest: test1: OK
== Test   alarmtest: test2 ==
  alarmtest: test2: OK
== Test usertests == usertests: OK (283.4s)
    (Old xv6.out.usertests failure log removed)
== Test time ==
time: OK
Score: 85/85
```

## Lazy Lab
Lazy Lab is mostly straightforward except the `sbrkarg` test is tricky. Recall from the page table lab, user page table does not overlap with kernel page table except `trampoline` and `trapframe`. The way `write` works is that kernel first follows the user page table to copy the user input to the kernel space and then performs the write, so `walkaddr` is actually where kernel can know a page is not allocated.

### Lab Result
```sh
== Test running lazytests == (4.8s) 
== Test   lazy: map == 
  lazy: map: OK 
== Test   lazy: unmap == 
  lazy: unmap: OK 
== Test usertests == (253.8s) 
== Test   usertests: pgbug == 
  usertests: pgbug: OK 
== Test   usertests: sbrkbugs == 
  usertests: sbrkbugs: OK 
== Test   usertests: argptest == 
  usertests: argptest: OK 
== Test   usertests: sbrkmuch == 
  usertests: sbrkmuch: OK 
== Test   usertests: sbrkfail == 
  usertests: sbrkfail: OK 
== Test   usertests: sbrkarg == 
  usertests: sbrkarg: OK 
== Test   usertests: stacktest == 
  usertests: stacktest: OK 
== Test   usertests: execout == 
  usertests: execout: OK 
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyout == 
  usertests: copyout: OK 
== Test   usertests: copyinstr1 == 
  usertests: copyinstr1: OK 
== Test   usertests: copyinstr2 == 
  usertests: copyinstr2: OK 
== Test   usertests: copyinstr3 == 
  usertests: copyinstr3: OK 
== Test   usertests: rwsbrk == 
  usertests: rwsbrk: OK 
== Test   usertests: truncate1 == 
  usertests: truncate1: OK 
== Test   usertests: truncate2 == 
  usertests: truncate2: OK 
== Test   usertests: truncate3 == 
  usertests: truncate3: OK 
== Test   usertests: reparent2 == 
  usertests: reparent2: OK 
== Test   usertests: badarg == 
  usertests: badarg: OK 
== Test   usertests: reparent == 
  usertests: reparent: OK 
== Test   usertests: twochildren == 
  usertests: twochildren: OK 
== Test   usertests: forkfork == 
  usertests: forkfork: OK 
== Test   usertests: forkforkfork == 
  usertests: forkforkfork: OK 
== Test   usertests: createdelete == 
  usertests: createdelete: OK 
== Test   usertests: linkunlink == 
  usertests: linkunlink: OK 
== Test   usertests: linktest == 
  usertests: linktest: OK 
== Test   usertests: unlinkread == 
  usertests: unlinkread: OK 
== Test   usertests: concreate == 
  usertests: concreate: OK 
== Test   usertests: subdir == 
  usertests: subdir: OK 
== Test   usertests: fourfiles == 
  usertests: fourfiles: OK 
== Test   usertests: sharedfd == 
  usertests: sharedfd: OK 
== Test   usertests: exectest == 
  usertests: exectest: OK 
== Test   usertests: bigargtest == 
  usertests: bigargtest: OK 
== Test   usertests: bigwrite == 
  usertests: bigwrite: OK 
== Test   usertests: bsstest == 
  usertests: bsstest: OK 
== Test   usertests: sbrkbasic == 
  usertests: sbrkbasic: OK 
== Test   usertests: kernmem == 
  usertests: kernmem: OK 
== Test   usertests: validatetest == 
  usertests: validatetest: OK 
== Test   usertests: opentest == 
  usertests: opentest: OK 
== Test   usertests: writetest == 
  usertests: writetest: OK 
== Test   usertests: writebig == 
  usertests: writebig: OK 
== Test   usertests: createtest == 
  usertests: createtest: OK 
== Test   usertests: openiput == 
  usertests: openiput: OK 
== Test   usertests: exitiput == 
  usertests: exitiput: OK 
== Test   usertests: iput == 
  usertests: iput: OK 
== Test   usertests: mem == 
  usertests: mem: OK 
== Test   usertests: pipe1 == 
  usertests: pipe1: OK 
== Test   usertests: preempt == 
  usertests: preempt: OK 
== Test   usertests: exitwait == 
  usertests: exitwait: OK 
== Test   usertests: rmdot == 
  usertests: rmdot: OK 
== Test   usertests: fourteen == 
  usertests: fourteen: OK 
== Test   usertests: bigfile == 
  usertests: bigfile: OK 
== Test   usertests: dirfile == 
  usertests: dirfile: OK 
== Test   usertests: iref == 
  usertests: iref: OK 
== Test   usertests: forktest == 
  usertests: forktest: OK 
== Test time == 
time: OK 
Score: 119/119
```

## Copy-on-Write Lab
Copy on write is similar to lazy page allocation. A few key points:
- When `fork` calls `uvmcopy`, let the child page table entry to be mapped to same physical address as its parent. The PTE should be set unwritable (with `PTE_W` as 0). For distinguish if a page is COW page, I also use the reserved flag bit 8 (`PTE_COW`) as an indicator. Keep track of the page reference to know when should the page be actually freed.
- Change the `kfree` to first decrement the page reference count and then check if it needs to be freed.
- Add trap handler to allocate a new page when the page fault if caused by COW. The new page should be given the write access (`PTE_W`) and clear the COW flag (`PTE_COW`). Depending on the implementation approach, the old page entry could be set to `PTE_W & ~PTE_COW` when it only has 1 reference, or it can be left as unchanged until the next page fault on it is triggered.
- Similar to lazy page allocation lab. Because the way kernel and user space passing arguments to communicate, a potential page fault could be triggered when user performs a `read` since kernel needs to follow the user page table to find and write content to a physical address. Since the process is implemented by software, so page fault will not be triggered by hardware and it is required to be handled by our code. Different from the lazy allocation, this will only happen when user try to read but not on write.

### Lab Result
```sh
== Test running cowtest == 
$ make qemu-gdb
(10.2s) 
== Test   simple == 
  simple: OK 
== Test   three == 
  three: OK 
== Test   file == 
  file: OK 
== Test usertests == 
$ make qemu-gdb
(129.9s) 
    (Old xv6.out.usertests failure log removed)
== Test   usertests: copyin == 
  usertests: copyin: OK 
== Test   usertests: copyout == 
  usertests: copyout: OK 
== Test   usertests: all tests == 
  usertests: all tests: OK 
== Test time == 
time: OK 
Score: 110/110
```
