# 异步通讯框架xxl-mq
github地址：https://github.com/xuxueli/xxl-mq

git.osc地址：http://git.oschina.net/xuxueli0323/xxl-mq

博客地址(内附使用教程)：http://www.cnblogs.com/xuxueli/p/4918535.html

技术交流群(仅作技术交流)：367260654

## V1.1规划

##### 角色
- mq: 消息
    - destination:
        - TOPIC=广播, 消息推送给该主题下所有 consumer
        - QUEUE=串行队列, 消息队列方式执行, 支持Delay, 消息将会按照被生产的顺序被串行的消费, 即如存在多个 consumer 时, 只会有一个被激活进行消息消费;
        - CONCURRENT_QUEUE=并发队列, 消息队列方式执行, 支持Delay, 消息将会被多个broker并行消费, 不保证消费顺序, 消息会根据broker进行分组;
    - name: 消息主题
    - data: 消息数据, Map<String, String>对象系列化的JSON字符串
    - delay_time: 延迟执行的时间, new Date()立即执行, 否则在延迟时间点之后开始执行;
    - add_time: 创建时间
    - update_time: 更新时间
    - status: 消息状态, 可选范围: NEW=新消息、ING=消费中、SUCCESS=消费成功、FAIL=消费失败、TIMEOUT=超时
    - msg: 历史流转日志
- producer: 生产者
- consumer: 消费者

- broker: 代理, 负责: 1、接收 producer 生产的消息, 2、向 consumer 推送订阅的消息, 3、接受 consumer 对消息的消费结果回调;
    - message 需要登记: 1、主题下一旦有消息数据,不可删除; 只可修改备注 ,1、broker 只服务登记的消息;2、登记便于报表统计;3、便于邮件报警;4、
- client: 提供 producer 和 consumer 支持;

##### 实现

ZK节点:
```
-xxl-mq
    - broker()
        - address1(6080)
        - address2
    - mq
        - name1
            - address1(6070)
            - address2
        - name2
            - address1
            - address2
```

- client(producer): rpc客户端, 向远程rpc发送请求服务
- client(consumer) rpc服务, 端口 6070, 自动注册
- broker: rpc服务, 端口 6080, 自动注册
    
实现原理:
- TOPIC处理器线程池: 一个 mq 一个线程, 节点最小的 broker,加载全部 TOPIC 消息, 匹配 mq 找到 consumer 地址列表, 广播发送消息(rpc请求, client接收到立即响应,执行结果异步通知);
- QUEUE处理器线程池: 一个 mq 一个线程, 节点最小的 broker,加载全部 TOPIC 消息, 匹配 mq 找到 consumer 地址列表, 选中最小的节点发送消息(rpc请求, client接收到立即响应,执行结果异步通知);
- CONCURRENT_QUEUE处理器: 

- 超时处理器: 每间隔30MIN扫描一次全表, 把超过1H状态仍然为ING的消息状态改为TIMEOUT,并在消息msg字段记录日志
- 报警器: 每间隔30MIN扫描一次全表, 以消息主题为维度进行统计, 邮件通知消息执行失败情况; 

##### 消息状态

##### 项目划分
- xxl-mq-broker: 端口6080, zk + netty集群: 接受生产消息入库, 消费消息推送至client端口;
- xxl-mq-client: 向admin推送消息, 接受admin消费消息;
- xxl-mq-example: 

瓶颈在于消息持久化,即mysql, 适用于2W/天以及以下的队列场景, 需要定期做表数据清理;


## 简介：
	一款轻量级、设计极简的 “异步通讯框架” ；
	支持Topic和Queue两种异步通讯模型；
	去中心化，可插拔式，完美集成spring；
	消息mysql持久化，上手简单；
	参考JMS1.1规范，一定程度上借鉴activemq、diagping-swollow和diagping-tiger；


## 同类型产品 (排名不分先后)：
	activemq : 一个完全支持JMS1.1和J2EE 1.4规范的 JMS Provider实现，应用广泛，自带队列状况监控。然而，ha通过zk的master-slave实现的，并没有负载分流能力；
	memcacheq : 一款轻量级的分布式队列服务，基于memcache协议，消息数据持久化写入BerkeleyDB，只有get/set两个方法，支持ha，性能比通用的MQ高很多倍；
	kafka : 效率极高的mq服务，天生支持ha；但是有得必有失，kafka不保证消息事务，没有消息确认机制，适合于比较不严谨的数据；
	diagping-swollow : 队列服务；支持master-slave；cs结构，client端通过pegion发送消息；通过netty接收server端推送消息，队列逻辑在server端；使用mongo持久化消息，一个topic对应一个数据库实例；
	diagping-tiger : 队列服务；mysql持久化消息；producer和consumer均通过jdbc主动pull队列消息；通过zk协调各节点消息分配；自带执行结果监控；可以精确查看每条消息的详细信息和执行状况；支持delay；

## xxl-mq实现原理：
	Topic实现原理：【每条Topic消息，每个监听对应topicName的comsumer线程都会执行且只一次；】
		producer：通过jdbc向mysql中push消息；
		consumer：根据心跳时间，周期性通过jdbc从pull新消息，执行成功后记录执行日志；一条topic消息一个comsumer线程只会执行一次；超过topic生存周期(3*beat)的topic被抛弃；
	
	Queue实现原理：【每条Queue消息，一生只会被执行一次】
		
		1、串行queue：【同一时间只会有一个comsumer维持life状态，其他comsumer会被阻塞掉；该QueueName下的所有消息都会被分配给life状态的consumer；保证消息在单节点顺序执行】
			：producer：通过jdbc向mysql中push消息；
			：comsumer：
				通过竞争queue行锁，确保同一时刻只有一个comsumer是life状态；
				各comsumer通过心跳检测锁状态；竞争失败则阻塞；竞争成功则周期性保护queue锁；
				竞争成功的comsumer主动pull队列数据执行对应逻辑，确保所有消息被同一个comsumer串行消费；
				
		2、并发queue：【该QueueName下的每条Queue消息只会被分配给其中一个comsumer；各个consumer线程并行的pull分配给自己的消息并执行；每条Queue消息都会被执行且只执行一次】
		
			2.1 并发Queue实现-方式A：“取余算法”：
				：producer：通过jdbc向mysql中push消息，每条消息生成sequence id；
				：consumer：
					各consumer通过心跳检测life状态该queueName下所有consumer列表，得出长度count，对列表排序计算自己的排名rank；
					按照计算公式： sequence id % count = rank 查询出分配给自己的消息；保证每个消息只会分配给一个comsumer；
					各个consumer线程并行pull出分配给自己的消息并消费；
				
			2.2 并发Queue实现-方式B：“Hash一致性算法”：
				：producer：通过jdbc向mysql中push消息，通过“Hash一致性算法”分配consumer，将consumer uuid记录到消息中；
				：comsumer：通过心跳，周期性获取分配给自己的消息并消费；
				：boss线程：通过心跳，周期性检测消息的consumer是否有效，对于consumer已经非life状态的消息，通过 “Hash一致性算法” 重新分配；
				：“Hash一致性算法”为消息分配consumer的逻辑：
					为每个consumer uuid(NODE)创建VirtualNum个虚拟节点(VirtualNode)，将虚拟节点Hash(hash(queueName + consumer uuid + virtual index))后散列到一致性Hash环上；
					对消息的进行hash (queueName + sequence id)作为fromKey，匹配一致性Hash环上的虚拟节点，映射获取分配的comsumer uuid；
		
1、特点：
--------------------
	1、异步处理，避免客户机等待；
	2、针对 “耗时且不需即时响应” 的操作；
	3、解耦：提高扩展性；“事件驱动架构” 的核心，各组件以异步方式响应事件。例如，复杂订单支付、审核等各分支逻辑异步执行；
	4、消息持久化：提高系统可靠性；
	5、重发机制：提高消息到达的成功率；
	6、消息确认机制 (jms事务)：确保消息的发送与接收可靠；
	7、topic: 一条消息按照Pub/Sub的消息模型广播给所有相关的consumer；针对每个topic消息，每个consumer都会执行执行且仅执行一次；
	8、queue: 消息按照FIFO方式和PTP消息模型被相关的consumer消费；针对每个queue消息，只有一个consumer允许执行且仅执行一次；
	9、delay: topic和queue消息均支持delay；
	9、concurrent: queue默认为并发模式，此时不保证消费顺序， 但是消费迅速；queue允许开启非并发模式，通过数据库心跳行锁竞争实现，此时保证执行顺序；

2、概念：
--------------------
	消息(message)：通讯双方传递的信息主体；
	消息队列(message poll)：负责存储消息，支持FIFO，此文采用mysql存储；
	生产者(producer)：负责生产消息，并push到队列中；
	消费者(consumer)：负责从队列中pull出消息，并执行消息相关逻辑；

3、重要参数：
--------------------
	consumer_uuid : 每个consumer的唯一性标示；
	ConnectionFactory : 维护xxl-mq的boss线程和底层所有配置信息，单例；
	Destination : Topic和Queue消息的父类；
	MessageProducer : 生产者；
	MessageConsumer : 消费者；
	MessageListener : 消费者listener；
	
4、配置说明
--------------------
	topic_beat	: 
		作用一：Topic消息心跳检测；
		作用二：超过 “Topic声明周期(3 * topic_beat)” 未被消费的消息将会被抛弃；
	topic_pagesize	: 单次获取的topic消息数量；
	topic_cleandead : 超过“Topic声明周期(3 * topic_beat)”的消息数据，是否允许被删除，运行周期(3*topic_beat)，默认true；
	queue_beat	: 
		作用一：queue消息心跳检测；
		作用二：queue非concurrent方式运行时，超过 “消费者竞争锁生命周期(3*queue_beat)” 的竞争锁，且存在竞争者时，将会被抢夺；
		作用三：queue非concurrent方式运行时，超过 “消费者竞争锁生命周期(3*queue_beat)” 的竞争锁，且不存在竞争者时，将会被删除；
		作用四：queue以concurrent方式运行时，超过 “消费者声明周期(3*queue_beat)” 的消费者， 将会被删除；
	queue_pagesize	: 单次获取的queue消息数量；
	queue_cleansucess	: 已经执行成功的queue消息，是否允许被删除，运行周期(3*queue_beat)，默认false；

5、使用步骤：(去中心化，所有逻辑在producer和comsumer中，插拔式)
--------------------
	a. 引入依赖；
		<!-- xxl-mq-core -->
		<dependency>
			<groupId>com.xxl</groupId>
			<artifactId>xxl-mq-core</artifactId>
			<version>0.0.1-SNAPSHOT</version>
		</dependency>
		
	b. 执行建表sql (在xxl-mq/doc/db/db.sql文件中)，配置xxl-mq消息持久化数据库连接参数(配置文件：jdbc-xxl-mq.properties)
			c3p0.driverClass=com.mysql.jdbc.Driver
			c3p0.url=jdbc:mysql://ip:3306/db?Unicode=true&amp;characterEncoding=UTF-8
			c3p0.user=root
			c3p0.password=root_pwd
		
	c. 配置xxl-mq的ConnectionFactory；
		<!-- xxl-mq : base init -->
		<import resource="classpath*:applicationcontext-xxl-mq-database.xml"/>
		<import resource="classpath*:applicationcontext-xxl-mq-tx.xml"/>
		<!-- xxl-mq : mq init -->
		<bean id="xxlMqConnectionFactory" class="com.xxl.mq.factory.ConnectionFactory" init-method="init">
			<property name="messageService" ref="messageService" />
		</bean>
		
	d. 配置消息producer，并使用：
		
		1、配置producer (根据destination支持Topic和Queue)
		<!-- topic producer -->
		<bean id="topic01Producer" class="com.xxl.mq.client.MessageProducer">
			<property name="connectionFactory" ref="xxlMqConnectionFactory" />
			<property name="destination">
				<bean class="com.xxl.mq.destination.impl.Topic">
					<constructor-arg value="topic_01" />
				</bean>
			</property>
		</bean>
		
		2、注入action/controller或者service中
		@Autowired
		private MessageProducer topic01Producer;
		
		3、发送消息	 (producer的destination必须和message类型匹配)
		TopicMessage message = new TopicMessage();
		message.setInvokeRequest(JacksonUtil.writeValueAsString(Topic消息));
		topic01Producer.send(message);
		
	e. 配置消息consumer，并使用：
	
		1、开发MessageListener：
		@Component("topic01MessageListener")
		public class Topic01MessageListener implements MessageListener {
			private static Logger logger = LoggerFactory.getLogger(Topic01MessageListener.class);
			
			@Override
			public StatusEnum onMessage(Serializable message) {
				logger.info("######### onMessage :{}", JacksonUtil.writeValueAsString(message));
				return StatusEnum.SUCCESS;
			}
		
		}
		
		2、扫描MessageListener，交由spring统一管理：
		<context:component-scan base-package="com.xxl.service.mq" />
		
		3、配置consumer，指定MessageListener (queue类型支持concurrent和非concurrent，concurrent下允许多线程，各线程身份对等)
		<bean id="queue01Consumer" class="com.xxl.mq.client.MessageConsumer" init-method="init">
			<property name="connectionFactory" ref="xxlMqConnectionFactory" />
			<property name="destination">
				<bean class="com.xxl.mq.destination.impl.Queue">
					<constructor-arg value="queue_01" />
				</bean>
			</property>
			<property name="messageListener" ref="topic01MessageListener" />
			<property name="consumer_concurrent_switch" value="true" />
			<property name="consumer_concurrent_num" value="3" />
		</bean>
	
	
	
		