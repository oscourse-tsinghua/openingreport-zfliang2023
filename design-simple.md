
### 任务调度器设计方案（简化版）

完整的 `任务调度器` 设计方案见 [design](./design-v4.md)，本方案删减了其中进程、线程切换、系统调用转发等部分功能，旨在 kernel bypass 或 unikernel 条件下，构建低时延服务。

本方案仅定义 `任务调度器` 暴露给软件的接口以及内部行为，将 `Executor` 集成到 `任务调度器` 中，不涉及具体的方案。（若不理解 `Executor`，请参考[资料](https://rust-lang.github.io/async-book/02_execution/01_chapter.html)）

#### 软件接口

<table rules="all" style="table-layout:word-wrap:break-word;word-break:break-all">
  <tr>
    <th>接口</th>
    <th>描述</th>
  </tr>
  <tr>
    <td>reset()</td>
    <td>重置 任务调度器</td>
  </tr>
  <tr>
    <td>lidt(idt: Idt) </td>
    <td>设置中断向量表，其中保存对应的中断处理 TaskRef</td>
  </tr>
  <tr>
    <td>spawn(task_ref: TaskRef)</td>
    <td>创建 Task ，将 Task 添加至就绪队列</td>
  </tr>
  <tr>
    <td>fetch() -> Option&lt;TaskRef&gt;</td>
    <td>取出优先级最高的 Task </td>
  </tr>
  <!-- <tr>
    <td>pass_args(args: Args)</td>
    <td>将系统调用参数传递给 任务调度器</td>
  </tr>
  <tr>
    <td>fetch_args() -> Option&lt;Args&gt;</td>
    <td>从 任务调度器 中取出系统调用参数</td>
  </tr> -->
</table>

#### 寄存器

1. `stask`：用于创建 `Task`
2. `ftask`：用于取出 `Task`
3. `idt`：用于 `lidt` 操作
4. `control`：用于控制

#### 内部行为

内部集成 `Executor`，`Executor` 内部为每个优先级设置一个 `FIFO`，保存 `TaskRef`，在软件调用 `spawn` 或 `fetch` 接口时，从对应的 `FIFO` 中添加或取出 `TaskRef`。


当收到中断信号时，根据中断向量号，从中断向量表中获得对应的中断处理 `TaskRef`，将 `TaskRef` 添加到 `Executor` 优先级最高的 `FIFO` 中。