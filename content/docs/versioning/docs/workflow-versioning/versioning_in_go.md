---
title: "如何在 go SDK 进行版本控制"
linkTitle: "如何在 go SDK 进行版本控制"
weight: 20
date: 2023-12-28
description: >
  Go SDK developer's guide - Versioning 中文翻译版本
---
# Go SDK developer's guide - Versioning

> 原文地址：https://docs.temporal.io/dev-guide/go/versioning#patching

Temporal Platform 要求工作流代码具有[确定性](https://docs.temporal.io/workflows#deterministic-constraints)。 由于这一要求，Temporal Go SDK 提供了两个专用的版本控制功能。

- [工作流 Patching API](https://docs.temporal.io/dev-guide/go/versioning#patching)
- [Worker Build Ids](https://docs.temporal.io/dev-guide/go/versioning#worker-versioning)

## Temporal Go SDK Patching APIs

Temporal 工作流的定义代码必须是确定的，因为 Temporal 使用事件溯源（event sourcing），通过在工作流定义代码上重放保存的历史事件数据来重建工作流状态。 这意味着，如果处理不当，工作流定义代码的任何不兼容更新都可能导致非确定性问题。

由于我们设计的工作流可能会长期大规模运行，因此使用 Temporal 进行版本管理的方式也有所不同。 我们将在 30 分钟的可选介绍中详细说明： https\://www\.youtube.com/watch?v=kkP899WxgzY

考虑以下工作流定义：

```go
func YourWorkflow(ctx workflow.Context, data string) (string, error) {
        ao := workflow.ActivityOptions{
                ScheduleToStartTimeout: time.Minute,
                StartToCloseTimeout:    time.Minute,
        }
        ctx = workflow.WithActivityOptions(ctx, ao)
        var result1 string
        err := workflow.ExecuteActivity(ctx, ActivityA, data).Get(ctx, &result1)
        if err != nil {
                return "", err
        }
        var result2 string
        err = workflow.ExecuteActivity(ctx, ActivityB, result1).Get(ctx, &result2)
        return result2, err
}
```

现在，假设我们用 ActivityC 替换了 ActivityA，并部署了更新的代码。 如果现有工作流执行是由工作流代码的原始版本启动的，其中 ActivityA 已完成并将结果记录到历史记录中，则新版本的工作流代码将选取该工作流执行并尝试从那里恢复。 但是，工作流将会失败，因为新代码期望从历史数据中得到 ActivityC 的结果，但得到的却是 ActivityA 的结果。 这会导致工作流因出现非确定性错误而失败。

因此，我们使用`workflow.GetVersion().`

```go
var err error
v := workflow.GetVersion(ctx, "Step1", workflow.DefaultVersion, 1)
if v == workflow.DefaultVersion {
        err = workflow.ExecuteActivity(ctx, ActivityA, data).Get(ctx, &result1)
} else {
        err = workflow.ExecuteActivity(ctx, ActivityC, data).Get(ctx, &result1)
}
if err != nil {
        return "", err
}

var result2 string
err = workflow.ExecuteActivity(ctx, ActivityB, result1).Get(ctx, &result2)
return result2, err
```

为新工作流执行运行 `workflow.GetVersion()` 时，会在工作流历史记录中记录一个标记，这样，今后在此工作流执行中对此更改 Id 的所有 `GetVersion` 调用--示例中的 `Step 1` --将始终返回给定的版本号，即示例中的 "1"。

如果要进行额外的更改，例如用 ActivityD 替换 ActivityC，则需要添加一些额外的代码：

```go
v := workflow.GetVersion(ctx, "Step1", workflow.DefaultVersion, 2)
if v == workflow.DefaultVersion {
        err = workflow.ExecuteActivity(ctx, ActivityA, data).Get(ctx, &result1)
} else if v == 1 {
        err = workflow.ExecuteActivity(ctx, ActivityC, data).Get(ctx, &result1)
} else {
        err = workflow.ExecuteActivity(ctx, ActivityD, data).Get(ctx, &result1)
}
```

请注意，我们已将 `maxSupported` 从 1 改为 2。 在引入此“GetVersion()”调用之前已通过此“GetVersion()”调用的工作流将返回“DefaultVersion”。 在将 `maxSupported` 设为 1 时运行的工作流返回 1。 新工作流返回 2。

在确定版本 1 之前的所有工作流执行都已完成后，可以删除该版本的代码。 它现在应如下所示：

```go
v := workflow.GetVersion(ctx, "Step1", 1, 2)
if v == 1 {
        err = workflow.ExecuteActivity(ctx, ActivityC, data).Get(ctx, &result1)
} else {
        err = workflow.ExecuteActivity(ctx, ActivityD, data).Get(ctx, &result1)
}
```

您会注意到，"minSupported "已从 "DefaultVersion "更改为 "1"。 如果在此代码上重放工作流执行历史的旧版本，则会失败，因为预期的最小版本是 1。 在确定版本 1 的所有工作流执行都已完成后，您可以删除版本 1，这样您的代码看起来就像下面这样：

```go
_ := workflow.GetVersion(ctx, "Step1", 2, 2)
err = workflow.ExecuteActivity(ctx, ActivityD, data).Get(ctx, &result1)
```

请注意，我们保留了对 `GetVersion()` 的调用。 保留此调用有两个原因：

1. 这样可以确保，如果旧版本的工作流执行仍在运行，它将在此处失败而无法继续。
2. 如果需要对 `Step1` 进行额外的更改，例如将 ActivityD 更改为 ActivityE，只需将 `maxVersion` 从 2 更新为 3，然后从这里开始分支。

您只需要为每个 `changeID` 保留对 `GetVersion()` 的第一次调用。 所有后续调用 `GetVersion()`，如果更改标识相同，则可以安全删除。 如有必要，您可以删除第一个`GetVersion()`调用，但需要确保以下几点：

- 所有使用旧版本的执行都已完成。
- 您不能再使用 `Step1` 作为 changeId。 如果将来需要对同一部分进行更改，例如从 ActivityD 更改为 ActivityE，则需要使用不同的 changeId，如 `Step1-fix2`，并从 DefaultVersion 重新开始 minVersion。 代码如下所示：

```go
v := workflow.GetVersion(ctx, "Step1-fix2", workflow.DefaultVersion, 1)
if v == workflow.DefaultVersion {
        err = workflow.ExecuteActivity(ctx, ActivityD, data).Get(ctx, &result1)
} else {
        err = workflow.ExecuteActivity(ctx, ActivityE, data).Get(ctx, &result1)
}
```

如果您不需要保留当前正在运行的工作流执行，则升级工作流非常简单。 在部署不使用 `GetVersion()`的新版本工作流代码时，您可以简单地终止所有当前正在运行的工作流执行，并暂停创建新的工作流，然后恢复工作流的创建。 但情况往往并非如此，您需要处理当前正在运行的工作流执行，因此使用 `GetVersion()` 来更新代码是最合适的方法。

但是，如果您希望当前正在运行的工作流基于当前工作流逻辑继续运行，但希望确保新工作流在新逻辑上运行，则可以将工作流定义为新的“WorkflowType”，并更改开始路径（调用“StartWorkflow（）”）以启动新的工作流类型。

## 健全性检查

Temporal 客户端 SDK 会进行健全性检查，以防止出现明显的不兼容更改。 健全性检查以相同的顺序验证在重播中发出的命令是否与历史记录中记录的事件匹配。 命令通过调用以下任一方法生成：

- workflow\.ExecuteActivity()
- workflow\.ExecuteChildWorkflow()
- workflow\.NewTimer()
- workflow\.RequestCancelWorkflow()
- workflow\.SideEffect()
- workflow\.SignalExternalWorkflow()
- workflow\.Sleep()

添加、删除或重新排序上述任何方法都会触发健全性检查，并导致不确定错误。

健全性检查不会执行彻底检查。 例如，它不检查 Activity 的输入参数或计时器持续时间。 如果对每个属性强制执行检查，则维护工作流代码会变得过于严格和困难。 例如，如果您将 Activity 代码从一个软件包移动到另一个软件包，这一移动就会改变 `ActivityType`，从技术上讲，这就变成了不同的 Activity。 但我们不想因为这种变化而失败，所以我们只检查 `ActivityType` 中的函数名部分。

## 如何在 Go 中使用 Worker 版本控制

{{% alert title="警告" color="warning" %}}

Worker 版本控制目前处于预发布状态，并且 Worker 版本控制 API 即将进行向后不兼容的更改。 目前，您需要向集群提供动态配置参数以启用 Worker Versioning：

```text
temporal server start-dev \
   --dynamic-config-value frontend.workerVersioningDataAPIs=true \
   --dynamic-config-value frontend.workerVersioningWorkflowAPIs=true \
   --dynamic-config-value worker.buildIdScavengerEnabled=true
```

{{% /alert %}}

要在 Go 中使用 Worker Versioning，您需要执行以下操作：

1. 确定并为已构建的 Worker 代码分配一个 Build ID，并选择版本控制。
2. 告诉您的 Worker 正在监听的任务队列该 Build ID，以及它是否与现有的 Builder ID 兼容。

### 将 Build ID 分配给 worker

比方说，您选择了 `deadbeef` 作为您的 Build ID，它可能是一个简短的 git 提交哈希值（作为 Build ID 的合理选择）。 要在 worker 代码中分配它，您需要分配以下 worker 选项：

```go
// ...
workerOptions := worker.Options{
   BuildID: "deadbeef",
   UseBuildIDForVersioning: true,
// ...
}
w := worker.New(c, "your_task_queue_name", workerOptions)
// ...
```

这就是您需要在 Worker 代码中做的全部工作。 重要的是，如果启动该 Worker，它不会接收任何任务。 这是因为您需要先将 Worker 的 Build ID 告知 Task Queue。

### 将 Worker 的 Builder ID 告知 Task Queue

现在，您可以使用 SDK（或 Temporal CLI）告诉任务队列您 Worker 的 Builder ID。 您可能希望将此作为 CI 部署流程的一部分。

```go
// ...
err := client.UpdateWorkerBuildIdCompatibility(ctx, &client.UpdateWorkerBuildIdCompatibilityOptions{
   TaskQueue: "your_task_queue_name",
   Operation: &client.BuildIDOpAddNewIDInNewDefaultSet{
      BuildID: "deadbeef",
   },
})
```

这段代码将 `deadbeef` Builder ID 添加到任务队列，作为新版本集中的唯一版本，并成为队列的默认版本。 新工作流在具有此 Build ID 的 Worker 上执行，现有工作流将继续由适当兼容的 Worker 进行处理。

如果您想将 Build ID 添加到现有的兼容集，则可以这样做：

```go
// ...
err := client.UpdateWorkerBuildIdCompatibility(ctx, &client.UpdateWorkerBuildIdCompatibilityOptions{
   TaskQueue: "your_task_queue_name",
   Operation: &client.BuildIDOpAddNewCompatibleVersion{
      BuildID:                   "deadbeef",
      ExistingCompatibleBuildId: "some-existing-build-id",
   },
})
```

这段代码将 `deadbeef` 添加到包含 `some-existing-build-id` 的现有兼容集，并将其标记为该兼容集的新默认构建 ID。

您还可以将一个集合中的现有 Build ID 提升为该集合的默认值：

```go
// ...
err := client.UpdateWorkerBuildIdCompatibility(ctx, &client.UpdateWorkerBuildIdCompatibilityOptions{
   TaskQueue: "your_task_queue_name",
   Operation: &client.BuildIDPromoteIDWithinSet{
      BuildID: "some-existing-build-id",
   },
})
```

您还可以将整组数据提升为队列的默认数据集。 新工作流将开始使用该集的默认值。

```go
// ...
err := client.UpdateWorkerBuildIdCompatibility(ctx, &client.UpdateWorkerBuildIdCompatibilityOptions{
   TaskQueue: "your_task_queue_name",
   Operation: &client.BuildIDPromoteSet{
      BuildID: "some-existing-build-id",
   },
})
```

### 为命令指定版本

默认情况下，如果活动、子工作流和 “Continue-as-New” 也使用相同的任务队列，则它们使用与调用它们的工作流相同的兼容版本集。

如果要覆盖此行为，可通过相应选项结构体上的 `VersioningIntent` 字段指定自己的意图。

例如，如果要对活动使用最新的默认版本，请在工作流代码中执行以下操作：

```go
// ...
ao := workflow.ActivityOptions{
    VersioningIntent: VersioningIntentDefault,
    // ...other options
}
activityCtx := workflow.WithActivityOptions(ctx, ao)
var yourActivityResult YourActivityResultType
err := workflow.ExecuteActivity(ctx, YourActivityDefinition, yourActivityParam).Get(ctx, &yourActivityResult)
// ...
```

#### 为 "Continue-As-New" 指定版本

使用 “Continue-As-New” 功能时，请使用“WithWorkflowVersioningIntent”上下文修饰符。

在使用 `NewContinueAsNewError` 构造 `ContinueAsNewError` 对象之前，函数 `WithWorkflowVersioningIntent` 会设置 `VersioningIntentDefault` ：

```go
ctx = workflow.WithWorkflowVersioningIntent(ctx, temporal.VersioningIntentDefault)
err := workflow.NewContinueAsNewError(ctx, "WorkflowName")
```

如果您正在不兼容的 Worker Build ID 之间迁移工作流，并且希望继续执行的工作流使用任务队列的最新默认版本，那么请在调用 `NewContinueAsNewError` 之前使用前面所示的 `WithWorkflowVersioningIntent`。
