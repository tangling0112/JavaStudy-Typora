# `Redis`哨兵的诞生背景

> ​		前面我们的`Redis`主从复制章节中提到,主从复制有着当`Redis Master`主节点宕机后,由于`Redis`并不会在`Redis Slave`从节点中暂时自动选取出一个临时`Redis Master`节点,导致整个主从复制架构在`Redis Master`宕机后只能提供数据读取服务无法提供数据写入服务的问题.
>
> **为了解决`Redis Master`宕机造成写入服务下线的问题我们就引入了`Redis`哨兵机制**

# `Redis`哨兵的解决方案

# `Redis`哨兵基础架构

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303092051147.png" alt="image-20230309205106077"  />

# `Redis`哨兵基本配置流程

- `cp /opt/redis-7.0.9/sentinel.conf /opt/MyRedis7/`
- 修改`sentinel.conf`配置文件
- `mkdir /opt/MyRedis7/sentinel`
- `vim redis-sentinel26379.log`
- `:wq!`
- `redis-sentinel /opt/MyRedis7/sentinel.conf --sentinel`启动哨兵

# `Redis Master`的主观下线与客观下线

> 我们知道,我们的`Redis`哨兵会按照一定的规则向我们的`Redis Master`发送数据包并等待其回复,以确认`Redis Master`的存活

## 主观下线

- 若某一次我们的`Redis`哨兵发送数据包给`Redis Master`后`Redis Master`**超过了指定的时间都没有给我们的`Redis`哨兵回复**,那么我们的`Redis`哨兵**就认为这台`Redis Master`主观下线**了

## 客观下线

- 如果认为我们的`Redis Master`**主观下线**的哨兵数量**大于等于**我们在哨兵的配置文件中**设置的数量**,那么我们的哨兵集群就会**达成一致认为我们的`Redis Master`客观下线**了.

# `Sentinel`配置文件

```shell
#是否开启保护模式
protected-mode no
#哨兵进程监听的端口号
port 26379

#是否让哨兵进程以守护进程的方式工作
	#yes:我们的哨兵进程开启后会在系统后台运行
	#no:我们的哨兵进程开启后会阻塞我们当前Linux终端
daemonize yes

#设置哨兵进程的进程文件的保存位置
pidfile /var/run/redis-sentinel26379.pid

#设置哨兵进程的日志文件的存储位置
logfile "/opt/MyRedis7/sentinel/redis-sentinel26379.log"

#设置哨兵进程的工作目录
dir /opt/MyRedis7/sentinel

#设置哨兵进程监听的Redis Master主机
	#mymaster:我们给这个Master主机设置的名字
	#127.0.0.1:Master主机的Host
	#6379:Master进程所监听的端口号
	#2:指定至少有两个哨兵认为该Master主机客观下线时,进行投票从Redis Slave选取新的Redis Master
sentinel monitor mymaster 192.168.184.160 6379 2

#设置哨兵进程访问指定Redis Master主机所用的密码
sentinel auth-pass mymaster 58828738

#指定一个毫秒级时间,当超过这个时间指定的Master主机没有发送消息告知当前哨兵存活的信息的话,当前哨兵认为该Master主观下线
sentinel down-after-milliseconds mymaster 30000

acllog-max-len 128

#
sentinel parallel-syncs mymaster 1

#设置一个毫秒级时间,当超过这个时间我们的故障转移(也就是新的Master节点没有发消息来告诉该哨兵它已经开始工作了)就被当前的哨兵认为是失败了
sentinel failover-timeout mymaster 180000

#
#sentinel notification-script mymaster /var/redis/notify.sh

#
#sentinel client-reconfig-script mymaster /var/redis/reconfig.sh

#
sentinel deny-scripts-reconfig yes

#
#SENTINEL rename-command mymaster CONFIG GUESSME

SENTINEL resolve-hostnames no
SENTINEL announce-hostnames no
SENTINEL master-reboot-down-after-period mymaster 0
```

# `Redis`哨兵投票选举产生新`Master`的实现原理

## 初始主从结构启动阶段

### `Redis`哨兵通过`Redis Master`主机发现其所有的`Redis Slave`从节点

> ​		我们知道,我们在`sentinel.conf`配置文件中显式地给出了要监控的`Redis Master`主节点的`Host`,监控端口`Port`以及`Redis Master`主节点的访问密码`auth-pass`,也就是说我们的`Redis`哨兵是能够访问我们的`Redis Master`主节点的`Redis-server`的
>
> ​		但是很显然的是,我们的哨兵在不借助`Redis Master`的情况下是无法获取到任何`Redis Slave`的信息的,无论是`Host`还是`Port`
>
> ​		因此`Redis`哨兵的借助于`Redis Master`的`Redis Slave`发现机制是十分重要的

> **前提**:我们在`Redis`主从复制章节提到过,当`Redis Slave`向我们的`Redis Master`发起第一次数据同步请求时,我们的`Redis Master`就会将`Redis Slave`的`Host`,`Port`等信息记录下来

- :one:我们的哨兵节点会向我们的`Redis Master`发送消息告知其哨兵身份,当`Redis Master`主节点接收到这个消息后就会在合适的时候将当前主从复制架构的一些信息打包起来,发送给该`Redis`哨兵,并且也会记录下这个哨兵的`Host`,`Port`等信息
- :two:我们的哨兵接收到`Redis Master`发送过来的数据之后就会进行解析,并且向数据包中指明的`Redis Slave`发送消息,以告知它们正在有`Redis`哨兵对它们实施监控
- :three:我们的哨兵进程会按照一定的间隔间断地向我们的`Redis Master`主节点发送消息并等待回应(心跳机制),若`Redis Master`的行为让我们的哨兵们判断为应该下线并重新选举`Master`,那么我们的哨兵们就会开启选举过程

### `Redis`哨兵根据获取到的主从复制架构信息使用心跳机制确定`Redis Master`与`Redis Slave`的存活

- 我们的`Redis`哨兵会每隔指定的时间向`Redis Master`以及`Redis Slave`发送数据包,并等待它们的回应,从而确定它们的存活情况
- **注意**:**即便对应的`Redis`节点宕机**了,这个**数据包发送机制也不会停止**.这样才能确保我们的哨兵尽可能准确地**获知**我们的主从复制架构中**哪些节点下线**了,**哪些节点上线**了,**哪些节点下线之后重启上线**了.

## `Redis Master`宕机

### `Redis`哨兵通过投票制判定`Redis Master`客观下线开启新`Redis Master`选举流程**??????**

> 我们前面知道我们的`Redis`哨兵会不间断地发送数据包以确认`Redis Master`主节点的存活

- :one:以下几步依次发生
    - 哨兵`A`发送了数据包给`Redis Master`之后迟迟没有等到恢复,此时哨兵`A`判断`Redis Master`主观下线了,因此哨兵`A`通知其他的各个哨兵说我觉得`Redis Master`主观下线了
    - 哨兵`B`发送了数据包给`Redis Master`之后迟迟没有等到恢复,此时哨兵`B`判断`Redis Master`主观下线了,因此哨兵`B`通知其他的各个哨兵说我觉得`Redis Master`主观下线了
    - 哨兵`C`发送了数据包给`Redis Master`之后迟迟没有等到恢复,此时哨兵`C`判断`Redis Master`主观下线了,因此哨兵`C`通知其他的各个哨兵说我觉得`Redis Master`主观下线了
- :two:由于哨兵`A`发送消息时,`B,C`还没有发现`Redis Master`主观下线了,所以它们两个哨兵都会回复哨兵`A`没有下线
- :three:然后哨兵`B`发送的主观下线通知送达了,此刻可能哨兵`C`还是没有检测到`Redis Maser`主观下线了,但是哨兵`A`是肯定知道`Redis Master`主观下线了,此时`A,B`就有两票认为`Redis Master`主观下线了,若设置的阈值就是`2`,那么哨兵`B`就会触发重新选举机制,通知`A,C`开启重新选举
- :four:若我们设置的阈值为`3`,那么就轮到哨兵`C`收取`A,B`的回复,此时的回复肯定都是主观下线,就有了三票,因此同样会触发重新选举机制,通知`A,B`开启重新选举

### `Redis`哨兵通过`Raft`算法从`Redis`哨兵中选举出领导者`Redis`哨兵用于主导主从架构的变更

### `Redis`哨兵在上面选出的主导者的主导下通过选举从`Redis Slave`中选举出新的`Redis Master`

- :one:

![image-20230310165741582](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303101657654.png)

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303101644318.png" alt="image-20230310164450270" style="zoom:67%;" />

### `Redis`哨兵主导者操控进行主从复制架构的变更

- 我们的`Redis`哨兵主导者会**依次向新的`Redis Master`,`Redis Slave`发送消息**
    - **发送的消息分为两部分**
        - **调用`SlaveOf`命令**改变主从复制架构
            - `Redis`哨兵主导者会调用新的`Redis Master`主节点的`SlaveOf No One`命令
            - `Redis`哨兵主导者会调用新的`Redis Slave`从节点的`SlaveOf 新Master的Host 新Master的Port`
        - **发送配置文件修改指令**要求`Redis`节点**修改配置文件**
            - `Redis`哨兵主导者会要求**新的`Redis Master`修改配置文件**,如将原本的`replicaof 老Master的Host 老Master的Port`属性配置删除等等
            - `Redis`哨兵主导者会要求**新的`Redis Slave`修改配置文件**,如将原本的`replicaof 老Master的Host 老Master的Port`修改为`replicaof 新Master的Host 新Master的Port`

> **注意**:虽然老`Redis Master`宕机了,但是我们的各个哨兵节点还是会保留下让其重新上线后变为`Redis Slave`的能力

## 老`Redis Master`重新上线

> ​		我们前面提到即便我们的`Redis`节点宕机,我们的**`Redis`哨兵也依旧会按照一定间隔向它们持续发送数据包**
>
> ​		这里我们的**老`Redis Master`重新上线**了,它**收到来自`Redis`哨兵的询问**就会尽快地**回答**,我们的`Redis`哨兵**收到回复后**就会**了解到我们的老`Redis Master`重新上线**了

- 具体哨兵集群怎么协商去告诉老`Redis Master`修改配置的流程我并不清楚,只不过大致会出现下面的情况
- 我们的老`Redis Master`会**收到以下两类请求**
    - :one:**调用老`Redis Master`的`SlaveOf 新Master的Host 新Master的Port`命令**,让我们的老`Redis Master`节点变为我们的新`Redis Master`的`Redis Slave`从节点
    - :two:要求老`Redis Master`**修改配置文件**,如为配置文件中**添加`replicaof 新Master的Host 新Master的Port`属性配置项**.

# `Redis`哨兵相关问题汇总

## :one:若选取新的`Master`完成后一段时间,原本宕机的`Redis Master`重新上线,那么此时`Redis`会呈现出怎样的行为呢?

> **老`Redis Master`会转变为新`Redis Master`的`Redis Slave`**

- :one:当我们的老`Redis Master`宕机期间,我们的`Redis`哨兵会间断地发送消息确认老`Redis Master`是否重新上线,如果它重新上线了
- :two:若老`Redis Master`重新上线了,那么`Redis`哨兵就会根据当前的主从复制架构发送一些信息给我们的老`Redis Master`
- :three:老`Redis Master`会根据收到的从`Redis`哨兵发送过来的信息,对其配置文件进行修改(这个修改是持久化的),以便使我们的老`Redis Master`重新融入我们的主从复制架构
    - 具体的修改就是会自动地给配置文件中添加或删除一些数据,如添加上`replicaof 新Master的Host Port`这样的属性
- :four:老`Redis Master`的配置文件修改完成后就会进行重新加载,从而我们的老`Redis Master`就变成了我们的新`Redis Master`的`Redis Slave`从节点
- :five:根据主从复制架构,我们的`Redis`哨兵也会将主从复制架构目前的状态存储起来.
- :six:至此老`Redis Master`重启后,我们的新主从复制架构就建立并稳定运行了起来.

## :two:`Broken Pipe`报错

## :three:当我们的哨兵管控的主从复制架构中的所有的主机从正常状态突然**全部宕机然后又依次重启**，那么此时我们的`Redis`主从复制架构会呈现出怎样的行为呢？

> ​		我们这里着重提到是**整个主从复制架构宕机重启**,并且**值得注意**的是,**各个`Redis`节点的启动速度肯定不一致**,因此我们的问题主要在于`Redis`哨兵**是否会指定第一个重启成功的`Redis`节点成为新的`Redis Master`**.

## :four:当我们的主从复制架构的`Master`主机宕机,然后`Redis`哨兵为我们选取出了一个新的`Master`,然后我们的新的主从复制架构中的所有主机全部宕机.在某一时刻我们的`Redis`主从复制架构上的全部主机又重新上线(包括老的`Redis Master`在内),那么此时我们的`Redis`主从复制架构会呈现出怎样的行为呢?

> **这里的主要问题有两个**
>
> - :one:`Redis`哨兵**是否**会指定**第一个重启成功**的`Redis`节点成为**新的`Redis Master`**
> - :two:由于我们的老`Redis Master`**整个过程都是宕机**的,因此**其`.conf`配置文件是没有被修改**的,那么当**老`Redis Master`作为第一个重启成功的节点**以及作为**非第一个重启成功的节点**这两种情况下**是否会有区别**,**如果有**这些**区别又是什么**

## :five:`Redis`是如何从借助`Raft`算法从哨兵集群中选取出一个哨兵代表来协调主从复制架构的调整的呢?

![image-20230310155830274](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303101558395.png)

# `Redis`哨兵的使用建议

![image-20230310170934022](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303101709071.png)
