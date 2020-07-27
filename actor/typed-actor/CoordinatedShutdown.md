# Coordinated Shutdown (协调停机)

正常情况下，一个ActorSystem被停止或者JVM进程关闭时，下面的actor和服务会按照特定的顺序被停止

CoordinatedShutdown 扩展注册在关闭过程中要执行的用户自定义任务，这些任务会按照配置定义的 "阶段"分组。这些阶段定义了关闭的顺序

关闭阶段的顺序在配置 `akka.coordinated-shutdown.phase` 中定义

| Phase      | Description   | 
| ---------- | ---------| 
| before-service-unbind | 关机期间的第一个预定义阶段 | 
| before-cluster-shutdown  | 用于在服务关闭后、集群关闭前要运行的自定义应用任务的阶段 | 
| before-actor-system-terminate  | 用于在群集关闭后和 ActorSystem 终止前运行的自定义应用任务的阶段 | 

reference.conf

```
# CoordinatedShutdown is enabled by default and will run the tasks that
# are added to these phases by individual Akka modules and user logic.
#
# The phases are ordered as a DAG by defining the dependencies between the phases
# to make sure shutdown tasks are run in the right order.
#
# In general user tasks belong in the first few phases, but there may be use
# cases where you would want to hook in new phases or register tasks later in
# the DAG.
#
# Each phase is defined as a named config section with the
# following optional properties:
# - timeout=15s: Override the default-phase-timeout for this phase.
# - recover=off: If the phase fails the shutdown is aborted
#                and depending phases will not be executed.
# - enabled=off: Skip all tasks registered in this phase. DO NOT use
#                this to disable phases unless you are absolutely sure what the
#                consequences are. Many of the built in tasks depend on other tasks
#                having been executed in earlier phases and may break if those are disabled.
# depends-on=[]: Run the phase after the given phases
phases {

  # The first pre-defined phase that applications can add tasks to.
  # Note that more phases can be added in the application's
  # configuration by overriding this phase with an additional
  # depends-on.
  before-service-unbind {
  }

  # Stop accepting new incoming connections.
  # This is where you can register tasks that makes a server stop accepting new connections. Already
  # established connections should be allowed to continue and complete if possible.
  service-unbind {
    depends-on = [before-service-unbind]
  }

  # Wait for requests that are in progress to be completed.
  # This is where you register tasks that will wait for already established connections to complete, potentially
  # also first telling them that it is time to close down.
  service-requests-done {
    depends-on = [service-unbind]
  }

  # Final shutdown of service endpoints.
  # This is where you would add tasks that forcefully kill connections that are still around.
  service-stop {
    depends-on = [service-requests-done]
  }

  # Phase for custom application tasks that are to be run
  # after service shutdown and before cluster shutdown.
  before-cluster-shutdown {
    depends-on = [service-stop]
  }

  # Graceful shutdown of the Cluster Sharding regions.
  # This phase is not meant for users to add tasks to.
  cluster-sharding-shutdown-region {
    timeout = 10 s
    depends-on = [before-cluster-shutdown]
  }

  # Emit the leave command for the node that is shutting down.
  # This phase is not meant for users to add tasks to.
  cluster-leave {
    depends-on = [cluster-sharding-shutdown-region]
  }

  # Shutdown cluster singletons
  # This is done as late as possible to allow the shard region shutdown triggered in
  # the "cluster-sharding-shutdown-region" phase to complete before the shard coordinator is shut down.
  # This phase is not meant for users to add tasks to.
  cluster-exiting {
    timeout = 10 s
    depends-on = [cluster-leave]
  }

  # Wait until exiting has been completed
  # This phase is not meant for users to add tasks to.
  cluster-exiting-done {
    depends-on = [cluster-exiting]
  }

  # Shutdown the cluster extension
  # This phase is not meant for users to add tasks to.
  cluster-shutdown {
    depends-on = [cluster-exiting-done]
  }

  # Phase for custom application tasks that are to be run
  # after cluster shutdown and before ActorSystem termination.
  before-actor-system-terminate {
    depends-on = [cluster-shutdown]
  }

  # Last phase. See terminate-actor-system and exit-jvm above.
  # Don't add phases that depends on this phase because the
  # dispatcher and scheduler of the ActorSystem have been shutdown.
  # This phase is not meant for users to add tasks to.
  actor-system-terminate {
    timeout = 10 s
    depends-on = [before-actor-system-terminate]
  }
}
```


如果需要的话，在 `application.conf` 中可以添加更多的 phase, 方法是用附加的 depends-on 覆盖一个 phase

返回的Future[Done]应在任务完成时完成。任务名称参数仅用于调试/日志。

添加到同一阶段的任务是并行执行的，没有任何顺序假设。在上一阶段的所有任务完成之前，下一阶段不会开始。

如果任务没有在配置的超时时间内完成(参见 reference.conf)，下一阶段将被启动。可以为一个阶段配置 recover=off，以便在任务失败或未在超时内完成的情况下，中止关闭过程的其余部分。

如果需要取消之前添加的任务。


要启动协调关机进程，可以使用ActorSystem调用terminate() 或者 扩展CoordinatedShutdown 
并将一个实现CoordinatedShutdown.Reason的类传递给它

ActorSystem将在最后阶段被终止，默认情况下JVM不会被强制停止(如果所有非守护进程的线程都被终止，它将被停止)。
要启用 System.exit 为最后一个执行动作，可以配置:
`akka.coordinated-shutdown.exit-jvm = on`


一旦actor系统的根actor被停止，协调关机进程也会被启动。

在使用Akka集群时，当集群节点将自己视为Exiting时，CoordinatedShutdown将自动运行，即从另一个节点离开将触发离开节点上的关机进程。当使用Akka Cluster时，会自动添加用于优雅地离开集群的任务，包括优雅地关闭集群单子和集群共享，也就是说，如果关闭过程还没有进行，运行关闭过程也会触发优雅地离开。

默认情况下， CoordinatedShutdown 将在JVM进程退出时运行，可以通过以下配置禁用:
`akka.coordinated-shutdown.run-by-jvm-shutdown-hook=off`

对于某些测试来说通过CoordinatedShutdown来终止actor系统可能是不可取的,可以通过以下配置来禁用:
```
# Don't terminate ActorSystem via CoordinatedShutdown in tests
akka.coordinated-shutdown.terminate-actor-system = off
akka.coordinated-shutdown.run-by-actor-system-terminate = off
akka.coordinated-shutdown.run-by-jvm-shutdown-hook = off
akka.cluster.run-coordinated-shutdown-when-down = off
```
