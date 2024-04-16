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

    fn receive(&mut self) -> DevResult<NetBufPtr> {
        if !self.can_receive() {
            return Err(DevError::Again);
        }
        if let Some(transfer) = self.rx_transfers.pop_front() {
            let buf = transfer.wait().map_err(|_| panic!("Unexpected error"))?;
            let slice = vec![0u8; RX_BUFFER_SIZE].into_boxed_slice();
            let rx_buf = boxbuf_to_bufptr(slice);
            match self.dma.rx_submit(rx_buf) {
                Ok(transfer) => self.rx_transfers.push_back(transfer),
                Err(err) => panic!("Unexpected err: {:?}", err),
            };
            Ok(NetBufPtr::from(buf))
        } else {
            // RX queue is empty, receive from AxiNIC.
            let slice = vec![0u8; RX_BUFFER_SIZE].into_boxed_slice();
            let rx_buf = boxbuf_to_bufptr(slice);
            let completed_buf = self.dma.rx_submit(rx_buf).map_err(|_| panic!("Unexpected error"))?.wait().unwrap();
            let slice = vec![0u8; RX_BUFFER_SIZE].into_boxed_slice();
            let rx_buf = boxbuf_to_bufptr(slice);
            match self.dma.rx_submit(rx_buf) {
                Ok(transfer) => self.rx_transfers.push_back(transfer),
                Err(err) => panic!("Unexpected err: {:?}", err),
            };
            Ok(NetBufPtr::from(completed_buf))
        }
    }

    fn transmit(&mut self, tx_buf: NetBufPtr) -> DevResult {
        let tx_buf = netbuf_to_buf(tx_buf)?;
        match self.dma.tx_submit(tx_buf) {
            Ok(transfer) => {
                self.tx_transfers.push_back(transfer);
                Ok(())
            },
            Err(err) => panic!("Unexpected err: {:?}", err),
        }
    }

}
```

底层驱动需要实现 `receive` 和 `transmit` 这两个接口，smoltcp 协议栈内部调用这两个接口。

#### 困难

在线程内部创建协程，按照 ATSINTC 的方式进行，开销只会增大

1. 创建协程的开销
2. 发送/接收协程执行阻塞后，中断控制器唤醒后，由于没有在 arceos 的调度层面进行修改，没有合适的时机让恢复成就绪的协程继续运行
   
#### 可能的解决方案

中断控制器的队列可以不局限于存放任务，也可以存放缓冲区的指针，因为 DMA 一次事务必须要等待传输完成后才能释放掉缓冲区的所有权，传输过程中这个缓冲区不能使用，目前对这个缓冲区的管理是一直等 DMA 传输完成，这里存在可以利用 ATSINTC 的空间


