---
layout: post
title: Mutexes vs Semaphores in the Linux kernel
comments: True
---

"What is the difference between a mutex and a semaphore?". This topic (and this question is particular) represents the holy grail of most coding interviews. It is often used as a "weed-out" question if the position in question is related to embedded software development. Unfortunately, most operating system texts do not do a great job of explaining the subtle difference between these OS primitives.

There are plenty of articles around the www that attempt to explain the difference. But, by far, the best explanation of
this topic comes from [Niall Cooling from Sticky Bits](https://blog.feabhas.com/2009/09/mutex-vs-semaphores-%E2%80%93-part-1-semaphores/)
you want to cement your understanding on the subject. The jist of this is that :

* mutex is (as the name specifies) used to provide mutual exclusion when accessing a shared
resource and a semaphore (atleast the way it was intended to be used) is used as a signalling mechanism between tasks.
* One other critical difference has to do with the concept of **ownership** i.e. a mutex can only be unlocked by the task that
originally locked it (or "owns" it).

I wanted to further explore this difference by studying the mutex and semaphore implementations in the Linux kernel. The
easiest way to do these experiments would be using [kernel loadable
modules](http://tldp.org/HOWTO/Module-HOWTO/x73.html). In order to enforce strict mutex semantics (eg. check for
ownership etc.), the kernel should be compiled with the CONFIG_DEBUG_MUTEXES option enabled.

Mutex testing
-------------

I created a simple [character driver](http://tjworld.net/books/ldd3/#CharDrivers). The driver source can be found
[here](https://github.com/zodiac0prg/linux_kernel_test/blob/master/mutex_demo/mutex_demo.c). As a part of the "init"
function of the module, I go ahead and initialize this driver as a character device driver, register the character
driver fops (open, close, read, write), create a dev file called `/dev/mymutex0` and also (more importantly) initializes
a mutex using `mutex_init`. The read and the write functions are where most of the "interesting" stuff is happening. The
read function attempts to "acquire" the mutex and the write attempts to "release" the mutex.

I tested this as follows:

From process 1:
```shell
$ insmod mutex_demo.ko
$ ls /dev/my*ymutex0
$ cat /dev/mymutex0
```

From process 2:
```shell
$ echo "1" > /dev/mymutex0

------------[ cut here ]------------
WARNING: CPU: 0 PID: 120 at kernel/locking/mutex-debug.c:80 debug_mutex_unlock+0x130/0x140()
DEBUG_LOCKS_WARN_ON(lock->owner != current)
Modules linked in: mutex_demo(O) ipv6
CPU: 0 PID: 120 Comm: sh Tainted: G           O    4.2.8 #9
Call trace:
[<ffffffc000089c58>] dump_backtrace+0x0/0x110
[<ffffffc000089d7c>] show_stack+0x14/0x20
[<ffffffc000570c68>] dump_stack+0x84/0xc4
[<ffffffc0000962a8>] warn_slowpath_common+0x98/0xd0
[<ffffffc000096320>] warn_slowpath_fmt+0x40/0x48
[<ffffffc0000caca8>] debug_mutex_unlock+0x130/0x140
[<ffffffc000574854>] __mutex_unlock_slowpath+0x8c/0x158
[<ffffffc00057492c>] mutex_unlock+0xc/0x18
[<ffffffbffc0560a8>] my_write+0x38/0xe0 [mutex_demo]
[<ffffffc00014c264>] __vfs_write+0x1c/0x108
[<ffffffc00014cbf8>] vfs_write+0x90/0x188
[<ffffffc00014d5bc>] SyS_write+0x44/0xa0
---[ end trace 09bf3f43eda93640 ]---
``` 

Semaphore testing
-----------------

The [kernel module](https://github.com/zodiac0prg/linux_kernel_test/blob/master/bin_sem_demo/bin_sem.c) for semaphore
testing is almost identical to that of the mutex example. The only difference is that now instead of initializing a
mutex, we instead initialize the semaphore with initial count as 1 (making this a binary semaphore). The "read" on
/dev/mysem0 perfoms a "down" on the semaphore and the "write" operation performs an "up" on the semaphore.

With this setup, we can now perform a bunch of experiments to understand different scenarios.

#### Scenario 1

From process 1:
```shell
$ cat /dev/mysem0
```

From process 2:
```shell
$ echo "1" > /dev/mysem0
```
Notice that it did not matter which process did the "up" and which process 
did the "down" on the semaphore, further emphasizing that there is no concept of ownership when it comes to semaphore
"locking"/"unlocking"

#### Scenario 2

From process 1:
```shell
$ cat /dev/mysem0
```

From process 2:
```shell
$ cat /dev/mysem0
<< Blocks! >>
```
#### Scenario 3

From process 1:
```shell
$ cat /dev/mysem0
```

From process 2:
```shell
$ echo "1" > /dev/mysem0
<< Value of sem is 1 >>
$ echo "1" > /dev/mysem0
<< Value of sem is 2 >>
$ cat /dev/mysem0
<< Value of sem is 1 >>
$ cat /dev/mysem0
<< Value of sem is 0 >>
$ cat /dev/mysem0
<< Blocks! >>
```

Obviously, there is a **lot** more to discuss when it comes to mutex and semaphore design in the Linux kernel. But this
should help clarify fundamental differences between these two OS primitives.

Cheers! 
