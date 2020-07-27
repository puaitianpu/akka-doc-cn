# Fault Tolerance (actor 的容错)

验证错误是指发送给actor的命令数据无效，这应该作为actor消息协议的一部分进行建模，而不是让actor抛出异常。

失败则是指一些意料之外的事情，或者是在actor本身控制之外的事情，例如数据库连接中断。
不能将失败作为消息协议的一部分来建模，因为发送消息的actor很少能对它做任何有用的处理

对于故障，应用使用"让它崩溃 "的理念

## Supervision (监督)

Supervision 允许使用声明性的描述当某个异常类型在actor内部被抛出时应该怎么处理

默认的监督策略是在抛出异常时停止该角色，Actor行为是使用Behaviors.supervise包装的
```
Behaviors.supervise(behavior).onFailure[IllegalStateException](SupervisorStrategy.restart)
```

## Wrapping behaviors (包装行为)

通过改变behavior的状态来处理，每一个返回的behavior都会被上级的actor重新包装
这样只需要在顶层添加监督策略


## Child actors are stopped when parent is restarting (父actor重启时停止子actor)

子角色通常是在设置块中启动的，当父角色被重新启动时，子角色会被再次运行。停止子角色是为了避免每次重启父角色时创建新的子角色的资源泄漏

如果需要重启父actor时不停止子actor, 使用 `SupervisorStrategy.restart.withStopChildren(false)`
这样的话 `setup` block 只是在第一次启动的时候运行，后面重启不会在运行


## The PreRestart signal (预重启信号)

受监督的actor被重新重启之前，它会被发送PreRestart信号，可以用来清理资源
actor停止时会发送PostStop信号，重启不会发送PostStop


## Bubble failures up through the hierarchy (层层递进的失败)

在某些情况下，可能需要在actor中向上推送关于在失败时做什么的决定，并让父actor处理失败时应该发生的事情（在经典的Akka Actors中，这是默认的工作方式）。

要想让父actor在子actor被终止时得到通知，它必须观察子actor。如果子actor是因为失败而被停止，会收到一个ChildFailed信号，
它将包含失败原因。ChildFailed扩展了Terminated，所以如果你的用例不需要区分停止和失败，你可以用Terminated信号处理这两种情况。

如果父actor反过来不处理Terminated消息，它本身就会以akka.actor.typed.DeathPactException失败。

这意味着，一个层次的actor可以有一个子actor的失败冒出来，使得每一个在执行的actor都停下来，
但会告知最上面的父actor有一个失败，以及如何处理它，引起失败的原始异常可以直接让父actor开箱即用（这在大多数情况下是一件好事，而不是泄露实现细节）。

可能有的情况下，你希望原始异常在层次结构中冒出来，这可以通过处理Terminated信号，
并在每个actor中重新抛出异常来实现。




 