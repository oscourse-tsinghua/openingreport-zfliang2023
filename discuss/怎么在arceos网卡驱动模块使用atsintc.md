---
参加人: xy、zfl
时间: 2024.04.16
主题: 关于在 arceos 中仅在网卡驱动模块中使用 atsintc 的讨论
---

### 问题

开始的预期：希望把 arceos 底层提供给应用的服务都改造成异步的形式，但由于系统调用直接以函数调用进行，导致执行流没有统一的出入口，因此改造无法进行下去

尝试把修改范围局限在网卡驱动上

#### 网卡驱动对上层协议栈需要的接口

```rust
impl NetDriverOps for AxiEth {

    fn receive(&mut self) -> DevResult<NetBufPtr> {}

    fn transmit(&mut self, tx_buf: NetBufPtr) -> DevResult {}

}
```

底层驱动需要实现 `receive` 和 `transmit` 这两个接口，smoltcp 协议栈内部调用这两个接口。

#### 困难

在线程内部创建协程，按照 ATSINTC 的方式进行，开销只会增大

1. 创建协程的开销
2. 发送/接收协程执行阻塞后，中断控制器唤醒后，由于没有在 arceos 的调度层面进行修改，没有合适的时机让恢复成就绪的协程继续运行
   
#### 解决方案

创建一个线程，独占 CPU 运行异步网卡驱动，而其他核则正常运行。当其他核的任务需要通过网卡向外发送、接收数据时，在 `receive` 和 `transmit` 接口中只需要提交缓冲区即可，真正的发送/接收由独占 CPU 的异步网卡驱动进行。
