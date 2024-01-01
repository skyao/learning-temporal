---
title: "如何在 Java SDK 进行版本控制"
linkTitle: "如何在 Java SDK 进行版本控制"
weight: 30
date: 2023-12-28
description: >
  Go SDK developer's guide - Versioning 中文翻译版本
---
# Java SDK developer's guide - Versioning

> 原文地址：https://docs.temporal.io/dev-guide/java/versioning#patching

Temporal Platform 要求工作流代码具有[确定性](https://docs.temporal.io/workflows#deterministic-constraints)。 由于这一要求，Temporal Java SDK 提供了两个专用的版本控制功能。

- [Workflow Patching APIs](https://docs.temporal.io/dev-guide/java/versioning#patching)
- [Worker Build Ids](https://docs.temporal.io/dev-guide/java/versioning#worker-versioning)

## 如何在 Java 中修补工作流

使用 `Workflow.getVersion` 函数返回应执行代码的版本，然后使用返回值选择正确的分支。 让我们来看一个例子：

```java
public void processFile(Arguments args) {
    String localName = null;
    String processedName = null;
    try {
        localName = activities.download(args.getSourceBucketName(), args.getSourceFilename());
        processedName = activities.processFile(localName);
        activities.upload(args.getTargetBucketName(), args.getTargetFilename(), processedName);
    } finally {
        if (localName != null) { // File was downloaded.
            activities.deleteLocalFile(localName);
        }
        if (processedName != null) { // File was processed.
            activities.deleteLocalFile(processedName);
        }
    }
}
```

现在，我们决定计算已处理文件的校验和，并将其传递给上传。 实现此更改的正确方法是：

```java
public void processFile(Arguments args) {
    String localName = null;
    String processedName = null;
    try {
        localName = activities.download(args.getSourceBucketName(), args.getSourceFilename());
        processedName = activities.processFile(localName);
        int version = Workflow.getVersion("checksumAdded", Workflow.DEFAULT_VERSION, 1);
        if (version == Workflow.DEFAULT_VERSION) {
            activities.upload(args.getTargetBucketName(), args.getTargetFilename(), processedName);
        } else {
            long checksum = activities.calculateChecksum(processedName);
            activities.uploadWithChecksum(
                args.getTargetBucketName(), args.getTargetFilename(), processedName, checksum);
        }
    } finally {
        if (localName != null) { // File was downloaded.
            activities.deleteLocalFile(localName);
        }
        if (processedName != null) { // File was processed.
            activities.deleteLocalFile(processedName);
        }
    }
}
```

稍后，当所有使用旧版本的工作流都完成后，可以删除旧分支。

```java
public void processFile(Arguments args) {
    String localName = null;
    String processedName = null;
    try {
        localName = activities.download(args.getSourceBucketName(), args.getSourceFilename());
        processedName = activities.processFile(localName);
        // getVersion call is left here to ensure that any attempt to replay history
        // for a different version fails. It can be removed later when there is no possibility
        // of this happening.
        Workflow.getVersion("checksumAdded", 1, 1);
        long checksum = activities.calculateChecksum(processedName);
        activities.uploadWithChecksum(
            args.getTargetBucketName(), args.getTargetFilename(), processedName, checksum);
    } finally {
        if (localName != null) { // File was downloaded.
            activities.deleteLocalFile(localName);
        }
        if (processedName != null) { // File was processed.
            activities.deleteLocalFile(processedName);
        }
    }
}
```

传递给`getVersion`调用的 Id 用于标识更改。 每项变更都将有自己的 Id。 但是，如果一项变更在工作流代码中产生多个位置，而新代码要么在所有位置执行，要么不在任何位置执行，那么它们就必须共享 Id。

## 如何在 Java 中使用 Worker 版本控制

警告

Worker 版本控制目前处于预发布状态，并且 Worker 版本控制 API 即将进行向后不兼容的更改。 目前，您需要向集群提供动态配置参数以启用 Worker Versioning：

```text
temporal server start-dev \
   --dynamic-config-value frontend.workerVersioningDataAPIs=true \
   --dynamic-config-value frontend.workerVersioningWorkflowAPIs=true \
   --dynamic-config-value worker.buildIdScavengerEnabled=true
```

要在 Java 中使用 Worker Versioning，您需要执行以下操作：

1. 确定并为已构建的 Worker 代码分配一个 Build ID，并选择版本控制。
2. 告诉您的 Worker 正在监听的任务队列该 Build ID，以及它是否与现有的 Builder ID 兼容。

### 将 Build ID 分配给 worker

比方说，您选择了 `deadbeef` 作为您的 Build ID，它可能是一个简短的 git 提交哈希值（作为 Build ID 的合理选择）。 要在 worker 代码中分配它，您需要分配以下 worker 选项：

```java
// ...
WorkerOptions workerOptions = WorkerOptions.newBuilder()
    .setBuildId("deadbeef")
    .setUseBuildIdForVersioning(true)
    // ...
    .build();
Worker w = workerFactory.newWorker("your_task_queue_name", workerOptions);
// ...
```

这就是您需要在 Worker 代码中做的全部工作。 重要的是，如果启动该 Worker，它不会接收任何任务。 这是因为您需要先将 Worker 的 Build ID 告知 Task Queue。

### 将 Worker 的 Builder ID 告知 Task Queue

现在，您可以使用 SDK（或 Temporal CLI）告诉任务队列您 Worker 的 Builder ID。 您可能希望将此作为 CI 部署流程的一部分。

```java
// ...
workflowClient.updateWorkerBuildIdCompatability(
    "your_task_queue_name", BuildIdOperation.newIdInNewDefaultSet("deadbeef"));
```

这段代码将 `deadbeef` Builder ID 添加到任务队列，作为新版本集中的唯一版本，并成为队列的默认版本。 新工作流在具有此 Build ID 的 Worker 上执行，现有工作流将继续由适当兼容的 Worker 进行处理。

如果您想将 Build ID 添加到现有的兼容集，则可以这样做：

```java
// ...
workflowClient.updateWorkerBuildIdCompatability(
    "your_task_queue_name", BuildIdOperation.newCompatibleVersion("deadbeef", "some-existing-build-id"));
```

这段代码将 `deadbeef` 添加到包含 `some-existing-build-id` 的现有兼容集，并将其标记为该兼容集的新默认构建 ID。

您还可以将一个集合中的现有 Build ID 提升为该集合的默认值：

```java
// ...
workflowClient.updateWorkerBuildIdCompatability(
    "your_task_queue_name", BuildIdOperation.promoteBuildIdWithinSet("deadbeef"));
```

您还可以将整组数据提升为队列的默认数据集。 新工作流将开始使用该集的默认值。

```java
// ...
workflowClient.updateWorkerBuildIdCompatability(
    "your_task_queue_name", BuildIdOperation.promoteSetByBuildId("deadbeef"));
```

### 为命令指定版本

默认情况下，如果活动、子工作流和 “Continue-as-New” 也使用相同的任务队列，则它们使用与调用它们的工作流相同的兼容版本集。

如果您想覆盖此行为，可以通过 `ActivityOptions`、`ChildWorkflowOptions` 或 `ContinueAsNewOptions` 对象上的 `setVersioningIntent` 方法指定您的意图。

例如，如果要为某个活动使用最新的默认版本，可以这样定义活动选项：

```java
// ...
private final MyActivity activity =
    Workflow.newActivityStub(
        MyActivity.class,
        ActivityOptions.newBuilder()
          .setScheduleToCloseTimeout(Duration.ofSeconds(10))
          .setVersioningIntent(VersioningIntent.VERSIONING_INTENT_DEFAULT)
          // ...other options
          .build()
    );
// ...
```



