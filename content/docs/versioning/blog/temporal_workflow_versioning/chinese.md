---
title: "中文翻译版"
linkTitle: "中文翻译版"
weight: 20
date: 2023-12-28
description: >
  改变的未来：temporal工作流版本化中文翻译版
---

> 原文地址：https://keithtenzer.com/temporal/temporal_workflow_versioning/



## 概述

在本文中，我们将讨论 Temporal 工作流版本控制。 由于工作流可以运行很长时间，甚至无限期运行，因此工作流的版本控制并不总是一项微不足道的任务。 毕竟，如何在工作流程仍在运行时对其进行更改？ 此外，Temporal 工作流是确定性的，更改输入参数、活动顺序或在工作流代码中使用随机数、日期甚至 uuid 可能会破坏确定性，从而引发非确定性错误。 值得庆幸的是，根据目标，有几种工作流版本控制方法：

- 基于工作流代码的版本控制
- 基于任务队列的版本控制
- 基于工作流名称的版本控制

## 基于工作流代码的版本控制

基于工作流代码的版本控制涉及对工作流代码进行分支。 它允许您在工作流代码本身中维护版本更改。 根据使用的 SDK 不同，实现方式也略有不同。 Go 和 Java SDK 都提供 GetVersion API，而其他 SDK（如 Typescript 或 Python）则提供 Patch。 下面我们将讨论 GetVersion 机制的使用。 实质上，您的原始工作流版本是默认版本。 执行工作流时，版本将记录为工作流事件历史记录中的标记。 搜索属性 TemporalChangeVersion 已更新插入，因此可以轻松搜索具有这些更改的工作流。 您可以递增工作流版本，并使用 if-else 块来实现工作流特定版本中发生的情况。 这听起来可能非常令人困惑，因此下面我们将演练发布多个工作流版本的进度。

### 默认版本示例

在我们的原始版本中，我们将简单地执行一个名为 ActivityA 的活动。

```go
err := workflow.ExecuteActivity(ctx, ActivityA).Get(ctx, &result)
if err != nil {
	logger.Error("Activity failed.", "Error", err)
	return "", err
}
```

### 版本 1 示例

我们现在不想执行 ActivityA，而是执行 ActivityB。

```go
v := workflow.GetVersion(ctx, "Version1", workflow.DefaultVersion, 1)
if v == workflow.DefaultVersion {
	err := workflow.ExecuteActivity(ctx, ActivityA).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}
} else {
	err := workflow.ExecuteActivity(ctx, ActivityB).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}
}
```

### 版本 2 示例

我们现在不想执行 ActivityB，而是执行 ActivityB 和 ActivityC。

```go
v := workflow.GetVersion(ctx, "Version2", workflow.DefaultVersion, 2)
if v == workflow.DefaultVersion {
	err := workflow.ExecuteActivity(ctx, ActivityA).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}
} else if v == 1 {
	err := workflow.ExecuteActivity(ctx, ActivityB).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}
} else {
	err := workflow.ExecuteActivity(ctx, ActivityB).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}

	err = workflow.ExecuteActivity(ctx, ActivityC).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}
}
```

### 搜索工作流版本

了解何时可以删除或合并工作流版本代码的关键是确定给定版本的所有正在运行的工作流是否都已完成（不再运行）。 如上文所述，在 Go 中，搜索属性 TemporalChangeVersion 会被自动升级。 计划以其他语言提供此属性，但截至目前，您需要自己手动更新插入此属性，但这也很简单。 检查工作流历史记录，我们可以看到 TemporalChangeVersion 已更新插入。

![Upsert](https://keithtenzer.com/assets/2023-1-23/upsert.png)

使用 TemporalChangeVersion 搜索属性，我们现在可以查找与工作流代码的特定版本匹配的工作流。

![Search](https://keithtenzer.com/assets/2023-1-23/search.png)

最后，我们可以从原始代码块中删除不需要的版本，保持代码干净。

## 基于任务队列的版本控制

基于任务队列的版本控制涉及对任务队列名称进行版本控制。 Worker 和任何工作流启动器都需要传递任务队列名称和工作流定义。 当实例化一个新的 Worker 对象时，该 Worker 就会完成上述操作。

任务队列的名称是版本控制。

```go
w := worker.New(c, "versioning", worker.Options{})
```

为了使用任务队列进行版本控制，您需要为工作流的每个版本创建一个新的任务队列。

```go
w := worker.New(c, "versioning-v1", worker.Options{})
```

您还需要更新所有的工作流启动器，以通过 workflowOptions 传递任务队列名称。

```go
workflowOptions := client.StartWorkflowOptions{
	ID:        "versioning-workflowId",
	TaskQueue: "versioning-v1",
}
```

为了确定正在运行的工作流版本，您可以检查工作流 worker。 这将显示任务队列（您的版本）以及针对该任务队列进行轮询的任何 worker。

![Taskqueue](https://keithtenzer.com/assets/2023-1-23/taskqueue.png)

## 基于工作流名称的版本控制

基于工作流名称的版本控制与基于任务队列的版本控制非常相似。 每当创建新版本时，都需要更新 worker 和 starter，因为两者都加载了工作流定义。 每次创建新版本时，只需创建一个包含 v1、v2 等的新工作流定义文件。

在 worker 中，您需要更改工作流和活动的注册。

```go
w.RegisterWorkflow(versioning-v1.Workflow)
w.RegisterActivity(versioning-v1.ActivityA)
w.RegisterActivity(versioning-v1.ActivityB)
w.RegisterActivity(versioning-v1.ActivityC)
```

在 starter 中，您需要使用新的工作流定义更新 ExecuteWorkflow 调用。

```go
we, err := c.ExecuteWorkflow(context.Background(), workflowOptions, versioning-v1.Workflow, "Temporal")
```

当然，仅仅这样做很难确定正在运行的版本，此外，你还可以考虑在工作流类型或工作流 Id 中反映版本。

## 选择版本控制策略

选择正确的版本管理策略在很大程度上取决于您的需求和 CI/CD 流程。 通常，如果您不想对任务队列/worker 进行微观管理，不想更改启动器指向的位置，并且需要在不破坏现有工作流兼容性的情况下更改正在运行的工作流的未来行为，则工作流代码版本控制可能是更好的方法。 不过，基于工作流的版本管理还有一点需要注意，那就是如果要维护很多版本，就会变得非常难以管理。 您肯定需要清理未使用的旧版本，以保持代码简单。 幸运的是，查询工作流程并确定何时不再需要某个版本的工作相当简单，就像演示的那样。 此外，如果您不介意管理额外的 worker 或任务队列，并且不需要更改运行工作流的未来行为，则基于任务队列或工作流名称的版本控制将是一种更简单的整体方法。

Spencer Judge 还围绕各种[版本化策略](https://community.temporal.io/t/workflow-versioning-strategies/6911) 的优缺点提供了更多细节。

## 总结

在本文中，我们讨论了 Temporal 工作流版本控制的不同策略。 我们详细探讨了工作流代码、工作流名称和任务队列版本控制。 最后，就如何选择适当的策略提供了一些一般性指导。
