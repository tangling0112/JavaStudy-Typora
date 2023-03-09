# `Redis`主从复制的诞生背景

> ​		我们前面的叙述中,我们的数据库**完全由我们的一台主机上的`Redis-server`提供服务**,无论是数据的读取还是数据的写入都由其完成.显然这样的结构当我们**对于数据库的操作激增时会导致`Redis-server`无法及时地为我们提供服务**.
>
> ​		我们知道,通常情况下对于我们的数据库而言,**一般写入操作是远少于读取操作**的,因此我们参考`MySQL`的主从复制架构,将其运用到我们的`Redis`上来,构建起一个**`Redis Master`主节点来负责支持写入操作**,构建起**大量的`Redis Slave`从节点来支持读取操作**.这也就是我们的`Redis`主从复制的诞生背景

# `Redis`主从复制特性

## 基于`replicaof`属性的主从复制与基于`slaveof`命令的主从复制的共同特性

- :one:只有`Redis Master`**主节点支持读写操作**,我们的`Redis Slave`从节点是**只允许进行读操作**的,**写操作是非法的**
    - ![image-20230309135431759](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303091354356.png)
- :two:**无论**我们的`Redis Slave`**从节点何时启动**,**只要其正确连接**到我们的`Redis Master`主节点上,就会**在合理的延迟内保证**与`Redis Master`中`Redis Database`数据库的**一致性**
- :three:如果**主从关系维持**的过程中,我们的`Redis Master`**主机宕机**了,那么我们的`Redis Slave`**从节点会停止同步**,但是**不会跟着宕机**,此时我们**依然可以读取**`Redis Slave`在`Redis Master`主机**宕机之前同步到的数据**.当然同样还是**只可读不可写**,当`Redis Master`主机**重新上线后**,`Redis Slave`会**自动连接`Redis Master`主机并同步数据**
- :four:如果**主从关系维持**的过程中,我们的`Redis Slave`**从节点宕机**了,那么当我们的该`Redis Slave`**从节点重新上线**时会**自动连接**到我们的`Redis master`主机并**同步数据**

## 基于`SlaveOf`命令的主从复制的特有特性

### 情形一

> ​		我们考虑这样一种情况,我们有三台`Redis`服务器主机`A,B,C`,每一台都为不同的应用提供着服务,但是突然`B,C`主机对应的应用被砍掉了,并且此时我们的`A`主机服务的应用使用量激增,急需扩容.此时我们希望能够把`B,C`主机变为`A`主机的从机,但是我们不希望去重启我们的`B,C`主机,我们希望直接通过`SlaveOf`命令来实现.
>
> ​		这时我们遇到的问题就在于,我们的`B,C`主机中**尚且还存有其原本服务的应用的数据**,那么我们使用`SlaveOf`命令之后`Redis`是否会**自动为我们清空这些旧数据**,然后**同步`A`主机上的数据**呢?
>
> ​		**答案当然是:``是的``**

### 情形二

> ​		我们考虑这样一种情况,我们有三台主机`A,B,C`其中主机`A`为`Redis Master`节点,`B,C`为`Redis Slave`从节点,这三个节点共同组成一个主从结构在运行.突然`B,C`主机由于一些问题宕机了,那么由于我们一般用`Redis-Slave`从节点来进行数据读取,这种情况下显然我们的数据读取服务便无法提供,因此我们必须重启`B,C`主机以恢复数据读取服务.
>
> ​		这时我们遇到的问题在于,我们的`SlaveOf`命令仅仅只是临时生效的,因此当我们将`B,C`主机重启之后,`B,C`主机便不再是我们主机`A`的`Redis Slave`从节点,因此此时我们必须再次使用`SlaveOf`指定`B,C`以主机`A`为`Redis Master`主节点
>
> ​		也就是说我们的`SlaveOf`命令的生效期为对应`Redis-server`的一次生命周期,若`Redis-server`被`Kill`掉了那么下一次再启动这个`Redis-server`后,我们上一次的`SlaveOf`指令便不再生效.

# `Redis`主从配置步骤

## 基于`replicaof`与`masterauth`属性的主从关系配置

### 配置文件设置

- **对`Master`主机配置文件进行设置**

```shell
#**********************************主要修改的部分************************************#
#bind 127.0.0.1 -::1
loglevel notice
logfile "/opt/Myredis/redis_6379.log"
pidfile /var/run/redis_6379.pid
daemonize yes
protected-mode no
port 6379
dbfilename dump6379.rdb
appendonly yes
appendfilename "appendonly6379.aof"
appenddirname "appendonlydir"
appendfsync everysec
requirepass 58828738
#**********************************主要修改的部分************************************#
```

- **对`Slaver1`第一台从机的配置文件进行设置**

```shell
#**********************************主要修改的部分************************************#
#bind 127.0.0.1 -::1
replicaof 192.168.184.160 6379
masterauth 58828738
loglevel notice
logfile "/opt/MyRedis7/redis_6380.log"
pidfile /var/run/redis_6380.pid
daemonize yes
protected-mode no
port 6380
dbfilename dump6380.rdb
appendonly yes
appendfilename "appendonly6380.aof"
appenddirname "appendonlydir"
appendfsync everysec
requirepass 58828738
#**********************************主要修改的部分************************************#
```

- **对`Slaver2`第二台从机的配置文件进行设置**

```shell
#**********************************主要修改的部分************************************#
#bind 127.0.0.1 -::1
replicaof 192.168.184.160 6379
masterauth 58828738
loglevel notice
logfile "/opt/Myredis/redis_6381.log"
pidfile /var/run/redis_6381.pid
daemonize yes
protected-mode no
port 6381
dbfilename dump6381.rdb
appendonly yes
appendfilename "appendonly6381.aof"
appenddirname "appendonlydir"
appendfsync everysec
requirepass 58828738
#**********************************主要修改的部分************************************#
```

```shell
#**********************************遵循默认的部分************************************#
tcp-backlog 511
timeout 0
tcp-keepalive 300
databases 16
always-show-logo no
set-proc-title yes
proc-title-template "{title} {listen-addr} {server-mode}"
save 5 2
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
rdb-del-sync-files no
dir ./
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync yes
repl-diskless-sync-delay 5
repl-diskless-sync-max-replicas 0
repl-diskless-load disabled
repl-disable-tcp-nodelay no
replica-priority 100
acllog-max-len 128
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-lazy-flush no
lazyfree-lazy-user-del no
lazyfree-lazy-user-flush no
oom-score-adj no
oom-score-adj-values 0 200 800
disable-thp yes
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes
aof-timestamp-enabled no
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-listpack-entries 512
hash-max-listpack-value 64
list-max-listpack-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-listpack-entries 128
zset-max-listpack-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes
jemalloc-bg-thread yes
```

### `Redis`各服务器启动

- **启动`Master`主机**

```shell
#Master主机上
redis-server /opt/MyRedis/redis6379.conf
```

- **启动`Slaver1`从机**

```shell
#Slaver1从机上
redis-server /opt/MyRedis/redis6380.conf
```

- **启动`Slaver2`从机**

```shell
#Slaver2从机上
redis-server /opt/MyRedis/redis6381.conf
```

## 基于`SlaveOf`命令与`masterauth`属性的主从关系配置

### 配置文件设置

- **对`Master`主机配置文件进行设置**

```shell
#**********************************主要修改的部分************************************#
#bind 127.0.0.1 -::1
loglevel notice
logfile "/opt/Myredis/redis_6379.log"
pidfile /var/run/redis_6379.pid
daemonize yes
protected-mode no
port 6379
dbfilename dump6379.rdb
appendonly yes
appendfilename "appendonly6379.aof"
appenddirname "appendonlydir"
appendfsync everysec
requirepass 58828738
#**********************************主要修改的部分************************************#
```

- **对`Slaver1`第一台从机的配置文件进行设置**

```shell
#**********************************主要修改的部分************************************#
#bind 127.0.0.1 -::1
masterauth 58828738
loglevel notice
logfile "/opt/MyRedis7/redis_6380.log"
pidfile /var/run/redis_6380.pid
daemonize yes
protected-mode no
port 6380
dbfilename dump6380.rdb
appendonly yes
appendfilename "appendonly6380.aof"
appenddirname "appendonlydir"
appendfsync everysec
requirepass 58828738
#**********************************主要修改的部分************************************#
```

- **对`Slaver2`第二台从机的配置文件进行设置**

```shell
#**********************************主要修改的部分************************************#
#bind 127.0.0.1 -::1
masterauth 58828738
loglevel notice
logfile "/opt/Myredis/redis_6381.log"
pidfile /var/run/redis_6381.pid
daemonize yes
protected-mode no
port 6381
dbfilename dump6381.rdb
appendonly yes
appendfilename "appendonly6381.aof"
appenddirname "appendonlydir"
appendfsync everysec
requirepass 58828738
#**********************************主要修改的部分************************************#
```

```shell
#**********************************遵循默认的部分************************************#
tcp-backlog 511
timeout 0
tcp-keepalive 300
databases 16
always-show-logo no
set-proc-title yes
proc-title-template "{title} {listen-addr} {server-mode}"
save 5 2
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
rdb-del-sync-files no
dir ./
replica-serve-stale-data yes
replica-read-only yes
repl-diskless-sync yes
repl-diskless-sync-delay 5
repl-diskless-sync-max-replicas 0
repl-diskless-load disabled
repl-disable-tcp-nodelay no
replica-priority 100
acllog-max-len 128
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no
replica-lazy-flush no
lazyfree-lazy-user-del no
lazyfree-lazy-user-flush no
oom-score-adj no
oom-score-adj-values 0 200 800
disable-thp yes
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
aof-use-rdb-preamble yes
aof-timestamp-enabled no
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-listpack-entries 512
hash-max-listpack-value 64
list-max-listpack-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-listpack-entries 128
zset-max-listpack-value 64
hll-sparse-max-bytes 3000
stream-node-max-bytes 4096
stream-node-max-entries 100
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes
jemalloc-bg-thread yes
```

### `Redis`各服务器启动

- **启动`Master`主机**

```shell
#Master主机上
redis-server /opt/MyRedis/redis6379.conf
```

- **启动`Slaver1`从机**

```shell
#Slaver1从机上
redis-server /opt/MyRedis/redis6380.conf
```

- **启动`Slaver2`从机**

```shell
#Slaver2从机上
redis-server /opt/MyRedis/redis6381.conf
```

- **使用`SlaveOf`让`Slaver1`从机连接并从属到`Master`主机上**

```shell
SLAVEOF 192.168.184.160
```

- **使用`SlaveOf`让`Slaver2`从机连接并从属到`Master`主机上**

```shell
SLAVEOF 192.168.184.160
```

# `Redis`主从复制相关命令

## `Info`

> **作用**
>
> - 获取当前`Redis`节点与主从复制相关的一些信息

```shell
info replication
```

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303091357592.png" alt="image-20230309135749519" style="zoom: 80%;" />![image-20230309135814332](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303091358377.png)

## `SlaveOf`

> **作用**
>
> - **用于给当前`Redis`主机指定其`Master`节点**
>     - `Host Port`就是**指定**一个**特定的主机**上运行在**特定的端口上的`Redis-server`进程**为其`Master`
>     - `NO ONE`就是**取消当前`Redis`主机的所有主从关系**,执行完毕后,我们当前主机便**不再具有`Master`**,但是在执行**之前从`Master`同步到的数据**还是**能够照旧访问**,只**不过不会再被更新**了

```shell
SLAVEOF HOST PORT
SLAVEOF NO ONE
```

## `Role`

# `Redis`主从复制基本流程

## 基于`replicaof`属性的主从架构

> 假设我们的`Master`主机已经启动完毕

- :one:我们通过`redis-server /opt/Myredis7/redis6380.conf`来启动我们的`Redis Slave`从节点
- :two:我们的`Redis`在读取到`replicaof`属性时就会知道我们当前启动的这个`Redis-server`是要作为某个`Redis Master`节点的从节点的.因此会在该`Redis-server`启动时的某一时刻向`Redis Master`节点发送数据同步请求
- :three:当`Redis Master`主节点收到来自`Redis Slave`从节点的数据同步请求时,就会对`Redis Master`数据库的情况进行快照保存,生成`RDB`格式的持久化格式快照,并且快照生成期间`Redis Master`收到的所有的涉及到数据写入的命令都会被额外保存一份,并在`RDB`持久化格式快照生成完毕后随其一起发送给`Redis Slave`从节点
    - **当然`Redis Master`也会同时记录下这个`Redis Slave`的信息包括`Host`,`Port`等信息**
- :four:`Redis Slave`从节点收到`Redis Master`发送过来的同步数据时就会使用其来初始化其数据库,并根据`Redis Master`发送过来的写入命令进一步同步数据,当一切准备就绪后我们的`Redis Slave`节点的`Redis-server`就能够真正地启动起来.
- :five:这之后就进入了平稳通信阶段,`Redis Master`主节点会不断地在合适的时候把其接收并成功执行的写入命令发送给`Redis Slave`从节点,然后`Redis Slave`从节点就会根据这些命令进行数据库的同步.
    - **注意**:在这一平稳通信阶段我们的`Redis Master`主节点会根据配置以一定的间隔不断地向`Redis Slave`从节点发送数据包直至`Redis Slave`下线,以确定我们的`Redis Slave`的存活.
- :six:如果我们假设运行一段时间后我们的`Redis Slave`下线了,并且一段时间后`Redis Slave`又上线,那么会进行以下流程
- :seven:`Redis Master`与`Redis Slave`会进行通信从而确定如何同步数据
    - ![image-20230309193703906](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303091937971.png)

# `Redis`主从复制的两大经典架构

## 一主多仆架构

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303091901465.png" alt="image-20230309190114185" style="zoom:80%;" />

# 薪火相传架构

> **该架构能够实现的理论基础在于`Redis`的`Redis Slave`节点是能够被其他的`Redis`节点作为`Master`节点的.**

> **作用**
>
> - 我们前面提到了,我们的`Redis Master`会**间断地向其所有的`Redis Slave`从节点发送数据**写入相关的信息,当我们的`Redis Slave`过多时就**会导致我们的`Redis Master`对于主从复制的压力过大**.
> - 而薪火相传架构则**让`Redis Slave`子节点作为一个新的主节点来发展下线**,这样我们的`Redis Master`就可以通过借助这一机制**维持最少的直接`Redis Slave`**的方式**建立最多的`Redis Slave`节点**

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303091901602.png" alt="image-20230309190159545" style="zoom:80%;" />

# `Redis`主从复制的缺点

- :one:由于主从复制要通过**网络来传输同步数据**,因此会带来一定的`Redis Master`与`Redis Slave`之间数据的**短时间不一致性**.
- :two:前面我们也提到了,当我们的**`Redis Master`宕机**之后,我们的整个主从复制架构的**剩下的`Redis Slave`从节点就会停止数据的同步**只**提供数据读取服务**,此时我们的主从复制架构由于`Redis Master`的宕机**缺失了写入数据的能力**.`Redis`并**不会在剩下的`Redis Slave`中选取一台成为新的`Redis Master`**,要选出新的`Redis Master`必须**由我们人工使用配置文件或者`SlaveOf`重新构建主从复制架构**
