---
title: "Saga.java源码学习"
linkTitle: "Saga.java"
weight: 10
date: 2021-01-29
description: >
  Temporal Java SDK 中 Saga.java 的源码
---



## Javadoc 中的介绍

该类实现了 Saga 应用程序中经常需要的执行补偿操作的逻辑。下面是一个骨架，展示了如何在工作流代码中使用该类：

```java
Saga saga = new Saga(options);
 try {
   String r = activity.foo();
   saga.addCompensation(activity::cleanupFoo, arg2, r);
   Promise r2 = Async.function(activity::bar);
   r2.thenApply(r->saga.addCompensation(activity.cleanupBar(r));
   ...
   useR2(r2.get());
 } catch (Exception e) {
    saga.compensate();
    // Other error handling if needed.
 }
```





## 辅助类

### 内嵌的 Options Class

```java
  public static final class Options {
    private final boolean parallelCompensation;
    private final boolean continueWithError;
    
    private Options(boolean parallelCompensation, boolean continueWithError) {
      this.parallelCompensation = parallelCompensation;
      this.continueWithError = continueWithError;
    }
  }
```

构造函数为 private，只能通过下面的 Builder 来进行构建。

#### 内嵌的 Builder Class

内嵌的 Options Class 中再度内嵌了一个 Builder class，用来帮助构建 Options 类。就两个属性：

```java
    public static final class Builder {
      private boolean parallelCompensation;
      private boolean continueWithError;
      ......
    }
```

parallelCompensation 将决定补偿操作是否并行运行。如果 parallelCompensation 为 false，那么补偿操作将以添加时的相反顺序运行：

```java
public Builder setParallelCompensation(boolean parallelCompensation) {
  this.parallelCompensation = parallelCompensation;
  return this;
}
```

continueWithError 为用户提供了在运行补偿操作时出现异常时跳出补偿操作的选项。只有当并行补偿为 false 时，这个选项才有用。如果并行补偿设置为 "true"，那么无论如何，所有补偿操作都将被触发，如果有异常，调用者将收到异常反馈：

```java
public Builder setContinueWithError(boolean continueWithError) {
  this.continueWithError = continueWithError;
  return this;
}
```

build() 方法根据设置的 parallelCompensation 和 continueWithError 构建相应的 Options 对象实例：

```java
public Options build() {
  return new Options(parallelCompensation, continueWithError);
}
```

### 内嵌的 CompensationException class

CompensationException 定义了需要进行Compensation 处理的异常，继承自 RuntimeException：

```java
  public static class CompensationException extends RuntimeException {
    public CompensationException(Throwable cause) {
      super("Exception from saga compensate", cause);
    }
  }
```



## Saga 类的实现

### saga 类定义

```java
public final class Saga {
  private final Options options;
  private final List<Functions.Func<Promise>> compensationOps = new ArrayList<>();
......
}
```

构造方法需要传入Options 实例，前面提到需要用 Builder 构建 ：

```java
  public Saga(Options options) {
    this.options = options;
  }
```

compensationOps 字段保存有需要执行的补偿操作，List 结构表明是有顺序的。

补偿操作的类型是 `Functions.Func<Promise>`，这是一个 temporal 定义的接口：

```java
package io.temporal.workflow;
public final class Functions 
    public interface TemporalFunctionalInterfaceMarker {}

    @FunctionalInterface
    public interface Func<R> extends TemporalFunctionalInterfaceMarker, Serializable {
      R apply();
    }
  ......
}
```



### 登记补偿操作

addCompensation() 登记补偿操作:

```java
  public void addCompensation(Functions.Proc operation) {
    compensationOps.add(() -> Async.procedure(operation));
  }
```

这代码等价于：

```java
  public void addCompensation(Functions.Proc operation) {
    compensationOps.add(new Func<Promise>() {
      @Override
      public Promise apply() {
        return Async.procedure(operation);
      }
    });
    };
}
```

也就是在 `List<Functions.Func<Promise>> compensationOps` 这个字段中增加了一个匿名的内部类实现，需要时执行其 apply() 方法就会调用 Async.procedure(operation) 来执行补偿操作。

其中，Async.procedure() 方法会调用 AsyncInternal 来进行处理（后续的这里不展开）：

```java
  public static Promise<Void> procedure(Functions.Proc procedure) {
    return AsyncInternal.procedure(procedure);
  }
```

上面这个 addCompensation()  方法的 operation 是没有参数的，如果 operation 需要参数，则需要如下几个方法，在登记补偿操作的同时保存 operation 的参数，根据参数的个数，有6个重载方法方法分别支持 1 到 6 个 operation 参数：

```java
  public <A1> void addCompensation(Functions.Proc1<A1> operation, A1 arg1) {
    compensationOps.add(() -> Async.procedure(operation, arg1));
  }

  public <A1, A2> void addCompensation(Functions.Proc2<A1, A2> operation, A1 arg1, A2 arg2) {
    compensationOps.add(() -> Async.procedure(operation, arg1, arg2));
  }

  public <A1, A2, A3> void addCompensation(
      Functions.Proc3<A1, A2, A3> operation, A1 arg1, A2 arg2, A3 arg3) {
    compensationOps.add(() -> Async.procedure(operation, arg1, arg2, arg3));
  }

  public <A1, A2, A3, A4> void addCompensation(
      Functions.Proc4<A1, A2, A3, A4> operation, A1 arg1, A2 arg2, A3 arg3, A4 arg4) {
    compensationOps.add(() -> Async.procedure(operation, arg1, arg2, arg3, arg4));
  }

  public <A1, A2, A3, A4, A5> void addCompensation(
      Functions.Proc5<A1, A2, A3, A4, A5> operation, A1 arg1, A2 arg2, A3 arg3, A4 arg4, A5 arg5) {
    compensationOps.add(() -> Async.procedure(operation, arg1, arg2, arg3, arg4, arg5));
  }

  public <A1, A2, A3, A4, A5, A6> void addCompensation(
      Functions.Proc6<A1, A2, A3, A4, A5, A6> operation,
      A1 arg1,
      A2 arg2,
      A3 arg3,
      A4 arg4,
      A5 arg5,
      A6 arg6) {
    compensationOps.add(() -> Async.procedure(operation, arg1, arg2, arg3, arg4, arg5, arg6));
  }
```



### 补偿操作

compensate() 的实现根据是否要并行补偿分为两种情况，

```java
  public void compensate() {
    if (options.parallelCompensation) {
      // 并行执行所有的补偿操作，这里会有多个线程同时工作
      ......
    } else {
      ......
      // 不并行，则意味着单线程处理，会按照登记的顺序逆序进行逐个补偿操作
      // 逐个补偿时，如果遇到报错，就会按照 continueWithError 来决定是否跳过错误继续执行后续的补偿操作
    }
```

### 并行补偿

代码实现为：

```java
List<Promise> results = new ArrayList<>();
// 将登记的每个补偿方法取出来
for (Functions.Func<Promise> f : compensationOps) {
  // 调用其 apply() 方法执行补偿方法，注意 apply() 方法返回的是 Promise
  results.add(f.apply());
}

// 准备 sagaException备用
CompensationException sagaException = null;
for (Promise p : results) {
  // 对于每个 Promise
  try {
    // 等待其执行的结果
    p.get();
  } catch (Exception e) {
    // 如果发生错误（抛出Exception）
    if (sagaException == null) {
      // 如果是第一次抛出，则构建 CompensationException 对象
      sagaException = new CompensationException(e);
    } else {
      // 如果CompensationException 对象不为空，表示不是第一次，意味着之前已经有异常抛出
      // 则 append 新的异常到已有的 CompensationException
      sagaException.addSuppressed(e);
    }
  }
}

if (sagaException != null) {
  // 最后检查，如果有 CompensationException 则抛出
  // 如果没有，当然是表明所有的补偿操作都顺利执行
  // 注意在并行处理过程中，continueWithError 是被忽略的
  // 因为都一起并行执行了，也就不存在一个补偿操作发生错误要不要跳过下一个补偿的逻辑判断
  throw sagaException;
}
```

不行补偿可以保证每个补偿操作都会被执行，但是无法保证执行的顺序。

## 逐个补偿

代码实现为：

```java
      for (int i = compensationOps.size() - 1; i >= 0; i--) {
        // 逆序取出每一个保存的补偿操作
        Functions.Func<Promise> f = compensationOps.get(i);
        
        try {
          // 同样调用其 apply() 方法执行补偿方法，注意 apply() 方法返回的是 Promise
          Promise result = f.apply();
          // 直接调用 Promise 的 get() 方法来等待并得到执行结果
          result.get();
        } catch (Exception e) {
          // 如果补偿操作发生错误，抛出异常
          if (!options.continueWithError) {
            // 则根据 continueWithError 判断是否继续执行下一个补偿操作（抛出异常就会退出for循环）
            // continueWithError = true, 不抛出异常
            // continueWithError = false, 抛出异常
            throw e;
          }
        }
      }
```

逐个补偿可以保证执行的顺序（和登记的顺序相反），如果设置 continueWithError 为 true 则可以保证每个补偿操作都会被执行（缺点是抛出的异常会被吃掉，而且 temporal 连日志都不打印）。如果没有设置 continueWithError 为 true （默认为false），则中途有补偿操作失败时，后续的补偿操作将不会被执行，这里需要特别小心。

