
```
Thread 16 "ms_dispatch" hit Breakpoint 1, Objecter::ms_dispatch (this=0x7f4455f49000, m=0x7f4435604540) at ./src/osdc/Objecter.cc:991
991	./src/osdc/Objecter.cc: No such file or directory.
(gdb) bt
#0  Objecter::ms_dispatch (this=0x7f4455f49000, m=0x7f4435604540) at ./src/osdc/Objecter.cc:991
#1  0x00007f444d08c008 in Dispatcher::ms_dispatch2 (m=..., this=0x7f4455f49008) at ./src/msg/Dispatcher.h:124
#2  Messenger::ms_deliver_dispatch (m=..., this=0x7f4455f3f800) at ./src/msg/Messenger.h:703
#3  DispatchQueue::entry (this=0x7f4455f3fb28) at ./src/msg/DispatchQueue.cc:199
#4  0x00007f444d1278ad in DispatchQueue::DispatchThread::entry (this=<optimized out>) at ./src/msg/DispatchQueue.h:101
#5  0x00007f44569c6fa3 in start_thread (arg=<optimized out>) at pthread_create.c:486
#6  0x00007f44568e406f in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
```
