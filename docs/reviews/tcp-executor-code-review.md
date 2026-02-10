# TCP Executor Code Review（`hcomm/transport/tcp/executor.cpp`）

本文档记录对 `hcomm/transport/tcp/executor.cpp` 的代码审查结论，聚焦并发安全、epoll 事件管理与生命周期控制。

## 范围

- 目标文件：`hcomm/transport/tcp/executor.cpp`
- 相关接口契约：`hcomm/promise/promise.hpp`（`Executor::schedule` 线程安全约束）

## 关键问题

### 1) `schedule()/reschedule()` 与 `loop()` 并发访问队列，违反线程安全契约（高优先级）

`Executor::schedule` 注释声明该方法应当是线程安全的，但当前实现中：

- `schedule()` / `reschedule()` 直接写 `ready_queue_`；
- `loop()` 中同时读取并移动 `ready_queue_`；
- 全程没有互斥锁或无锁并发结构保护。

这会导致数据竞争和未定义行为（UB），在多线程场景下有高风险。

**建议修复：**

- 为 `ready_queue_` 增加互斥保护（或替换为可靠的 MPSC 队列）；
- 明确所有跨线程入口（`schedule`、`wake -> reschedule`）的并发语义。

---

### 2) 同一 FD 的读写事件注册互相覆盖（高优先级）

`registerWaker(fd, events, waker)` 每次将 `reg.interested = events`，并直接以本次 `events` 调用 `epoll_ctl`。

后果：

- 若先注册读再注册写（或反向），后一次会覆盖前一次关注事件；
- 导致一个方向的 readiness 永远收不到。

**建议修复：**

- 在 `IORegistration` 中分别管理读/写 waker；
- `epoll` 关注集合采用合并掩码（例如读写 interest 做 OR）；
- `EPOLL_CTL_MOD` 使用合并后的总掩码，而不是本次单方向掩码。

---

### 3) `stop()` 无法唤醒阻塞中的 `epoll_wait`，可能导致无法及时退出（高优先级）

当前 `stop()` 仅设置 `stop_ = true`，但 `loop()` 可能阻塞在 `epoll_wait(..., -1)`。

如果没有新的 I/O 事件，事件循环不会返回检查 `stop_`，可能出现退出延迟甚至“卡住”现象。

**建议修复：**

- `stop()` 中在设置停止标志后主动触发 `notify()`；
- 析构路径确保可打断阻塞等待并安全退出循环。

---

### 4) `schedule()` 未自动唤醒 loop，跨线程提交任务可能延迟执行（中优先级）

当前 `schedule()` 只是入队，不会触发 `eventfd`。如果 loop 正阻塞在 `epoll_wait` 且无 I/O，新任务可能长时间不执行。

**建议修复：**

- `schedule()/reschedule()` 入队成功后触发 `notify()`；
- 或引入“仅在从空队列变非空时 notify”的优化，减少唤醒开销。

## 建议落地顺序

1. 先修复并发安全（问题 1）和停止唤醒（问题 3），确保稳定性。
2. 修复读写 interest 合并逻辑（问题 2），确保功能完整。
3. 补充调度唤醒策略（问题 4），改善时延与可用性。
4. 增加回归测试：
   - 多线程并发 `schedule` 压测；
   - 同一 fd 同时注册读写并验证均可触发；
   - `stop()` 在空闲 loop 阻塞态下可及时退出。

## 结论

当前实现在单线程、轻负载路径下可能“看起来可用”，但在真实异步网络负载和跨线程调度下存在明显稳定性风险。建议尽快按上述优先级修复，并补齐回归测试。
