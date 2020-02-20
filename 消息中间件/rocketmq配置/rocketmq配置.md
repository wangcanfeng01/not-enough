borker配置说明文档

#broker所属的集群名字

brokerClusterName=rocketmq-cluster

#broker名字，同个集群中的每个broker应当具有它自己独有的名字

brokerName=broker-a

#设置主broker和从broker  其中0 表示 主机，>0 表示 从机

brokerId=0

#nameServer地址（地址为ip：端口），多个地址之间用分号分割

namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876

#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数

defaultTopicQueueNums=4

#是否允许 Broker 自动创建Topic，测试时可以开启，实用时关闭

autoCreateTopicEnable=true

#是否允许 Broker 自动创建订阅组，测试时可以开启，实用时关闭

#在pull形式消费时若设置了falsename会报subscription group not exist，且收不到消息，在push形式消费时没有影响

autoCreateSubscriptionGroup=true

#Broker 对外服务的监听端口

listenPort=10911

#haService中使用

haListenPort=10912

#主要用于slave同步master

fastListenPort=10909

#定时删除文件时间点，默认凌晨 4点

deleteWhen=04

#文件保留最长时间，默认 48 小时

fileReservedTime=120

#commitLog每个文件的大小默认1G

mapedFileSizeCommitLog=1073741824

#ConsumeQueue每个文件默认存30W条，根据业务情况调整

mapedFileSizeConsumeQueue=300000

#强制删除文件时间间隔（单位毫秒）

#destroyMapedFileIntervalForcibly=120000

#定期检查Hanged文件间隔时间（单位毫秒）

#redeleteHangedFileInterval=120000

#检测物理文件磁盘空间,磁盘空间使用率不能超过88%

diskMaxUsedSpaceRatio=88

#存储总路径

storePathRootDir=/usr/local/rocketmq/store

#commitLog 存储路径

storePathCommitLog=/usr/local/rocketmq/store/commitlog

#消费队列存储路径存储路径

storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue

#消息索引存储路径

storePathIndex=/usr/local/rocketmq/store/index

#异常退出产生的文件存储路径

storeCheckpoint=/usr/local/rocketmq/store/checkpoint

#abort 文件存储路径

abortFile=/usr/local/rocketmq/store/abort

#限制的消息大小

maxMessageSize=65536

#Commitlog每次刷盘最少页数，每页4kb

flushCommitLogLeastPages=4

#ConsumeQueue每次刷盘最页数，每页4kb

#flushConsumeQueueLeastPages=2

#刷盘时间间隔（单位毫秒），此间隔时间优先级高于上面两个参数，即当时间间隔超过之后直接进行刷盘，不考虑页数问题

#flushCommitLogThoroughInterval=10000

#flushConsumeQueueThoroughInterval=60000

#Broker 的角色 （1） ASYNC_MASTER 异步复制Master （2） SYNC_MASTER 同步双写Master （3） SLAVE

brokerRole=ASYNC_MASTER

#刷盘方式 （1） ASYNC_FLUSH 异步刷盘  （2）SYNC_FLUSH 同步刷盘

flushDiskType=ASYNC_FLUSH

#是否开启事务check过程，消息体量大的时候可以不开启，默认为关闭状态

checkTransactionMessageEnable=false

#发消息线程池数量（如果不做配置，个数为16+（核*线程）*4）

#sendMessageThreadPoolNums=128

#拉消息线程池数量（如果不做配置，个数为16+（核*线程）*4）

#pullMessageThreadPoolNums=12



参考资源链接

http://code.taobao.org/p/astrotrain/diff/13/trunk/RocketMQ-3.1.0/rocketmq-store/src/main/java/com/alibaba/rocketmq/store/config