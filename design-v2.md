### 软硬件结合的共享调度器设计方案  v2 

时间：2023年11月05

核心思想：将中断、异常看作一种消息事件。在用户态发生时，给用户进程和内核分别发送一个消息；在内核态发生时，给内核发送消息。

#### 系统框架：

![arch](./assets/archv2.svg)

内核与用户态的所有任务都是以 `Rust` 协程的形式存在。内核管理进程控制块，为进程控制块实现 `Future`，从而与其他的内核协程形式上相统一。
①：调度到目标进程，通过跳板进入到用户态的共享调度器；
②：共享调度器开始工作，调度用户态协程；
③：用户态正在执行的协程由于中断和异常，执行过程被打断，进入到 `AsyncTrap`，`AsyncTrap` 根据打断的原因，分别向内核和用户进程传递消息；
④：`AsyncTrap` 向用户进程传递消息，并且必要时会保存上一个协程的上下文，`SharedScheduler` 调度下一个就绪协程；
⑤：`AsyncTrap` 向内核发送消息，内核进行相关的处理（唤醒对应的协程）；
同理，若在内核中的协程被打断，也采用相同的处理方式。

#### 任务

##### 内核

内核中的任务分为两类，一类是普通的内核协程，另一类是封装成协程形式的进程。

###### 进程控制块结构

```rust
pub struct Process {
    // The `Stack` points to the stack_top in user address space.

    // The length of `tasks` is the amount of CPU allocated to the `Process`.
    // In the system, Stack is a attribute of CPU, 
    // so we have to bind a stack to a CPU when a task is ready for running.
    pub tasks: Vec<Stack>,
    // The preallocated `stack_poll`` is used to provide stack for task 
    // when Exception or Interrupt is happend.
    // The previous stack is bind to the suspended task in user space,
    // so we need to fetch a stack from the `stack_poll` to execute the next ready task.
    pub stack_pool: Vec<Stack>,
    ......
}
```

封装成协程形式的进程，在被调度时，需要从内核切换到用户态，这个过程的操作将被封装到手动实现的 `Future Trait` 中的 `poll` 函数中。

```rust
impl Future for Process {
    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Self::Output> {
        // jump to the entry of `AsyncTrap`
        // return Poll<Self::Output> so the execution can return to the `loop` function.
        // Then other kernel task can be scheduled.
    }
}
```

由于 `Rust future` 实现是惰性的，创建并不会执行，需要调用 `await` 才会让 `future` 执行。因此 `await` 包含了执行和让权的操作。原本 `await` 的语义是在执行不下去，需要等待事件时，主动让权，这是一种栈空让权，即把栈清空，同时将 CPU 分配给其他的就绪协程。而上述的操作改变了 `await` 关键字的语义。
- 在进程控制块的 `poll` 函数中跳转到 `AsyncTrap` 并从 `AsyncTrap` 进入用户态的操作，相当于把 `await` 的语义改变了，变成了带栈让权，当前的栈以及寄存器线程被保存，而 CPU 的使用权主动让出给用户进程，进入到用户态。
- 在 `poll` 函数返回后，CPU 的执行流将会回到 `loop` 函数，`fetch` 到下一个进程时，即为进程切换。符合 `await` 原本的语义。

##### 用户进程

###### 用户进程任务控制块

用户进程中的任务全部由协程组成。但会根据执行情况，与栈进行绑定。与栈绑定的协程即为线程，没有绑定的则为协程。

```rust
pub struct TaskStruct {
    // When a task is interrupted by Exception or Interrupt,
    // the current stack will be stored in the Context struct. 
    // so the task will be a `Thread`.
    pub context: Option<Context>,
    ......
}
```

与内核中的 `Process` 相似，也为 `TaskStruct` 实现 `Future trait`。

```rust
impl Future for TaskStruct {
    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.context.is_some() {
            // If the `context` is not none, 
            // then we must restore registers from the context,
            // so the previous execution can be continue. 
            // The previous is usually be interrupted by Exception or Interrupt.
        } else {
            // If the `context` is none, the task can run directly.
        }
    }
}
```

当 `context` 不为空时，意味着这个协程之前是被异常或者中断打断的，这时需要从 `context` 中恢复上下文，才能在被打断的地方继续执行，这个过程中会涉及到 `stack` 切换，当前正在使用的 `stack` 是可以直接切换的，并且不需要保存上下文，因为此时是处于可以让出的点。被切换的当前的 `stack` 将会回到 `stack_pool` 中，供发生下一次异常或者中断时使用。


#### SharedScheduler + AsyncTrap 模块

这两个模块是整个系统的核心。
将这两个模块放在一起，一方面是因为这两个模块都需要在内核与用户进程之间共享，另一方面是因为这两个模块之间存在紧密的联系。

##### AsyncTrap 子模块

功能：上下文切换 + 消息传递（通过寄存器，向处于内核态/用户态的 `SharedScheduler` 传递信息。）

详细设计方案见 [async-trap](./async-trap-design.md)

问题：
1. 控制流转移的分类维度
2. 内核切换到用户态时，对应的内核协程处于的状态
3. 进程切换需要经过内核吗，还是说仅仅通过 `AsyncTrap` 即可完成进程切换

##### SharedScheduler 子模块

功能：协程调度

结合 [emabssy](https://embassy.dev/dev/index.html) 

#### 实施方案

##### 第一阶段

在软件上，将上述的系统实现。

##### 第二阶段

分析其中 `AsyncTrap` 可能在硬件上能够实现的部分功能，在 qemu 中实现。

##### 第三阶段

参考第二阶段，在 FPGA 中实现。


