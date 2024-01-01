---
title: "英文版"
linkTitle: "英文版"
weight: 10
date: 2023-12-28
description: >
  Workflow Versioning Strategies 英文原版
---

> 原文地址：https://community.temporal.io/t/workflow-versioning-strategies/6911

Once you have an understanding of the importance of writing [deterministic workflow code 180](https://docs.temporal.io/workflows#deterministic-constraints), you may find yourself grappling with the best way to make changes to that code. Here, I’ll outline a few common strategies and when and how you might employ them.

There are fundamentally three approaches available to you: Worker Versioning, the / APIs, and workflow-name based versioning. Each has its relative merits and demerits. You should prefer to use the first two, as they’re the officially supported methods provided by Temporal (and can be used together).

Let’s take a look at each:

## Worker Versioning

See the official docs on [Worker Versioning 316](https://docs.temporal.io/workers#worker-versioning). This should be your go-to strategy unless you have more specific needs.

**Advantages:**

- Simple, built-in
- Robust. Changes are, by default isolated from each other in a way that makes mistakes unlikely.
- Flexible – you can handle both compatible and incompatible changes

**Disadvantages:**

- Some operational burden in the form of worker management

## Patch & GetVersion APIs

We [have functions available to you 317](https://docs.temporal.io/workflows#workflow-versioning) in the SDK which allow you to branch in your workflow code based on whether or not the workflow is running with newer or older code. They take slightly different forms depending on what language you’re using. See the linked docs for more.

**Advantages:**

- Don’t need to change where workflow starters are pointing
- Allows you to change the yet-to-be-executed behavior of currently open workflows while remaining compatible with existing histories. The behavior of new workflows always takes the “new” path.
- Can be used with Worker Versioning to make compatible changes

**Disadvantages:**

- Conceptually complex
- Cognitive burden of needing to understand how both the “old” and “new” code paths work
- If used indefinitely on the same workflow definition, can lead to a mess of branching

## Workflow name based Versioning

Very similar to task-queue based versioning, now when you make changes, simply copy your workflow code to . You now redeploy **all** your workers, and change your workflow starters to point at the workflow.

**Advantages:**

- Don’t need to keep separate worker fleets pointed at different queues
- Conceptually simple
- Easier to patch old versions when necessary without affecting the code for new versions

**Disadvantages:**

- Code duplication - you can only delete the old workflow code when you know all workflows of that type/version have finished
- Still need to update workflow starters/clients

## Task Queue based Versioning (Replaced by Worker Versioning)

> ![:warning:](https://emoji.discourse-cdn.com/twitter/warning.png?v=12) Don’t use this! Use Worker Versioning as described above instead! This section is only kept for posterity.



In this approach, after making changes to your workflow code, you will want to deploy your workers pointed at a new task queue than your existing ones. For example if you previously were targeting the task queue , you’d now target or similar. You must also update whatever sources are starting new workflows to point at the new task queue name. Leave your old workers running, possibly reducing the number of instances, until there are no more open workflows on . Then you can decommission all of them.

**Advantages:**

- Conceptually simple
- Robust. Changes are isolated from each other in a way that makes mistakes unlikely.

**Disadvantages**

- Operationally complex (need to keep old workers alive and change what task queue clients point to). This can result in a number of sets of old workers, especially with long running workflows.
- Can’t be used to fix a bug in currently running/open workflows
