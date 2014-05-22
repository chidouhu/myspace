# Scheduler
## DAGScheduler
DAGScheduler实现了面向stage的调度层. 它将每个job的stage划分成DAG, 追踪RDD和stage输出, 找出一个可以运行作业的最小化调度方法. 接下来就将每个stage以TaskSet提交给TaskScheduler执行.

为了构造出stage的DAG图, 该类还需要根据当前的cache状态, 计算出每个task优先跑在那个节点上, 并将该location传递给低层的TaskScheduler. 此外, 它需要处理shuffule输出文件丢失的异常, 此时前一个stage可能会被重新提交. 不是由shuffle文件丢失引起的错误将会由TaskScheduler处理, 这些task在取消整个stage之前会被重试多次.

***THREADING: 该类的所有逻辑操作都是在执行run()方法的单线程完成的, 事件都被提交到一个同步队列(eventQueue)中. public的API方法, 如runJob, taskEnded和executorLost, 都会异步的把事件放入到这个队列中. 所有其他的方法都应该是private的.***

先看start方法. start方法启动了eventProcessorActor, 该actor有两个职责:
- 等待事件, 如作业提交, 作业结束, 作业失败等. 调用[[org.apache.spark.scheduler.DAGScheduler.processEvent()]]来处理事件.
- 调度周期task来重新提交失败的stage.

***注意: 该actor不能在构造函数里启动, 因为周期任务参考了一些内部状态的封闭[[org.apache.spark.scheduler.DAGScheduler]]对象, 因此在[[org.apache.spark.scheduler.DAGScheduler]完全构造好之前不能调度.***

	def start() {
		eventProcessActor = env.actorSystem.actorOf(Props(new Actor {
			
			def receive = {
				case event: DAGSchedulerEvent =>
					logTrace("Get event of type " + event.getClass.getName)

					/**
					 * All events are fowarded to `processEvent()`, so that the event processing logic can
					 * easily tested without starting a dedicated actor. Please refer to `DAGSchedulerSuite`
					 * for details.
					 * /
					if (!processEvent(event)) {
						submitWaitingStages()
					} else {
						context.stop(self)
					}
			}
		}))
	}

可见processEvent是所有事件的入口, 需要注意一下这个函数的返回值, 如果是true表示事件循环可以结束了. processEvent主要针对不同的事件做不同的处理:

- JobSubmitted
	- newStage 创建一个Stage, 要么直接用于result stage, 或者用于newOrUsedStage中的shuffle map stage的创建的一部分. 注意, 如果要生成shuffle map stage, 那么一定要用newOrUsedStage方法, 而不是直接使用newStage. 该方法的实现其实就是new一个Stage和一个StageInfo, 并且更新一堆map.

	- newActiveJob

	- 如果允许本地启动, 那么向listenerBus post一个SparkListenerJobStart消息, 再调用runLocally方法
		- listenerBus是什么? 其实是一个异步地向SparkListeners传递SparkListenerEvents事件的工具. 后面单独介绍.
		- runLocally方法主要是用于在本地的一个RDD上运行一个作业, 假设它只有一个partition并且没有依赖. 我们会另起一个线程来完成这个操作, 这样就不会阻塞住DAGScheduler的事件循环.
		- runLocally的实际工作线程主要就是调用job的func方法,调用成功后向job的listener发送taskSucceeded消息, 执行taskContext的executeOnCompleteCallbacks. 如果在过程中出现了异常, 就会向job的listener发送jobFailed消息, 最终不论成功与否, 都会向listenerBus发送SparkListenerJobEnd消息.


	- 如果非本地, 那么还是向listenerBus post一个SparkListenerJobStart消息, 再调用submitStage方法
		- submitStage方法首先调用activeJobForStage方法, 如果返回的jobID已定义且stage不在waiting, running, failed的集合里, 则首先获取所有的missingParentStage, 有missing的话先递归提交missing的, 没有的话调用submitMissingTasks
		- submitMissingTasks(父stage已经全都执行完了). 首先取出所有的pendingTask, 接下来根据stage类型进行操作, 如果stage是ShuffleMap, 则new出一系列ShuffleMapTask; 否则这就是final stage, 先从resultStageToJob找到所有的job, 接下来new出一系列ResultTask. 这时候会先给listenerBus发送一个SparkListenerStageSubmitted消息, 然后会调用SparkEnv.get.closureSerializer.newInstance().serialize(tasks.head), 先把task序列化好, 最后就调用taskSched.submitTasks(new TaskSet(...))

	
- JobCancelled
	- handleJobCancellation(jobId)方法.
		- removeJobAndIndependentStage. 删除所有和其他作业或者stage无关的job. 返回所有要删除的stage id. 所有跟这些stage相关的task都要被删除. 该方法主要就是找到独立Stage的task.
		- 对前面获得的所有独立的task, 调用taskSched.cancelTasks
		- 接下来通知job.listener该job已被cancel, 并向listenerBus发送SparkListenerJobEnd消息

	
- JobGroupCancelled
	- 找到所有与该group id相关的active job, 依次调用handleJobCancellation
	
- AllJobCancelled
	- 对running map中的所有job调用handleJobCancellation

- ExecutorGained
	- handleExecutorGained(execId, host)方法. 其实就是前面已经lost的executor又找到了, 从failedEpoch中移除

- ExecutorLost(TODO)
	- handleExecutorLost方法. 主要针对exector丢失的情况. 由于该方法在事件循环中调用, 所以它可以改变scheduler内部的状态. 外部通过调用executorLost方法来push一个lost事件. 这里有一个可选择的maybeEpoch参数, 如果已经捕捉到错误可以直接传入, 避免触发抓取错误的逻辑. 
		- 首先从blockManagerMaster中移除该executor, 接下来对shuffleToMaoStage中所有的stage调用removeOutputsOnExecutor, 接下来从stage的outputLocs中取出locs注册到mapOutputTracker.


- BeginEvent
	- 该事件是由TaskScheduler调用用来汇报task启动信息的. 对task的stage做检测, 如果taskInfo的serializedSize过大或者stageInfo.emittedTaskSizeWarning预警, 会打印warning日志, 接下来就向listenerBus发送SparkListenerTaskStart消息.


- GettingResultEvent
	- 向listenerBus发送SparkListenerTaskGettingResult消息.


- CompletionEvent
	- 向listenerBus发送SparkListenerTaskEnd消息. 然后调用handleTaskCompletion方法.
	- handleTaskCompletion方法(TODO):


- TastSetFailed
	- 对taskSet里所有的stageId, 调用abortStage方法.
	- abortStage(TODO)


- ResubmitFailedStages
	- 首先判断failed.size是否大于0, 这里做判断是因为可能failed stage已经被job cancellation移除了, 因此即使这里触发了ResubmitFailedStages, failed也有可能为空. 接下来就直接调用resubmitFailedStages
	- resubmitFailedStages(TODO)


- StopDAGScheduler
	- 取消所有的active job. 对所有的active job, 调用job listener的jobFailed接口, 并向listenerBus发送SparkListenerJobEnd消息.


## SparkListenrBus
主要成员变量有
	
	// 一个SparkListener的数组
	private val sparkListeners = new ArrayBuffer[SparkListener] with SynchronizedBuffer[SparkListner]
	// 一个事件队列
	private val eventQueue = new LinkedBlockingQueue[SparkListenerEvents](EVENT_QUEUE_CAPACITY)

主线程(守护)一直尝试从事件队列里取出事件, 然后对每个SparkListener执行该事件.

主要方法:
post方法就是往事件队列里面push事件

	def post(event: SparkListenerEvents) {
		val eventAdded = eventQueue.offer(event)
		if (!eventAdded && !queueFullErrorMessageLogged) {
			queueFullErrorMessageLoged = true
		}
	}

## SparkListener
首先定义了一系列SparkListenerEvents:

- SparkListenerStageSubmitted

- SparkListenerStageCompleted

- SparkListenerTaskStart

- SparkListenerTaskGettingResult

- SparkListenerTaskEnd

- SparkListenerJobStart

- SparkListenerJobEnd

特质SparkListener主要定义了一堆onCompletion接口.

- onStageCompleted

- onStageSubmitted

- onTaskStart

- onTaskGettingResult

- onTaskEnd

- onJobStart

- onJobEnd

StatsReportListener类主要记录了当每个stage完成时的一些统计信息, 因此只实现了一个onStageCompleted接口.伴生对象StatReportListener实现了一堆metrics获取的方法.

## TaskScheduler && TaskSchedulerImpl
TaskScheduler提供了更底层的task调度接口, 目前是由ClusterScheduler实现的. 这些接口定义允许用户自定义可插拔的task调度器. 每一个TaskScheduler为一个独立的SparkContext调度任务. 这些调度器会从DAGScheduler的每个stage获取一个task集合, 然后将这些task发送给集群, 运行, 在发生错误时进行重试, 并且***减轻落后者***. 最终以事件的形式将结果反馈给DAGScheduler.
主要成员变量与接口:

- rootPool

- schedulingMode 调度模式

- start

- postStartHook 在系统成功初始化后会被调用(特别是spark context初始化后). Yarn可以用这个接口来bootstrap基于preferred location的资源分配, 等待slave注册等.

- stop 和集群断开连接

- submitTasks 提交一系列task来运行

- cancelTasks 取消某个stage

- setDAGScheduler

- defaultParallelism 获取该集群的默认并发数

TaskSchedulerImpl是TaskScheduler接口的一个实现. 通过SchedulerBackend来为多类cluster调度任务. 也可以通过配置使用LocalBackend. 它主要负责普通逻辑, 如决定作业之间的调度顺序, 唤醒speculative任务等. Clients必须先调用initialize和start, 然后通过runTasks方法提交task集合.

***THREADING: SchedulerBackends和提交任务的客户端可以从不同的线程中调用该类, 所以在public API方法中需要加锁. 此外, 一些SchedulerBackends在发送事件时可以在这里进行同步, 它们在这里会获取锁, 所谓我们必须确保在持有锁时不会尝试锁住backend***

成员变量和接口:

- SPECULATION_INTERVAL 检测speculative任务的频率

- STARVATION_TIMEOUT 警告用户饥饿时间的上限

- activeTaskSets: HashMap[String, TaskSetManager] TaskSetManager不是线程安全的, 所以在这里需要同步.

- taskIdToTaskSetId: HashMap[Long, String]

- taskIdToExecutorId: HashMap[long, String]

- nextTaskId: AtomicLong 每次自增的task IDs

- activeExecutorIds: HashSet[String]

- executorsByHost: HashMap[String, HashSet[String]] 每个host上拥有的executor集合; 这个变量用于计算hostsAlive, 其返回值用户决定是否在给定host上可以获得数据locality.

- executorIdToHost: HashMap[String, String]

- dagScheduler: DAGScheduler

- backend: SchedulerBackend

- mapOutputTracker: SparkEnv.get.mapOutputTracker

- schedulableBuilder: SchedulableBuilder

- rootPool: Pool

- schedulingMode: SchedulingMode

- taskResultGetter: TaskResultGetter

- initialize: 初始化, 设置backend, new出rootPool, 创建schedulableBuilder并调用schedulableBuilder的buildPools方法

- start: 启动backend, 如果有speculation需求, 启动speculative executor thread.

- submitTasks: 加锁.
	- 首先new出一个TaskSetManager用于管理这一组task, 在该记录的数据结构里记下这组映射, 并且在schedulableBuilder里addTaskSetManager

	- 接下来等待hasLaunchedTask为true, 如果在饥饿时间内还没有为true(就是没有分配到资源)就报警.

	- 调用backend的reviveOffers

- cancelTasks: 加锁(传入的是stageId)
	- 首先从activeTaskSets中查找taskSetManager, 这里有两种情况, 一种是taskSetManager已经创建并且已经调度了一些任务, 这种情况下需要给executor发送signal信号来kill task并且终止该stage; 另一种是还没有调度任务, 那只要直接abort这个stage就好了

	- 如果有runningTasksSet, 调用backend的killTask

	- 调用taskSetManager的abort终止stage.

- taskSetFinished: 加锁. 当taskSetManager管理的所有task attempt完成时调用该方法. 从activeTaskSets里面移除task set, 从该manager的父manager中移除该manager.

- resourceOffers: 加锁. 由cluster manager调用, 提供slave的资源. 我们按照优先级顺序从active task里取出task, 然后以round-robin的形式把这些task分配到cluster上以保证均衡.
	- 记录汇报上来的资源, 放到各个map里去

	- 创建一个待分配的task列表. 

	- 从排好序的task列表中取出task, 然后按照locality层增序提供给每个计算节点, 这样可以让大家都有机会运行本地task.***重要***

- statusUpdate: 加锁. 更改某个tid的状态.
	- 如果要改成LOST状态, 首先从activeExecotrIds从取出, 并且加入到failedExecutor中.

	- 找到对应的TaskSet, 如果是正常结束, 调用taskResultGetter.enqueueSuccessfulTask, 如果是异常结束, 则调用taskResultGetter.enqueueFailedTask

	- 如果有failedExecotr, 需要通知DAGScheduler, 并且调用backend的reviveOffers. ***注意这里在锁外, 如果在锁内会有死锁***

- handleTaskGettingResult
	- 裸调taskSetManager的对应方法

- handleSuccessfulTask
	- 裸调taskSetManager的对应方法

- handleFailedTask
	- 裸调taskSetManager的对应方法

	- 如果taskManager已经进入了Zombie状态, 需要通知backend的reviveOffers.

TaskSchedulerImlp的伴生对象, 仅提供了一个prioritizeContainers方法. 这个方法是用于跨机器均衡containers的. 该方法传入一个host和host拥有资源的map, 返回一个资源分配的顺序列表. 注意传入的host资源已经是有序的, 因此我们在同一台机器上我们优先分配前面的container.

示例: 给定 <h1, [o1, o2, o3]>, <h2, [o4]>, <h1, [o5, o6]>, 返回[o1, o5, o4, o2, o6, o3].

## SchedulerBackend
调度系统的后台接口, 允许在ClusterScheduler下挂载不同的调度器. 我们假定Mesos-like的系统, 可以从机器上获取资源并在上面运行任务.
接口:
- start

- stop

- reviveOffers

- defaultParallelism

- killTask

## Schedulable
特质, 定义了可调度实体的接口, 实际有两类可调度实体, 分别是Pools和TaskSetManagers.

## Pool
- schedulableQueue: ArrayBuffer[Scheduable] 记录所有的Schedulable

- schedulableNameToSchedulable: HashMap[String, Schedulable] 记录所有Schedulable和名字的映射

后面提供的一些接口就是对这两个数据集的增删改查.

## TaskSetManagers(TODO)

## SchedulableBuilder
特质, 构建Schedulable tree的接口, 总共就两个接口:
- buildPools 用来构建所有树的节点

- addTaskSetManager 构建树的叶子节点(TaskSetManager)

针对该特质提供了两个实现:
- FIFOSchedulableBuilder
	- buildPools 空

	- addTaskSetManager 直接调用rootPool的addSchedulable接口

- FairSchedulableBuilder
	- buildPools  从配置文件中getResourceAsStream, 挨个调用buildFairSchedulerPool, 最后调用buildDefaultPool
		- buildDefaultPool new出一个Pool来加入rootPool

		- buildFairSchedulerPool 根据配置文件new出Pool加入rootPool

	- addTaskSetManager 从rootPool中取出所有符合要求的parentPool, 如果没有则new出一个新的, 接下来把manager加到parentPool中去.

## TaskResultGetter
## TaskSet
## Task
## TaskInfo

## Stage
Stage是一个Spark job中运行相同计算逻辑的一组独立task的集合, 这些task的shuffle依赖都相同. 每个task的DAG都被切分成不同的stage, 不同的stage是以shuffle发生为边界的. DAGScheduler会以逻辑顺序运行这些stage.

Stage就分两种, 一种是shuffle map stage, 这类task的结果是另一阶段task的输入; 另一种是result stage, 这类task可以直接完成计算行为如初始化job(e.g. count(), save(), etc). 对于shuffle map stage, 我们还要追踪每个输出partition在哪些节点上.

每个Stage都有一个jobId, 可以和提交该stage的job做区分. 在使用FIFO调度时, 这使得先来job的stage可以先被计算或者快速回复.

- isAvailable 判断是否可用. 如果是shuffle map stage则直接返回true, 而如果是result stage则需要所有可用输出等于paritition数目

- addOutputLoc && removeOutputLoc outputLocs是一个Array.fill[List[MapStatus]], 记录每个partition的MapStatus列表

- removeOutputsOnExecutor(TODO)
- newAttemptId(TODO)

## StageInfo
保存所有从scheduler发送给SparkListener的stage信息. 其中taskInfos保存了所有已完成的task的metrics信息, 包括冗余的, 特殊的task. 两个元素, stage和taskInfos. 其中taskInfos是mutable.Buffer[(TaskInfo, TaskMetrics)]类型.
	