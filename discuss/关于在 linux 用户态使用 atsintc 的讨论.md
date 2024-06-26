---
参加人: xy、zfl
时间: 2024.04.08
主题: 关于在 linux 用户态使用 atsintc 的讨论
---

### 讨论在 linux 用户态的使用 atsintc

1. IPC 的发送方发送中断可以通过硬连线或者写 IO 端口（软件写 IO 端口发起中断）
2. 中断的接受方，存在一个协程一直处于阻塞状态，

#### 设计关键：协程标识层次化，包含进程的地址空间信息

层次划分（尽可能减少访存操作）
1. 进程是否在运行
2. 进程内阻塞的协程是否在控制器队列中

进程的信息在控制器里

是否可能在从上一个进程切换到下一个进程时，不经过内核，直接从 A -> B？
直接从A进程的内核态切换到B进程的内核态？

-------------------

中断请求方：目前仅支持外设；发中断请求仅涉及中断控制器内的操作，未来软件发送中断时可能有I/O端口访问，不涉及内存访问；

中断接收方：目前仅支持内核；中断响应的作为仅是唤醒对应协程，这个操作仅在中断控制器中完成，也不涉及内存访问；


- 未来会把一部分放不下的协程放在内存中；
- 收发方的标识：协程标识有层次结构，能表示所在进程信息；如果进程状态不在控制器有描述，则由高特权级实体处理；目标是，接收不涉及内存访问；

协程切换：依据切换前后两个协程的地址空间，判断是否需要当前进程主动让权；

- 协程就绪队列：取出最高优先级协程标识；就绪位图？
- 进程内的协程切换：当前协程标识的变化体现在函数“task.clone().poll()”；先保存让权协程到控制器的等待队列，然后返回调度器中的循环，再取下一个就绪协程，继续运行；
- 地址空间切换：目前的思路是，需要先切特权级，然后切换内核的地址空间，再切到下一个地址空间，最后换特权级回用户态；也放后续可以改成不用进入内核进程的地址空间。