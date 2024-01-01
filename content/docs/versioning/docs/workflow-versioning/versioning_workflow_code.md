---
title: "工作流代码版本控制"
linkTitle: "工作流代码版本控制"
weight: 10
date: 2023-12-28
description: >
  Versioning Workflow code 中文翻译版本
---


> 原文地址：[Versioning Workflow code](https://docs.temporal.io/workflows#workflow-versioning)

Temporal Platform 要求工作流代码（工作流定义 / Workflow Definitions）具有确定性。这一要求意味着开发人员应考虑如何处理工作流代码随时间推移而发生的变化。

如果您的工作流执行时间足够长，以至于一个 Worker 必须能够执行同一工作流类型的多个版本，那么版本策略就更加重要了。

除了为具有相同名称的工作流类型创建新任务队列的功能外，Temporal Platform还提供了工作流 Patch API 和基于 Worker Build Id 的版本控制功能。

## Patching

Patching API 可根据开发人员指定的版本标识符，在工作流定义内部创建逻辑分支。这一功能对于需要更新但仍有依赖于其运行的工作流执行的工作流定义逻辑非常有用。

- How to patch Workflow code in Go
- [How to patch Workflow code in Java](https://docs.temporal.io/dev-guide/java/versioning#patching)
- [How to patch Workflow code in Python](https://docs.temporal.io/dev-guide/python/versioning#python-sdk-patching-api)
- [How to patch Workflow code in TypeScript](https://docs.temporal.io/dev-guide/typescript/versioning#patching)

## Worker Build Ids

Temporal [基于 Worker Build Id 的版本控制](https://docs.temporal.io/workers#worker-versioning) 可让你定义相互兼容的版本集，然后为定义 Worker 的代码分配一个 Build Id。

- [How to version Workers in Go](https://docs.temporal.io/dev-guide/go/versioning#worker-versioning)
- [How to version Workers in Java](https://docs.temporal.io/dev-guide/java/versioning#worker-versioning)
- [How to version Workers in Python](https://docs.temporal.io/dev-guide/python/versioning#worker-versioning)
- [How to version Workers in TypeScript](https://docs.temporal.io/dev-guide/typescript/versioning#worker-versioning)
