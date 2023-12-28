
### Process Switching

#### How to switch to Process from Kernel?

Once the `Process` in kernel `Executor` has been polled, It should return pending. Then the controller will check Whether to switch to another address process.


#### What are the criteria for switching process？

The kernel `Executor` priority.


#### What information is needed for the switching process?

1. The address space token of target process.
2. The `Executor` pointer of target process.

The information is passed by registers. When sending message, the message is generated according to registers.

When a process(kernel task) has been polled, it will pass `address space token` and `Executor pointer` in `a0`, `a1` registers;

#### How to initialize the first task in user process?

1. Transmute the traditional `main` function into an `async` function.
2. When parsing the `elf` of user application, 
   1. Applying for memory of the `Executor` and initializing it.
   2. Applying for memory of the `Heap` and initializing it.
3. When the control flow is in user process, the `Executor` has no `Task`, but it's state is `Ready`, so it will try to initialize the `main task`.








