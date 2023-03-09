# `Redis`哨兵的诞生背景

> ​		前面我们的`Redis`主从复制章节中提到,主从复制有着当`Redis Master`主节点宕机后,由于`Redis`并不会在`Redis Slave`从节点中暂时自动选取出一个临时`Redis Master`节点,导致整个主从复制架构在`Redis Master`宕机后只能提供数据读取服务无法提供数据写入服务的问题.`Redis`哨兵机制正是为了解决这一问题而诞生的.

# `Redis`哨兵基础架构

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303092051147.png" alt="image-20230309205106077" style="zoom: 80%;" />

# `Redis`哨兵基本配置流程

- `cp /opt/redis-7.0.9/sentinel.conf /opt/MyRedis7/`
- 修改`sentinel.conf`配置文件
- `mkdir /opt/MyRedis7/sentinel`
- `vim redis-sentinel26379.log`
- `:wq!`
- `redis-sentinel /opt/MyRedis7/sentinel.conf --sentinel`启动哨兵

# `Redis Master`的主观下线与客观下线

- 主观下线
- 客观下线

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



# `Redis`哨兵的基本作用

### `Redis`哨兵实现主从监控

### `Redis`哨兵实现主从监消息通知

### `Redis`哨兵实现主从监故障转移

### `Redis`哨兵实现配置中心

# `Redis`哨兵投票选举产生新`Master`的实现原理

## `Redis`哨兵通过`Redis Master`主机发现其所有的`Redis Slave`从节点

> ​		我们知道,我们在`sentinel.conf`配置文件中显式地给出了要监控的`Redis Master`主节点的`Host`,监控端口`Port`以及`Redis Master`主节点的访问密码`auth-pass`,也就是说我们的`Redis`哨兵是能够访问我们的`Redis Master`主节点的`Redis-server`的
>
> ​		但是很显然的是,我们的哨兵在不借助`Redis Master`的情况下是无法获取到任何`Redis Slave`的信息的,无论是`Host`还是`Port`
>
> ​		因此`Redis`哨兵的借助于`Redis Master`的`Redis Slave`发现机制是十分重要的

> **前提**:我们在`Redis`主从复制章节提到过,当`Redis Slave`向我们的`Redis Master`发起第一次数据同步请求时,我们的`Redis Master`就会将`Redis Slave`的`Host`,`Port`等信息记录下来

- :one:我们的哨兵节点会向我们的`Redis Master`发送消息告知其哨兵身份,当`Redis Master`主节点接收到这个消息后就会在合适的时候将当前主从复制架构的一些信息打包起来,发送给该`Redis`哨兵,并且也会记录下这个哨兵的`Host`,`Port`等信息
- :two:我们的哨兵接收到`Redis Master`发送过来的数据之后就会进行解析,并且向数据包中指明的`Redis Slave`发送消息,以告知它们正在有`Redis`哨兵对它们实施监控
- :three:我们的哨兵进程会按照一定的间隔间断地向我们的`Redis Master`主节点发送消息并等待回应(心跳机制),若`Redis Master`的行为让我们的哨兵们判断为应该下线并重新选举`Master`,那么我们的哨兵们就会开启选举过程

## `Redis`哨兵通过选举机制从`Redis Slave`中选举出新的`Redis Master`节点

- :one:每一个`Redis`哨兵节点都会根据其从原本的`Redis Master`获取到的`Redis Slave`的`Host`,`Port`等信息向不同的`Redis Slave`发送信息以确定这些`Redis Slave`中在线的部分
- :two:每一个哨兵都会选出一个`Redis Slave`,这个`Redis Slave`一定是当前哨兵能够连接到的,并且符合一定优势的`Redis Slave`.
- :three:然后就会综合各个`Redis`哨兵的选举结果根据少数服从多数的原则,选取出一个充当新`Redis Master`的节点.
- :four:当选举出新的`Redis Master`节点后,我们的`Redis`哨兵节点就会派出代表来通知我们的各个`Redis`节点,让他们使用运行时方式改变配置,从而改变`Master`,`Slave`关系,形成新的主从结构.

![image-20230309215253857](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303092152999.png)

# `Redis`哨兵相关问题汇总

## :one:若选取新的`Master`完成后一段时间,原本宕机的`Redis Master`重新上线,那么此时`Redis`会呈现出怎样的行为呢?

## :two:`Broken Pipe`报错

