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
- JobGroupCancelled
- AllJobCancelled
- ExecutorGained
- ExecutorLost
- BeginEvent
- GettingResultEvent
- CompletionEvent
- TastSetFailed
- ResubmitFailedStages
- StopDAGScheduler

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

## TaskScheduler
## TaskSet

	