---
title: "中文翻译版"
linkTitle: "中文翻译版"
weight: 20
date: 2023-12-28
description: >
  什么是 Worker 版本控制？中文翻译版
---

> 原文地址：https://docs.temporal.io/workers#worker-versioning

{{% alert title=“支持、稳定性和依赖项信息” color=“info” %}}

- 在 Temporal Server 版本 [1.21.0](https://github.com/temporalio/temporal/releases/tag/v1.21.0) 中引入
- 在 CLI 版本 [0.10.0](https://github.com/temporalio/cli/releases/tag/v0.10.0) 中可用
- 在 [Go SDK](https://docs.temporal.io/dev-guide/go/versioning#worker-versioning)版本 [1.23.0](https://github.com/temporalio/sdk-go/releases/tag/v1.23.0)中可用
- 在 [Java SDK](https://docs.temporal.io/dev-guide/java/versioning#worker-versioning)版本 [1.20.0](https://github.com/temporalio/sdk-java/releases/tag/v1.20.0)中可用
- 在 [Go SDK](https://docs.temporal.io/dev-guide/typescript/versioning#worker-versioning)版本 [1.8.0](https://github.com/temporalio/sdk-typescript/releases/tag/v1.8.0)中可用
- 在 [Java SDK](https://python.temporal.io/temporalio.worker.Worker.html)版本 [1.3.0](https://github.com/temporalio/sdk-python/releases/tag/1.3.0)中可用
- 在 [Java SDK](https://dotnet.temporal.io/api/Temporalio.Worker.TemporalWorkerOptions.html#Temporalio_Worker_TemporalWorkerOptions_UseWorkerVersioning)版本 [0.1.0](https://github.com/temporalio/sdk-dotnet/releases/tag/0.1.0-beta1)中可用
- 在 Temporal Cloud 中尚不可用

{{% /alert %}}

{{% alert title="警告" color="warning" %}}
Worker Versioning 目前处于 Pre-release 阶段，Worker Versioning API 即将进行向后兼容的更改。 目前，您需要向集群提供动态配置参数以启用 Worker Versioning：

```text
temporal server start-dev \
   --dynamic-config-value frontend.workerVersioningDataAPIs=true \
   --dynamic-config-value frontend.workerVersioningWorkflowAPIs=true \
   --dynamic-config-value worker.buildIdScavengerEnabled=true
```

{{% /alert %}}

Worker Versioning 简化了将更改部署到 [工作流定义](https://docs.temporal.io/workflows/#workflow-definition)的过程。 它可以让你定义相互兼容的版本集，然后为定义 Worker 的代码分配一个 Build ID。 Temporal Server 使用 Build ID 来确定 Worker 可以处理哪些版本的工作流定义。

我们建议您在继续操作之前阅读工作流定义，因为工作流版本控制主要涉及帮助管理对这些定义的非确定性更改。

Worker Versioning 提供了一种便捷的方法，可确保在同一[任务队列](https://docs.temporal.io/workers#task-queue) 上操作不同工作流和活动定义的[Worker](https://docs.temporal.io/workers#workers)不会根据您定义的与该任务队列相关的版本集，尝试处理它们无法成功处理的[工作流任务](https://docs.temporal.io/workers#workflow-task)和[活动任务](https://docs.temporal.io/workers#activity-task-execution)，从而帮助管理非确定性更改。

要实现这一目标，需要为定义 Worker 的代码分配一个 Build ID（自由格式字符串），并通过更新与任务队列相关联的版本集（由 Temporal Server 存储）来指定哪些 Build ID 相互兼容。

### 何时以及为何要使用 Worker 版本控制

使用此功能的主要原因是将不兼容的更改部署到生存期较短的 [工作流](https://docs.temporal.io/workflows)。 在使用此功能的任务队列中，工作流启动器不必知道新版本的推出。

新部署的 Worker 中的新代码会执行新的 [工作流执行](https://docs.temporal.io/workers#workflow-execution)，而只有具有相应版本的 Worker 才会处理旧的工作流执行。

#### 退役旧的 worker

在使用旧 Worker 版本归档所有打开的工作流程后，就可以停用旧 Worker。 如果不需要查询已关闭的工作流，可以在该版本没有打开的工作流时将其退出运行。

例如，如果你有一个在一天内完成的工作流，一个好的策略是为每个新的辅助角色生成分配一个新的生成 ID，并将其添加为版本集中的新整体默认值。

由于工作流在一天内完成，因此您知道在部署新版本后（假设可用）后，无需让旧版 Worker 运行超过一天。

没有必要从集合中删除旧版本。 放任它们在那里不会造成任何伤害。

您也可以将此技术应用于持续时间较长的工作流；不过，您可能需要在打开的工作流完成时同时运行多个 Worker 版本。

#### 将代码更改部署到 Worker

该功能还可让您对当前打开的工作流实施兼容更改，或防止在当前打开的工作流上执行有错误的代码路径。 要做到这一点，可以将新版本添加到现有版本集，并将其定义为与现有版本_兼容_，这样就不会执行任何未来的工作流任务。 由于新版本会处理现有的 [事件历史记录](https://docs.temporal.io/workflows/#event-history)，因此它必须遵守通常的 [确定性约束](https://docs.temporal.io/workflows/#deterministic-constraints)，而且您可能需要使用其中一个 [versioning API](https://docs.temporal.io/workflows/#workflow-versioning)。

此外，该功能还可让您在对活动定义进行不兼容更改的同时，对使用这些活动的工作流定义进行不兼容更改。 这一功能之所以能起作用，是因为工作流在同一任务队列中调度的任何活动，默认情况下只会被分派给与调度该活动的工作流兼容的工人。 如果您想在为 Worker 创建新的不兼容 Build ID 的同时更改活动定义的类型签名，您可以这样做，而不必担心该活动会在其他具有不兼容定义的 Worker 上执行失败。 同样的原则也适用于子工作流。 对于活动和子工作流，您可以覆盖默认行为，并以最新的默认版本运行活动或子工作流。

{{% alert title="TIP" color="primary" %}}

版本控制任务队列上面向公众的工作流不应更改签名，因为这样做与工作流启动客户端不知道工作流定义更改的目的相矛盾。 如果需要更改工作流程的签名，请使用不同的工作流程类型或全新的任务队列。

{{% /alert %}}

{{% alert title="注意" color="info" %}}

如果将活动或子工作流程安排在与工作流程运行的任务队列不同的\*\*任务队列上，系统不会分配特定版本。 这意味着，如果目标队列是版本控制的，它们将在最新的默认值上运行；如果目标队列是非版本控制的，它们将在没有此功能的情况下运行。

{{% /alert %}}

**Continue-As-New 和 Worker 版本控制**

默认情况下，受版本控制的任务队列的“作为新任务继续”功能在与原始工作流相同的兼容集上启动继续工作流。

如果您在不同的任务队列中继续执行新任务，系统不会分配任何特定版本。 您还可以选择指定继续工作流应使用任务队列的最新默认版本开始。

### 如何使用 Worker 版本控制

要使用 Worker 版本控制，请按照以下步骤操作：

1. 定义任务队列的 Worker 内部版本标识符版本集。 您可以使用 `temporal` CLI 或 SDK。
2. 通过指定 Build ID 在 Worker 上启用该功能。

#### 定义版本集

无论是使用 [Temporal CLI](https://docs.temporal.io/cli/) 还是 SDK，更新版本集的感觉都是一样的。 您可以指定要面向的任务队列、要添加（或升级）的生成 ID、它是否成为新的默认版本，以及应将其视为兼容的任何现有版本。

本节的其余部分使用对一个 Task Queue 版本集的更新作为示例。

默认情况下，Task Queues 和 Workers 都处于未版本控制状态。 [未版本控制的 Worker](https://docs.temporal.io/workers#unversioned-workers) 可以轮询未版本控制的任务队列并接收任务。 要使用此功能，Task Queue 和 Worker 都必须与 Build ID 相关联。

如果您针对任务队列运行使用版本控制的 Worker，而该任务队列尚未设置为使用版本控制（或缺少该 Worker 的构建 ID），则该 Worker 不会获得任何任务。 同样，一个未版本控制的 Worker 轮询一个有版本控制的任务队列也是行不通的。

版本不需要遵循 SEMVER 或任何其他语义版本控制方案！

为清楚起见，以下示例中的版本看起来像 semver 版本，但并非必须如此。 版本可以是任意字符串。

首先，将版本“1.0”添加到任务队列中作为新的默认值。 您的版本集现在如下所示：

| set 1 (default) |
| ---------------------------------- |
| 1.0 (default)   |

在任务队列中启动的所有新工作流程的首个任务都分配给版本 `1.0`。 Build ID 设置为 "1.0 "的 worker 会收到这些任务。

如果没有分配版本的工作流仍在任务队列上运行，则没有版本的 worker 将承担这些任务。 因此，如果在添加第一个版本时有任何工作流打开，请确保这些 Worker 仍在运行。 如果您部署了任何具有不同版本的Worker，则这些Worker不会收到任何任务。

现在，假设您出于某种原因需要更改工作流。

将 "2.0 "作为新的默认设置添加到集合中：

| set 1                            | set 2 (default) |
| -------------------------------- | ---------------------------------- |
| 1.0 (default) | 2.0 (default)   |

在任务队列中启动的所有新工作流的首个任务都分配给版本 `2.0`。 现有的 `1.0` 工作流程继续生成以 `1.0` 为目标的任务。 每个 Worker 部署都会收到各自的任务。 对于每个新的不兼容版本，相同的概念都会向前推进。

也许您在 `2.0` 中发现了一个错误，您想确保所有打开的 `2.0` 工作流都能尽快切换到新代码。 因此，您可以在集合中添加 `2.1`，将其标记为与 `2.0` 兼容。 现在，您的集合如下所示：

| set 1                            | set 2 (default) |
| -------------------------------- | ---------------------------------- |
| 1.0 (default) | 2.0                                |
|                                  | 2.1 (default)   |

为上次工作流任务完成版本为“2.0”的工作流生成的所有新工作流任务现在都分配给版本“2.1”。 由于您指定了“2.1”与“2.0”兼容，因此 Temporal Server 假定具有此版本的 Worker 可以成功处理现有事件历史记录。

继续正常的开发周期，添加 `3.0` 版本。 这里没什么新鲜事：

| set 1                            | set 2                            | set 3 (default) |
| -------------------------------- | -------------------------------- | ---------------------------------- |
| 1.0 (default) | 2.0                              | 3.0 (default)   |
|                                  | 2.1 (default) |                                    |

现在想象一下，"3.0 "版本并没有明显的错误，但业务逻辑的某些地方不太理想。 您可以让现有的 `3.0` 工作流运行到完成，但希望新的工作流使用旧的 `2.x` 分支。 通过执行以 `2.1`（或 `2.0`）为目标的更新，并将其设置为当前默认值，可支持此操作，从而产生这些设置：

| set 1                            | set 3                            | set 2 (default) |
| -------------------------------- | -------------------------------- | ---------------------------------- |
| 1.0 (default) | 3.0 (default) | 2.0                                |
|                                  |                                  | 2.1 (default)   |

现在，新的工作流程从 `2.1` 开始。

#### 版本集上允许和禁止的操作

更改数据集的请求可执行以下操作之一：

- 将版本添加到集合中，作为新的整体默认兼容集中的新默认版本。
- 将版本添加到与现有版本兼容的现有集。
  - 可选择将其设置为默认值。
  - 可选择将该设置作为总默认设置。
- 将现有程序集中的某个版本升级为该程序集的默认版本。
- 将一个数据集提升为总默认数据集。

您无法显式删除版本。这有助于避免工作流意外卡住而无法取得进展的情况，因为与工作流关联的版本不再存在。

但是，有时您可能希望有意这样做。 如果您_想要_确保当前正在处理的所有工作流（例如“2.0”停止）（即使您还没有准备好新版本），您可以将新版本“2.1”添加到标记为与“2.0”兼容的集合中。 新任务将以“2.1”为目标，但由于您尚未部署任何“2.1”Worker，因此它们不会取得任何进展。

#### 设置约束

这些集合具有最大大小限制，默认为所有集合中有 100 个生成 ID。 该限制可在 Temporal Server 上通过 `limit.versionBuildIdLimitPerQueue` 动态配置属性进行配置。 如果超出限制，向集合添加新 Build ID 的操作将失败。

组数也有限制，默认为 10 个。 该限制可在 Temporal Server 上通过 `limit.versionCompatibleSetLimitPerQueue` 动态配置属性进行配置。

在实践中，这些限制很少会引起关注，因为在没有打开的工作流使用某个版本后，该版本就不再需要了，而且后台进程会删除不再需要的 ID 和集。

每个 Build ID 或版本字符串的大小也有限制，默认为 255 个字符。 该限制可在 Temporal Server 上通过 `limit.workerBuildIdSize` 动态配置属性进行配置。

### Build ID 可访问性

最终，你会想知道是否可以停用旧的 Worker 版本。 Temporal 提供的功能可帮助您确定打开或关闭的工作流是否仍在使用某个版本。 您可以使用 Temporal CLI 通过以下命令执行此操作：

```bash
temporal task-queue get-build-id-reachability
```

该命令确定每个任务队列的 Build ID 是无法访问、只能由关闭的工作流访问，还是由打开的工作流和新工作流访问。 例如，CLI 在此处显示此“2.0”构建 ID，新工作流和某些现有工作流均可访问：

```bash
temporal task-queue get-build-id-reachability --build-id "2.0"
```

```bash
BuildId                         TaskQueue                                   Reachability
    2.0  build-id-versioning-dc0068f6-0426-428f-b0b2-703a7e409a97  [NewWorkflows
                                                                   ExistingWorkflows]
```

有关详细信息，请参阅 [CLI 文档](https://docs.temporal.io/cli/) 或帮助输出。

您也可以在语言 SDK 中直接使用此 API `GetWorkerTaskReachability`。

### 未版本控制的 worker

未版本控制的 Worker 指的是在配置中未选择 Worker 版本控制功能的 Worker。 它们只能从任务队列接收任务，这些任务队列上没有定义任何版本集，或者有在版本添加到队列之前就已开始执行的开放工作流。

要从未修改的任务队列迁移，请向任务队列添加新的默认 Build ID。 然后，使用相同的 Build ID 部署 Worker。 未版本控制的 Worker 将继续处理打开的工作流，而具有新 Build ID 的 Worker 将处理新的工作流执行。

