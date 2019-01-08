---
title: flowable源码解析 —— Command
urlname: sourece/flowable/command
categories:
  - source
  - flowable
date: 2018-12-13 18:27:28
tags:
---

# 命令Command

命令Command是flowable中很重要的一个概念，是设计模式中的命令模式一种运用。

Command是一个接口，其定义如下

```java
// Command.java
public interface Command<T> {
    T execute(CommandContext commandContext);
}
```

在flowable中，几乎所有的操作都是以command的形式进行的。因此，了解flowable是如何通过command来执行各种操作是非常重要的。

## TODO

为什么要以命令模式设计？

将操作封装成命令之后，就可以很方便地通过拦截器来对命令进行aop。

## 命令执行器CommandExecutor

CommandExecutor接口定义如下

```java
// CommandExecutor.java
public interface CommandExecutor {
    /**
     * 获取默认的配置
     */
    CommandConfig getDefaultConfig();

    /**
     * 通过指定的配置执行命令
     */
    <T> T execute(CommandConfig config, Command<T> command);

    /**
     * 通过默认的配置执行命令
     */
    <T> T execute(Command<T> command);
}
```

在flowable中，几乎所有的command都是通过CommandExecutor来执行的，例如

```java
// RuntimeServiceImpl.java
@Override
public ProcessInstance startProcessInstanceByKey(String processDefinitionKey) {
    return commandExecutor.execute(new StartProcessInstanceCmd<ProcessInstance>(processDefinitionKey, null, null, null));
}
```

CommandExecutor采用了拦截过滤器模式来实现。它内部有一个拦截器链，链上的每个节点都实现了CommandInterceptor（定义如下）。最后一个interceptor一般都是CommandInvoker，在真正地执行command之前会先执行之前的所有interceptor，例如LogInterceptor, TransactionInterceptor等，具体取决于配置。

```java
// CommandInterceptor.java
public interface CommandInterceptor {
    // 执行命令
    <T> T execute(CommandConfig config, Command<T> command);
    // 获取下一个拦截器
    CommandInterceptor getNext();
    // 设置下一个拦截器
    void setNext(CommandInterceptor next);
}
```

可以简单地认为CommandExecutor的设计在此是为了实现AOP，对每一个命令的执行进行切面处理。

## CommandInvoker

CommandInvoker是最终调用command的interceptor，不过这个调用过程还有点复杂。这里涉及到另一个概念——[议程Agenda](#议程Agenda)，如果不清楚议程是什么请先点击链接了解。

先看看CommandInvoker的源码

```java
public <T> T execute(final CommandConfig config, final Command<T> command) {
    final CommandContext commandContext = Context.getCommandContext();

    FlowableEngineAgenda agenda = CommandContextUtil.getAgenda(commandContext);
    if (commandContext.isReused() && !agenda.isEmpty()) {
        return (T) command.execute(commandContext);
    } else {
        // Execute the command.
        // This will produce operations that will be put on the agenda.
        // 将要执行的操作提到议程中
        agenda.planOperation(new Runnable() {
            @Override
            public void run() {
                // 在这个操作中执行command并将结果保存到command context中
                commandContext.setResult(command.execute(commandContext));
            }
        });

        // Run loop for agenda
        // 执行议程中的所有操作
        executeOperations(commandContext);

        // At the end, call the execution tree change listeners.
        // TODO: optimization: only do this when the tree has actually changed (ie check dbSqlSession).
        if (!commandContext.isReused() && CommandContextUtil.hasInvolvedExecutions(commandContext)) {
            agenda.planExecuteInactiveBehaviorsOperation();
            executeOperations(commandContext);
        }

        return (T) commandContext.getResult();
    }
}
```

上面代码中，重点的方法有两个：将执行命令的操作提到议程中的agenda.planOperation()，以及执行议程中的所有操作的executeOperations()。

planOperation()比较简单，只是简单地将其添加到队列中。代码如下

```java
// AbstractAgenda.java
protected LinkedList<Runnable> operations = new LinkedList<>();

@Override
public void planOperation(Runnable operation) {
    operations.add(operation);
    if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Operation {} added to agenda", operation.getClass());
    }
}
```

重点来看下executeOperations()

```java
protected void executeOperations(final CommandContext commandContext) {
    while (!CommandContextUtil.getAgenda(commandContext).isEmpty()) {
        Runnable runnable = CommandContextUtil.getAgenda(commandContext).getNextOperation();
        executeOperation(runnable);
    }
}

public void executeOperation(Runnable runnable) {
  if (runnable instanceof AbstractOperation) {
      AbstractOperation operation = (AbstractOperation) runnable;

      // Execute the operation if the operation has no execution (i.e. it's an operation not working on a process instance)
      // or the operation has an execution and it is not ended
      if (operation.getExecution() == null || !operation.getExecution().isEnded()) {
          if (LOGGER.isDebugEnabled()) {
              LOGGER.debug("Executing operation {}", operation.getClass());
          }
          runnable.run();
      }
  } else {
      runnable.run();
  }
}
```

在上面代码中，循环语句用的是while而不是for each，是因为在议程的操作执行的过程中有可能产生新的operation，而这些operation也应该接着被执行。

另外需要注意到，议程操作是直接通过Runnable.run()来**同步执行**的，所以不要看到议程操作的类型是Runnable就觉得是通过另外的线程异步执行的。

### 议程Agenda

每一个命令被执行的时候都会创建一个议程实例，所有的操作都会被置入这个议程实例中，然后由命令执行器一直执行到议程中没有其它的命令为止。在命令执行期间，总是可以通过Context.getAgenda()获取到议程实例。

TODO:: 这一章的层级

#### 疑问TODO

- 为什么需要议程这种设计呢？
- agenda实例是在哪里创建的

## 命令上下文CommandContext

## 其它

### 命令执行器是如何构造的






