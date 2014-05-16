#Akka
## Hello World

## Akka手册
### 概述
#### 2.1 术语和概念
##### 2.1.1 并发与并行
并发和并行是类似的概念,但是有一些不同.***并发***意味着两个或多个task是有进度的,它们可能并不是同时执行的.一个简单的例子是通过时间片切分让task的某些部分和其它task的部分顺序执行.而并行则是真正意义上的同时执行.

##### 2.1.2 异步 vs. 同步
如果调用者在函数调用返回或者抛出异常之前不能继续执行的话,该调用方法就是***同步***的.相反,异步调用允许调用者在有限的步骤后继续执行,在方法结束后可以通过一些附加机制来被唤醒(可以注册回调,future或者message).

同步API可能会用阻塞来实现同步,但这不是必须的.CPU密集型任务的行为可能会和阻塞相似.总的来说,还是建议用户使用异步API,只要他们确保系统可以继续执行. Actor模式天生就是异步的: 一个actor可以在发送完一个消息后继续执行别的任务,而不需要等到发送行为真正发生.

##### 2.1.3 非阻塞 vs. 阻塞
如果一个线程的延迟可以无限期的影响到其他线程,这就是阻塞. 一个很好的例子是被某个线程互斥使用的资源. 如果一个线程无限期持有该资源(比如运行在一个无线循环中)而其他等待该资源的线程则无法继续执行.相反的, 非阻塞意味着没有线程可以无限期的延迟别的线程.

相比阻塞操作, 我们更倾向于非阻塞操作, 因为如果包含阻塞操作的话, 系统的整体运行无法被保证.

##### 2.1.4 死锁 vs. 饥饿 vs. 活锁
死锁是在几个参与者互相等待达到某一个状态而无法继续运行的时候形成的. 在其他参与者没有达到某一种状态的时候, 没有人可以继续执行, 最终影响整个系统. 死锁和阻塞的概念很相关, 需要某个参与线程能够无限期延迟其他线程的执行.

在死锁的情况下, 没有参与者可以继续进行. 相反的是***饥饿***现象, 这时候有某些参与者可以继续执行, 但是有些却永远不能. 典型的场景是一个简单的调度算法, 每次都优先选择优先级高的作业. 如果高优先级的作业持续到来的话, 那么低优先级的作业就永远没有执行的机会.

活锁和死锁类似, 也是没有参与者可以继续运行. 区别在于活锁在等待其他线程处理的状态中不是被冻结, 而是参与者不断改变状态. 一个例子是当两个参与者分别有两份独立资源的时候. 他们互相试图获取资源, 但是它们也互相检测对方时候需要资源. 如果某个资源被另一个参与者请求了, 他们就试图获取另外一份实例. 在极端情况下会出现两个参与者在两份资源之间来回跳跃, 却永远不会获取, 总是让给另外一个人.

##### 2.1.5 竞争条件
当一系列事件集合的顺序可能被外部非确定影响所违反时, 我们称之为竞争条件. 竞争条件通常在多个线程使用一个共享可变状态的时候发生, 线程在某个状态的操作可能会被非预期的行为所交错. 共享状态并不是产生竞争条件的必要条件. 一个例子是客户端发送无序的包P1, P2给服务器. 因为包可能会走不同的网络路由, 因为服务器可能会先收到P2再收到P1. 如果没有包含它们之间的顺序信息, 那么服务端就不可能知道它们发送的顺序错乱了. 在这种情况下就会导致竞争条件.

*** 注意: Akka为消息传递提供的唯一保证就是一对actor之间的消息始终是保序的. ***

##### 2.1.6 非阻塞保证(过程条件)
如前面章节所述阻塞通常是导致几种异常的原因, 包括死锁和系统吞吐降低. 在下面几节我们讨论几种非阻塞的属性.

###### Wait-freedom
如果每次调用可以确保在有限的步骤内完成则称之为wait-free. 如果这些步骤有一个上界则称之为bounded wait-free.

从这个定义来看, wait-free方法永远不会阻塞, 因此也不会发生死锁. 此外, 每个参与者都可以在有限步骤后继续运行, wait-free方法也不存在接现象.

###### Lock-freedom
Lock-freedom的语义比wait-freedom较弱. 在lock-free场景下, 总有一些方法可以在有限步骤内结束. 这个定义意味着lock-free调用不会有死锁出现. 另一方面, ***有一些方法***可以在有限步骤内结束不足以保证所有的调用最终都能结束. 换言之, lock-freedom不足以避免饥饿现象.

###### Obstruction-freedom
TODO.

#### 2.2 Actor系统
Actors是封装了状态和行为的对象, 他们通过交换信息(存储在接受者的mailbox中)来完成通信. 在某种意义上, actors是最严格的面向对象编程, 它们可以被视为一个一个人: 通过actors建立解决方案, 分配出子任务划分给一组人, 将函数分配成一个组织结构并且考虑容错. 最终Actor可以形成一个构建软件的脚手架.

##### 2.2.1 分级结构
TODO.

### Actors
#### 3.1 Actors
Actors模型提供了写并发和分布式系统的高层抽象. 它使得开发者不用处理显示的锁和线程管理, 可以更容易的写出正确的并发和并行系统. Actors在1973年Carl Hewitt的论文中首次提出并在Erlang语言中流行起来, 被爱立信成功的用于构建高并发和高可用的电信系统.

Akka Actor的API和Scala的Actor很类似, 都从Erlang中借鉴了一些语法.

##### 3.1.1 创建Actors
***注意: 因为Akka强制父监控, 每个actor都被监控并且要监控它的孩子, 建议你熟悉下Actor Systems, Supervision和Monitoring, 最好也阅读下Actor References, Paths和Addresses.***

###### 定义Actor class
Actor类通过扩展Actor基类并且实现receive方法来实现. receive方法需要定义一系列case状态机来定义该Actor可以处理的消息(使用Scala的模式识别)以及如何处理该消息.

这里有一个例子:

	import akka.actor.Actor
	import akka.actor.Props
	import akka.event.Logging

	class MyActor extends Actor {
		val log = Logging(context.system, this)
		def receive = {
			case "test" => log.info("receive test")
			cast _      => log.info("received unkown message")
		}
	}

请注意Akka Actor的receive消息循环是完整的, 这和Erlang和Scala Actor不一样. 这意味着你需要提供一个所有消息的模式匹配, 如果你想处理未知的消息, 你需要提供一个default case. 否则会有一个akka.actor.UnhandleMessage(message, sender, recipient)会发布到ActorSystem的EventStream.

注意receive的返回类型是Unit;如果actor需要对收到的消息进行回复那么必须如下文显示操作.

receive方法的结果是一个partial function对象, 该对象会被actor保存作为"初始行为", 如果在actor创建后向修改该行为可以参见Become/Unbecome.

###### Props
Props是一个在创建actor时指定选项的配置类, 你可以认为它是一个不可变量, 因此创建带有部署信息的actor时可以自由共享. 这里有几个创建Props实例的例子:

	import akka.actor.Props

	val props1 = Props[MyActor]
	val props2 = Props[new ActorWithArgs("arg")) // careful, see below
	val props3 = Props(calssOf[ActorWithArgs], "arg")

第二个变量声明展示了如果在创建Actor的时候传递构造参数, 这种方法只能用在actor外部.

最后一行TODO

###### 危险的声明
	// NOT RECOMMENDED with another actor:
	// encourages to close over enclosing class
	val props7 = Props(new MyActor)

该方法在另一个actor内部不建议使用, 因为它鼓励close over the enclosing scope, 导致Props不可序列化并有可能导致竞争条件(打破了actor封装). 我们会在未来的版本中提供一个宏来支持相似的语法, 在目前该声明会被丢弃. 另外也可以在actor的伴随对象的Props工场中做这种声明.

这里有这些方法的两个use-case: 为actor传递参数-可以通过新引入的Props.apply(clazz, args)方法来解决, 或者在本地匿名类中创建actor. 后一种方法可以用actor来命名类(如果在最上层object中没有声明, 那么需要把该instance的this引用作为第一个参数传递进去)

***Warning: 在一个actor中声明另一个actor是十分危险的, 会破坏actor的封装. 永远不要把actor的this引用传递给Props!***

###### 建议实践
在每个Actor的伴随对象中提供一个工厂方法可以保证Props的创建和actor的定义尽可能的接近.这可以避免使用Props.apply(...)方法使用传名引用的陷阱, 因为伴随对象的代码段在作用域范围内不会维持引用.

	object DemoActor {
		/**
		 * Create Props for an actor of this type.
		 * @param magicNumber The magic number to be passed to this actor's constructor.
		 * @return a Props for creating this actor, which can then be further configured
	 	 * (e.g. calling `.withDispathcer()` on it)
	 	 * /
	 	def props(magicNumber: Int): Props = Props(new DemoActor(magicNumber))
	}

	class DemoActor(magicNumber: Int) extends Actor {
		def receive = {
			case x: Int => sender() ! (x + magicNumber)
		}
	}

	class SomeOtherActor extends Actor {
		// Props(new DemoActor(42)) would not be safe
		context.actorOf(DempActor.props(42), "demo")
		// ...
	}

###### 通过Props创建Actor
Actors可以通过向Props市里传递给actorOf工厂方法来创建, actorOf方法是ActorSystem和ActorContext提供的.

	import akka.actor.ActorSystem
	
	// ActorSystem is a heavy object: create only one per application
	val system = ActorSystem("mySystem")
	val myActor = system.actorOf(Props[MyActor], "myactor2")

使用ActorSystem可以创建顶层actor, 该actor由actor系统提供的监控actor监管, 使用actor的context可以创建一个子actor.

	class FirstActor extends Actor {
		val child = context.actorOf(Props[MyActor], name = "myChild")
		// plus some behavior ...
	}

强烈建议创建子, 孙子的层次结构, 这样和应用的逻辑容错处理结构吻合, 参见ActorSystems.

调用actorOf会返回一个ActorRef实例. 这是一个actor实例的handler并且是唯一可以与它交互的方式. ActorRef是不可变的, 并且和Actor之间有一对一的关系. ActorRef是可序列化并且可以网络感知的. 这意味着你可以序列化, 发送到网络上并且在一个远程机器上使用并且它仍然代表着同一个原始节点的actor.

其中name参数是可选的, 但是你必须为你的actor的命名, 因为它要被用来记录消息区分. 命名不能为空或者以$开头, 但是可以包含URL加密的字符(eg. %20代表空格). 如果给定的名字已经被另一个子actor使用会抛出InvalidAQctorNameException异常.

Actor在创建后会自动异步启动.

###### 依赖侵入
如上文所述, 如果你的Actor有一个带参构造函数那么它必须称为Props的一部分. 但有时候当必须使用工场方法, 例如当实际构造参数是由依赖侵入框架锁决定的.

	import akka.actor.IndirectActorProducer

	class DependencyInjector(applicationContext: AnyRef, beanName: String)
		extends IndirectActorProducer {
		override def actorClass = classOf[Actor]
		override def produce = 
			// obtain fresh Actor instance from DI framwork ...
	}

	val actorRef = system.actorOf(
		Props(classOf[DependencyInjector], applicationContext, "hello"),
		"helloBean")

***Warning: 有时候你可能被诱导提供一个IndirectActorProducer, 让它总是返回一个相同的instance, e.g. 使用一个lazy val. 这是不支持的, 因为它和一个actor充气的意义不符, 在这里有讨论: 重启意味着什么. 当使用依赖侵入框架时, actor必须有单例作用域***

###### 收件箱
当在actor外部写需要和actor交互的代码时, ask模式是一个解决方案, 但是你不能做以下两件事: 接受多个回复(e.g. 订阅一个ActorRef到一个通知服务), 观察其他actor的生命周期. 为了实现这些功能诞生了Inbox类:

	implicit val i = inbox()
	echo ! "hello"
	i.receive() should be("hello")

这里有一个从inbox到actor引用的隐式转换, 这意味着在这个例子里sender引用会被隐藏. 因为允许在最后响应接收信息. Watch一个actor也很简答:
	
	val target = // some actor
	val i = inbox()
	i watch target