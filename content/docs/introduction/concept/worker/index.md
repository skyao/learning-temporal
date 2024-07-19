---
title: "worker的概念"
linkTitle: "worker"
weight: 40
date: 2021-01-29
description: >
  Temporal 中 worker 的概念
---

> https://docs.temporal.io/workers

# 什么是Temporal Worker？

Temporal Task Queues 和 worker 进程之间存在紧密的耦合关系。

## 什么是 worker？

在日常对话中，"worker"一词用来指[ worker 程序](https://docs.temporal.io/workers#worker-program)、[worker 过程](https://docs.temporal.io/workers#worker-process)或 [worker 实体](https://docs.temporal.io/workers#worker-entity)。 Temporal 文档旨在明确并区分它们。

## 什么是任务？

任务是 Worker 执行特定 [工作流执行](https://docs.temporal.io/workflows#workflow-execution) 或 [活动执行](https://docs.temporal.io/activities#activity-execution) 所需的上下文。

有两种类型的任务：

- [活动任务](https://docs.temporal.io/workers#activity-task)
- [工作流任务](https://docs.temporal.io/workers#workflow-task)

## 什么是 worker 程序？

Worker 程序是定义 Worker 过程约束的静态代码，使用 Temporal SDK 的 API 开发。

- [如何使用 Go SDK 运行开发 Worker](https://docs.temporal.io/dev-guide/go/foundations#develop-worker)
- [如何使用 Java SDK 运行开发 Worker](https://docs.temporal.io/dev-guide/java/foundations#run-a-dev-worker)
- [如何使用 PHP SDK 运行开发 Worker](https://docs.temporal.io/dev-guide/php/foundations#run-a-dev-worker)
- [如何使用 Python SDK 运行开发 Worker](https://docs.temporal.io/dev-guide/python/foundations#run-a-dev-worker)
- [如何使用 TypeScript SDK 运行开发 Worker](https://docs.temporal.io/dev-guide/typescript/foundations#run-a-dev-worker)
- [如何使用 Go SDK 运行 Temporal Cloud Worker](https://docs.temporal.io/dev-guide/go/foundations#run-a-temporal-cloud-worker)
- [如何使用 TypeScript SDK 运行 Temporal Cloud Worker](https://docs.temporal.io/dev-guide/typescript/foundations#run-a-temporal-cloud-worker)

## 什么是 Worker Entity？

Worker Entity 是 Worker Process 中监听特定任务队列的单个 Worker。

Worker Entity 监听并轮询单个任务队列。 Worker Entiry 包含一个 Workflow Worker 和/或一个 Activity Worker，它们分别在工作流执行和活动执行方面取得进展。

**Worker 能否处理比其缓存大小或支持的线程数更多的工作流执行？**

是的，它可以。 不过，代价是增加了延迟。

Worker 是无状态的，因此任何处于阻塞状态的工作流执行都可以安全地从 Worker 中移除。 稍后，当需要时，它可以在相同或不同的 Worker 上复活（以外部事件的形式）。 因此，单个 Worker 可以处理数百万个开放的工作流执行，前提是它能处理更新率，并且不需要担心稍高的延迟。

**操作指南：**

- [如何调整 worker](https://docs.temporal.io/dev-guide/worker-performance)

## 什么是 worker 程序？

Worker进程和 Temporal Server 的组件图

![Component diagram of a Worker Process and the Temporal Server](https://docs.temporal.io/diagrams/worker-and-server-component.svg)

Worker 进程负责轮询 [任务队列](https://docs.temporal.io/workers#task-queue)、出列[任务](https://docs.temporal.io/workers#task)、执行代码以响应任务，并将结果响应给 [Temporal Cluster](https://docs.temporal.io/clusters)。

更正式地说，worker进程是实现任务队列协议和任务执行协议的任何进程。

- 如果一个工作进程执行了工作流任务队列协议，并执行了工作流任务执行协议以推进工作流执行，那么该进程就是工作流工作进程。 工作流工作进程可以监听任意数量的工作流任务队列，并执行任意数量的工作流任务。
- 如果一个工作进程执行了活动任务队列协议，并执行了活动任务处理协议以在活动执行中取得进展，那么该进程就是活动工作进程。 活动工作进程可以监听任意数量的活动任务队列，并执行任意数量的活动任务。

**工作进程位于 Temporal Cluster 的外部。** Temporal 应用程序开发人员负责开发 [Worker 程序](https://docs.temporal.io/workers#worker-program) 和运维 Worker Processes。 换句话说，[Temporal Cluster](https://docs.temporal.io/clusters)（包括 Temporal Cloud）不会在 Temporal Cluster 机器上执行你的任何代码（工作流和活动定义）。 群组只负责协调 [状态转换](https://docs.temporal.io/workflows#state-transition)，并向下一个可用的 [worker 实体](https://docs.temporal.io/workers#worker-entity)提供任务。

虽然事件历史记录中传输的数据在默认情况下是[通过 mTLS 保护的](https://docs.temporal.io/self-hosted-guide/security#encryption-in-transit-with-mtls)，但在 Temporal Cluster 中仍可静态读取。

为了解决这个问题，Temporal SDK 提供了[数据转换器 API](https://docs.temporal.io/dataconversion)，你可以用它来定制进出 Worker Entity 的数据的序列化，其最终效果是保证 Temporal Cluster 无法读取敏感的业务数据。

在我们的许多教程中，我们都向您展示了如何在同一台机器上同时运行 Temporal Cluster 和 Worker 以进行本地开发。 但是，生产级 Temporal 应用程序通常具有工作进程_舰队_，所有进程都运行在 Temporal Cluster 外部的主机上。 Temporal 应用程序可以根据需要具有任意数量的工作进程。

Worker 进程既可以是工作流工作进程，也可以是活动工作进程。 许多 SDK 支持在单个 Worker 进程中拥有多个 Worker 实体的功能。 (Worker 实体的创建和管理因 SDK 而异）。 单个 Worker 实体只能侦听单个 Task Queue。 但是，如果一个 worker 进程有多个 worker 实体，则该 worker 进程可能正在侦听多个任务队列。

![Worker Processes、Task Queues、Tasks 的实体关系图（元模型）](https://docs.temporal.io/assets/images/worker-and-server-entity-relationship-a2373ecadf6ee25541a1e3e3ffe1b57f.svg)

执行活动任务的 worker 进程必须有权访问执行活动定义中定义的操作所需的任何资源，例如：

- 外部 API 调用的网络访问。
- 用于基础设施供应的凭证。
- 用于机器学习工具的专用 GPU。

Temporal Cluster 本身有用于系统工作流执行的 [内部 worker](https://temporal.io/blog/workflow-engine-principles/#system-workflows-1910)。 但是，这些内部 worker 对开发人员不可见。

## 什么是工作流任务？

工作流任务是包含工作流执行进度所需的上下文的任务。

- 每次记录到可能影响工作流状态的新的外部事件时，包含该事件的工作流任务就会被添加到任务队列中，然后由工作流 worker 接收。
- 在处理完新事件后，工作流任务将以 [命令](https://docs.temporal.io/workflows#command)列表的形式完成。
- 工作流任务的处理通常非常快，与工作流调用操作的持续时间无关。

## 什么是工作流任务执行？

工作流任务执行发生在 [Worker](https://docs.temporal.io/workers#worker-entity) 拿起一个 [Workflow Task](https://docs.temporal.io/workers#workflow-task) 并用它来推进 [Workflow Definition](https://docs.temporal.io/workflows#workflow-definition) （也称为工作流函数）的执行时。

## 什么是活动任务？

活动任务包含执行[活动任务执行](https://docs.temporal.io/workers#activity-task-execution)所需的上下文。 活动任务主要代表活动任务计划事件，其中包含执行活动功能所需的数据。

如果正在传递 "心跳" 数据，活动任务也将包含最新的 "心跳" 详细信息。

## 什么是活动任务执行？

当 [Worker](https://docs.temporal.io/workers#worker-entity) 使用 [Activity Task](https://docs.temporal.io/workers#activity-task) 提供的上下文并执行 [Activity Definition](https://docs.temporal.io/activities#activity-definition)（也称为活动函数）时，就会发生活动任务执行。

[活动任务计划事件](https://docs.temporal.io/references/events#activitytaskscheduled) 与 Temporal Cluster 将活动任务放入任务队列的时间相对应。

[ActivityTaskStarted 事件](https://docs.temporal.io/references/events#activitytaskstarted) 与 Worker 从任务队列中提取活动任务的时间相对应。

无论是 [ActivityTaskCompleted](https://docs.temporal.io/references/events#activitytaskcompleted)，还是其他已关闭的活动任务事件，都与 worker 何时让渡给（yield） Temporal Cluster 相对应。

安排活动执行的 API 提供了 "effectively once"的体验，尽管要成功完成一项活动可能需要执行多个活动任务。

一旦活动任务执行完毕，Worker 就会通过特定事件响应集群：

- ActivityTaskCanceled
- ActivityTaskCanceled
- ActivityTaskFailed
- ActivityTaskTerminated
- ActivityTaskTimedOut

## 什么是任务队列？

任务队列是一个轻量级的动态分配队列，一个或多个[Worker Entities](https://docs.temporal.io/workers#worker-entity) 可对[任务](https://docs.temporal.io/workers#task) 进行轮询。

任务队列有两种类型：活动任务队列和工作流任务队列。

任务队列组件

![Task Queue component](https://docs.temporal.io/diagrams/task-queue.svg)

任务队列是非常轻量级的组件。 任务队列不需要显式注册，而是在工作流执行，或活动产生，或工作进程订阅时按需创建。 创建任务队列时，工作流任务队列和活动任务队列会以相同的名称创建。 对 Temporal 应用程序可使用的任务队列数量或 Temporal Cluster 可维护的任务队列数量没有限制。

Worker 通过同步 RPC 轮询任务队列中的任务。 此实现具有以下几个优点：

- Worker 进程只有在有剩余能力时才会轮询信息，以避免自身超负荷。
- 实际上，任务队列支持跨多个 worker 进程的负载均衡。
- 任务队列可以实现我们所说的[任务路由](https://docs.temporal.io/workers#task-routing)，即把特定任务路由到特定的 worker 进程，甚至是特定的进程。
- 任务队列支持服务器端限流，这使您能够限制 worker 进程池的任务分派率，同时在出现峰值时仍支持以更高的速率分派任务。
- 当所有的 worker 进程都宕机时，消息只会在任务队列中持续存在，等待 worker 进程恢复。
- Worker 进程不需要通过 DNS 或任何其他网络发现机制来通告自己。
- Worker 进程不需要开放任何端口，因此更加安全。

所有监听特定任务队列的 Worker 必须具有相同的活动和/或工作流注册。 唯一的例外是在服务器升级期间，在二进制文件推出时，注册暂时未对齐是可以的。

#### 在哪里设置任务队列

开发人员可以在四个地方设置任务队列的名称。

1. 生成工作流执行时，必须设置任务队列：

   - [如何使用 tctl 启动工作流执行](https://docs.temporal.io/tctl-v1/workflow#start)

   - [如何使用 Go SDK 启动工作流执行](https://docs.temporal.io/dev-guide/go/foundations#start-workflow-execution)

   - [如何使用 Java SDK 启动工作流执行](https://docs.temporal.io/dev-guide/java/foundations#start-workflow-execution)

   - [如何使用 PHP SDK 启动工作流执行](https://docs.temporal.io/dev-guide/php/foundations#start-workflow-execution)

   - [如何使用 Python SDK 启动工作流执行](https://docs.temporal.io/dev-guide/python/foundations#start-workflow-execution)

   - [如何使用 TypeScript SDK 启动工作流执行](https://docs.temporal.io/dev-guide/typescript/foundations#start-workflow-execution)


2. 创建 worker 实体和运行 worker 进程时必须设置任务队列名称：

   - [如何使用 Go SDK 运行开发 Worker](https://docs.temporal.io/dev-guide/go/foundations#develop-worker)

   - [如何使用 Java SDK 运行开发 Worker](https://docs.temporal.io/dev-guide/java/foundations#run-a-dev-worker)

   - [如何使用 PHP SDK 运行开发 Worker](https://docs.temporal.io/dev-guide/php/foundations#run-a-dev-worker)

   - [如何使用 Python SDK 运行开发 Worker](https://docs.temporal.io/dev-guide/python/foundations#run-a-dev-worker)

   - [如何使用 TypeScript SDK 运行开发 Worker](https://docs.temporal.io/dev-guide/typescript/foundations#run-a-dev-worker)

   - [如何使用 Go SDK 运行 Temporal Cloud Worker](https://docs.temporal.io/dev-guide/go/foundations#run-a-temporal-cloud-worker)

   - [如何使用 TypeScript SDK 运行 Temporal Cloud Worker](https://docs.temporal.io/dev-guide/typescript/foundations#run-a-temporal-cloud-worker)


​	请注意，监听同一任务队列名称的所有 Worker 实体必须注册为处理完全相同的工作流类型和活动类型。

​	如果 worker 实体为它不知道的工作流类型或活动类型轮询任务，则该任务会失败。 但是，任务失败不会导致相关工作流执行失败。

3. 在派生活动执行时，可以提供 Task Queue 名称：
   这是可选的。 如果没有提供任务队列名称，则活动执行会继承其工作流执行的任务队列名称。

   - [如何使用 Go SDK 启动活动执行](https://docs.temporal.io/dev-guide/go/foundations#activity-execution)

   - [如何使用 Java SDK 启动活动执行](https://docs.temporal.io/dev-guide/java/foundations#activity-execution)

   - [如何使用 PHP SDK 启动活动执行](https://docs.temporal.io/dev-guide/php/foundations#activity-execution)

   - [如何使用 Python SDK 启动活动执行](https://docs.temporal.io/dev-guide/python/foundations#activity-execution)

   - [如何使用 TypeScript SDK 启动活动执行](https://docs.temporal.io/dev-guide/typescript/foundations#activity-execution)

4. 生成子工作流执行时可提供任务队列名称：

   这是可选的。 如果没有提供任务队列名称，子工作流执行程序将继承父工作流执行程序的任务队列名称。

   - [如何使用 Go SDK 启动子工作流执行](https://docs.temporal.io/dev-guide/go/features#child-workflows)

   - [如何使用 Java SDK 启动子工作流执行](https://docs.temporal.io/dev-guide/java/features#child-workflows)

   - [如何使用 PHP SDK 启动子工作流执行](https://docs.temporal.io/dev-guide/php/features#child-workflows)

   - [如何使用 Python SDK 启动子工作流执行](https://docs.temporal.io/dev-guide/python/features#child-workflows)

   - [如何使用 TypeScript SDK 启动子工作流执行](https://docs.temporal.io/dev-guide/typescript/features#child-workflows)


#### 任务排序

任务队列可通过添加分区进行扩展。 [默认](https://docs.temporal.io/references/dynamic-configuration#service-level-rps-limits) 分区数为 4。

具有多个分区的任务队列没有任何排序保证。 一旦有已写入磁盘的任务积压，可以立即调度的任务（“同步匹配”）将在积压工作中的任务（“异步匹配”）之前交付。 此方法可优化吞吐量。

具有单个分区的任务队列几乎总是先进先出，只有极少数例外情况。 不过，使用单个分区只能满足中低吞吐量的使用要求。

注意

本节涉及单个任务的排序，不适用于单个工作流执行中工作流执行、活动执行或 [事件](https://docs.temporal.io/workflows#event)的排序。 工作流执行中的事件一旦被写入该工作流执行的 [历史记录](https://docs.temporal.io/workflows#event-history)，其顺序保证保持不变。

## 什么是粘性执行？

粘性执行是指 worker 实体将工作流缓存在内存中，并创建专用任务队列进行监听。

在 worker 实体完成工作流执行的工作流任务链中的第一个工作流任务后，就会发生 "粘性执行"。

Worker 实体会在内存中缓存工作流，并开始轮询专用任务队列中包含更新的工作流任务，而不是整个事件历史。

如果 worker 实体没有在适当时间内从专用任务队列中拾取工作流任务，群集将恢复在原始任务队列中调度工作流任务。 然后，另一个 worker 实体可以恢复工作流执行，并为未来的工作流任务设置自己的 "粘性执行"。

- [如何在 Go 中为 Worker Entity 设置 "StickyScheduleToStartTimeout"](https://docs.temporal.io/dev-guide/go/foundations#stickyscheduletostarttimeout)

粘性执行是 Temporal Platform 的默认行为。

## 什么是任务路由？

任务路由是指任务队列与一个或多个 Worker 配对，主要用于执行活动任务。

这也可能意味着采用多个任务队列，每个队列与一个 worker 进程配对。

任务路由有许多适用的用例。

一些 SDK 提供了[会话 API](https://docs.temporal.io/workers#worker-session)，它提供了一种直接的方法来确保活动任务与相同的 Worker 一起执行，而无需手动指定任务队列名称。 它还包括并发会话限制和 worker 故障检测等功能。

### 流量控制

从任务队列消费的 Worker 只有在有可用容量时才会请求活动任务，因此不会因请求高峰而超载。 如果活动任务的创建速度快于 worker 的处理速度，它们就会积压在任务队列中。

### 限流

每个活动任务执行者轮询和处理活动任务的速度可根据每个活动任务执行者进行配置。 即使有剩余能力， worker 也不会超过这一比例。 还支持全局任务队列速率限制。 此限制适用于给定任务队列的所有 Worker。 它通常用于限制活动调用的下游服务的负载。

### 特定环境

在某些情况下，您可能需要在专用环境中执行活动。 若要将活动任务发送到此环境，请使用专用的任务队列。

#### 将活动任务路由到特定主机

在某些用例中，如文件处理或机器学习模型训练，活动任务必须路由到特定的 worker 进程或 worker 实体。

例如，假设您有一个包含以下三个独立活动的工作流：

- 下载文件
- 以某种方式处理文件。
- 将文件上传到其他位置。

下载文件的第一个活动可以发生在任何主机上的任何 Worker 上。 但是，第二个和第三个 Activity 必须由 Worker 在第一个 Activity 下载文件的同一主机上执行。

在现实生活中，您可能会有许多 worker 进程分布在许多主机上。 您需要开发 Temporal 应用程序，以便在需要时将任务路由到特定的 worker 进程。

代码示例:

- [Go 文件处理示例](https://github.com/temporalio/samples-go/tree/main/fileprocessing)
- [Java 文件处理示例](https://github.com/temporalio/samples-java/tree/main/core/src/main/java/io/temporal/samples/fileprocessing)
- [PHP 文件处理示例](https://github.com/temporalio/samples-php/tree/master/app/src/FileProcessing)

#### 将活动任务路由到特定进程

有些活动会加载大型数据集，并在过程中对其进行缓存。 依赖于这些数据集的活动应路由到同一进程。

在这种情况下，所涉及的每个 worker 进程都将存在一个唯一的任务队列。

#### 具有不同能力的 worker

某些 Worker 可能存在于 GPU 机箱上，而不是非 GPU 机箱上。 在这种情况下，每种类型的盒子都有自己的任务队列，工作流可以选择其中一个发送活动任务。

### 多重优先事项

如果您的用例涉及多个优先级，您可以为每个优先级创建一个任务队列，每个优先级创建一个 worker 池。

### 版本管理

任务路由是实现代码版本控制的最简单方法。

如果您有新的向后不兼容的活动定义，请首先使用不同的任务队列。

### 什么是 Worker Session？

Worker Session 是某些 SDK 提供的一项功能，它为 [Task Routing](https://docs.temporal.io/workers#task-routing)提供了直接的 API，以确保活动任务与同一个 Worker 一起执行，而无需手动指定任务队列名称。 它还包括并发会话限制和 worker 故障检测等功能。

- [如何使用 worker 会话](

