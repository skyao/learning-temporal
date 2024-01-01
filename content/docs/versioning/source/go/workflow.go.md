---
title: "workflow.go"
linkTitle: "workflow.go"
weight: 10
date: 2023-12-28
description: >
  Temporal workflow 版本控制相关的 workflow.go 源码
---



### GetVersion() 方法

GetVersion 用于安全地对工作流定义执行向后不兼容的更改。不允许在工作流运行时更新工作流代码，因为这会破坏确定性。解决办法是，既要有用于重放现有工作流的旧代码，也要有首次执行时使用的新代码。首次执行时，GetVersion 会返回最大支持版本。该版本将作为标记事件记录到工作流历史中。即使修改了最大支持版本，重放时也会返回记录的版本。DefaultVersion 常量包含之前未进行版本控制的代码版本。例如，最初的工作流程有以下代码：

```go
err = workflow.ExecuteActivity(ctx, foo).Get(ctx, nil)
```

应更新为:

```go
err = workflow.ExecuteActivity(ctx, bar).Get(ctx, nil)
```

执行更新的向后兼容方法是：

```go
v :=  GetVersion(ctx, "fooChange", DefaultVersion, 0)
if v  == DefaultVersion {
    err = workflow.ExecuteActivity(ctx, foo).Get(ctx, nil)
} else {
    err = workflow.ExecuteActivity(ctx, bar).Get(ctx, nil)
}
```

然后，bar 必须改为 baz：

```go
v :=  GetVersion(ctx, "fooChange", DefaultVersion, 1)
if v  == DefaultVersion {
    err = workflow.ExecuteActivity(ctx, foo).Get(ctx, nil)
} else if v == 0 {
    err = workflow.ExecuteActivity(ctx, bar).Get(ctx, nil)
} else {
    err = workflow.ExecuteActivity(ctx, baz).Get(ctx, nil)
}
```

之后，当没有工作流执行 DefaultVersion 时，就可以删除相应的分支：

```go
v :=  GetVersion(ctx, "fooChange", 0, 1)
if v == 0 {
    err = workflow.ExecuteActivity(ctx, bar).Get(ctx, nil)
} else {
    err = workflow.ExecuteActivity(ctx, baz).Get(ctx, nil)
}
```

即使只剩下一个分支，也建议保留 GetVersion() 调用：

```go
GetVersion(ctx, "fooChange", 1, 1)
err = workflow.ExecuteActivity(ctx, baz).Get(ctx, nil)
```

保留它的原因是 1）它可以确保如果有旧版本的执行仍在运行，则执行会在此处失败，而不会继续；2）如果需要对 "fooChange "进行更多更改，例如将活动从 baz 更改为 qux，则只需将 maxVersion 从 1 更新为 2 即可。

请注意，您只需为每个 changeID 保留第一次调用 GetVersion()。所有后续调用 GetVersion() 的相同更改 ID 都可以安全删除。不过，如果你真的想删除第一次 GetVersion() 调用，也可以这样做，但你需要确保：1）所有旧版本的执行都已完成；2）你不能再使用 "fooChange "作为 changeID。如果您需要对同一部分进行修改，如将 baz 改为 qux，您需要使用不同的 changeID，如 "fooChange-fix2"，并重新从 DefaultVersion 开始 minVersion。代码如下

```go
v := workflow.GetVersion(ctx, "fooChange-fix2", workflow.DefaultVersion, 0)
if v == workflow.DefaultVersion {
  err = workflow.ExecuteActivity(ctx, baz, data).Get(ctx, nil)
} else {
  err = workflow.ExecuteActivity(ctx, qux, data).Get(ctx, nil)
}
```



源代码在 `internal/workflow.go` 中：

```go
// Version represents a change version. See GetVersion call.
Version int

// DefaultVersion is a version returned by GetVersion for code that wasn't versioned before
const DefaultVersion Version = -1

// TemporalChangeVersion is used as search attributes key to find workflows with specific change version.
const TemporalChangeVersion = "TemporalChangeVersion"

func GetVersion(ctx Context, changeID string, minSupported, maxSupported Version) Version {
	assertNotInReadOnlyState(ctx)
	i := getWorkflowOutboundInterceptor(ctx)
	return i.GetVersion(ctx, changeID, minSupported, maxSupported)
}

func (wc *workflowEnvironmentInterceptor) GetVersion(ctx Context, changeID string, minSupported, maxSupported Version) Version {
	return wc.env.GetVersion(changeID, minSupported, maxSupported)
}
```



`internal/internal_event_handlers.go`：

```go
	workflowEnvironmentImpl struct {
		workflowInfo *WorkflowInfo
		......
		changeVersions             map[string]Version
    ......
  }

func (wc *workflowEnvironmentImpl) GetVersion(changeID string, minSupported, maxSupported Version) Version {
	// 检查 changeID 所对应的 version 是否已经存在
  if version, ok := wc.changeVersions[changeID]; ok {
    // 如果有，验证通过后直接返回保存的version
		validateVersion(changeID, version, minSupported, maxSupported)
		return version
	}

	var version Version
	if wc.isReplay {
		// GetVersion for changeID is called first time in replay mode, use DefaultVersion
    // isReplay + version 没有被保存，说明在 replay 之前的执行中没有调用 GetVersion()方法，
    // 也就是说 GetVersion()方法是在 workflow 第一次执行之后再加入的
    // 这是老代码启动workflow instance + 新代码再次 replay 的场景
    // 这个时候确实只能取 DefaultVersion
		version = DefaultVersion
	} else {
		// GetVersion for changeID is called first time (non-replay mode), generate a marker command for it.
		// Also upsert search attributes to enable ability to search by changeVersion.
		// version 没有被保存 + 不是 replay，确认是 worklfow instance 的第一次执行
    // version 取容许的最大值
    version = maxSupported
    
    // 这一段是 Search Attributes 的处理，后面细看。
		changeVersionSA := createSearchAttributesForChangeVersion(changeID, version, wc.changeVersions)
		attr, err := validateAndSerializeSearchAttributes(changeVersionSA)
		if err != nil {
			wc.logger.Warn(fmt.Sprintf("Failed to seralize %s search attribute with: %v", TemporalChangeVersion, err))
		} else {
			// Server has a limit for the max size of a single search attribute value. If we exceed the default limit
			// do not try to upsert as it will cause the workflow to fail.
			updateSearchAttribute := true
			if wc.sdkFlags.tryUse(SDKFlagLimitChangeVersionSASize, !wc.isReplay) && len(attr.IndexedFields[TemporalChangeVersion].GetData()) >= changeVersionSearchAttrSizeLimit {
				wc.logger.Warn(fmt.Sprintf("Serialized size of %s search attribute update would "+
					"exceed the maximum value size. Skipping this upsert. Be aware that your "+
					"visibility records will not include the following patch: %s", TemporalChangeVersion, getChangeVersion(changeID, version)),
				)
				updateSearchAttribute = false
			}
			wc.commandsHelper.recordVersionMarker(changeID, version, wc.GetDataConverter(), updateSearchAttribute)
			if updateSearchAttribute {
				_ = wc.UpsertSearchAttributes(changeVersionSA)
			}
		}
	}

  // 验证版本
	validateVersion(changeID, version, minSupported, maxSupported)
  // 保存版本到 changeVersions 记录中，key 是 changeID
	wc.changeVersions[changeID] = version
	return version
}
```

#### validateVersion() 方法

validateVersion() 方法检查指定的 version 是否满足要求，如 minSupported < version < maxSupported

```go
func validateVersion(changeID string, version, minSupported, maxSupported Version) {
	if version < minSupported {
		panicIllegalState(fmt.Sprintf("Workflow code removed support of version %v. "+
			"for \"%v\" changeID. The oldest supported version is %v",
			version, changeID, minSupported))
	}
	if version > maxSupported {
		panicIllegalState(fmt.Sprintf("Workflow code is too old to support version %v "+
			"for \"%v\" changeID. The maximum supported version is %v",
			version, changeID, maxSupported))
	}
}
```



## changeVersion 相关

changeVersion 是 "changeID" + version，类如 "changeIDabc-1":

```go
func getChangeVersion(changeID string, version Version) string {
	return fmt.Sprintf("%s-%v", changeID, version)
}
```

getChangeVersions() 是把当前给定的 changeID + version，以及其他已经保存的 ChangeVersions，一起放在一个 []string 中返回。注意这个操作不会将当前 changeID + version 保存到 existingChangeVersions 中：

```go
func getChangeVersions(changeID string, version Version, existingChangeVersions map[string]Version) []string {
	res := []string{getChangeVersion(changeID, version)}
	for k, v := range existingChangeVersions {
		res = append(res, getChangeVersion(k, v))
	}
	return res
}
```

createSearchAttributesForChangeVersion() 方法：

```go
// TemporalChangeVersion is used as search attributes key to find workflows with specific change version.
const TemporalChangeVersion = "TemporalChangeVersion"

func createSearchAttributesForChangeVersion(changeID string, version Version, existingChangeVersions map[string]Version) map[string]interface{} {
	return map[string]interface{}{
		TemporalChangeVersion: getChangeVersions(changeID, version, existingChangeVersions),
	}
}
```

