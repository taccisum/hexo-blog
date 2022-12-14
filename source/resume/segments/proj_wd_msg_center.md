### `【业务中台】六一 & 豌豆思维消息中心`（2021.06 - 2022.04）

消息中心提供了基础的短信、推送、邮件等消息发送能力，以及防打扰策略，定时推送、批量推送等能力，方便各业务线统一接入

**职责**：后端 TL，负责对接、拆解产品需求，分配任务；负责规划相关技术架构演进

**背景**：由于集团多业务线并行（小灯塔、豌豆益智、画啦啦、咕比 AI 等等）的现状，多条业务线**重复开发**业务消息推送功能，带来了许多不必要的工作量，拖慢了业务节奏。因此规划消息中心，统一接入阿里云/腾讯云/创蓝/极光等供应商

**技术难点**：
  - 群发消息**体量不等，导致资源占用不均衡**（FIFO），用户体验差，影响运营节奏。为了解决此问题，采用**类多级反馈队列**（MLFQ）的设计，使得不同体量的消息在发送时会动态调整其优先级
  - 群发消息时每次批量拉取消息在内存中处理，整个过程可能会**占用大量的内存**，当消息量过大时会对机器配置提出较高的要求。采用 **Reactor 架构**改为流式处理解决此问题
  - 群发消息 - 群发消息时**生产和消费速率不均**，有时会导致大量消息积压在内存，对系统性能产生影响，有时甚至可能导致服务崩溃；最终采用 **Reactor BackPressure** 机制解决此问题

**成绩**：上线后三个月，承接了集团内 **80%** 以上的消息推送，消息日发送量达**千万级**，最高**百万级**单批次发送量，大大减轻了业务研发负担

**技术栈**：SpringCloud, ProjectReactor, RabbitMQ, XXL-Job, Redis, MySQL