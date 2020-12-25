# xv6-tracking
Repository to record the progress for self-learning MIT 6.S081. Out of the respect to MIT for making all the great course resource public, the code implementation will only be kept in a separate private repository.
- [Utilities Lab](#utilities-lab)
    - [Lab Result](#lab-result)

## Utilities Lab
This lab requires to write some basic user space functions utilizing some library functions and system call functions.

Most of the problems in this lab are not very hard to implement. The most interesting one is the `primes.c` program. The tricky part is every process will have two pipes to communicate with (one to communicate with its parent, one to communicate with its child), so how to use the `fork` and `pipe` to create the process? The way I approach this problem is:
- For a process, it will have two sets of file descriptors (corresponding to two pipes). It also keeps a `flag` to indicate which pipes is the one it should use to communicate with its parent.
- Main process will initialize a pipe and set the `flag` to correspond pipe. Then after the `fork`, the child process will get the same memory image as the parent. Since the `flag` will be the same as its parent, it knows which set of file descriptors to use to read the input, and the other one will be the one it uses to communicate with its child. Then it can `fork` to create the child, write values to the child pipe, and `wait` for child to return.
- For a child process:
    - If `read` returns 0, it can `exit`.
    - Otherwise, it will print the value and be responsible for filter out all the multiples of this value. It will initialize a pipe with another file descriptors set to write output to its child. Note here the `flag` will need to be change to correspond pipe.
    - `fork` to create the child, write filtered value to its child pipe, and then `wait` for child.

### Lab Result
```
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
