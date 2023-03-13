# `Redis`集群的诞生背景

> ​		`Redis`哨兵虽然为我们的主从复制结构解决了极为重要的`Redis Master`节点宕机重选问题,使得我们的主从复制架构较为稳定可靠.但是即便是这样我们**仍然面临一个极大的问题**,那么就是无论如何,我们的**主从复制架构中,数据写入的节点总是`Redis Master`节点**,**所有的`Redis Slave`节点**总是**只提供数据读取**服务.这**对于一个轻写入,高读取的应用而言可能能够负担**,但是假若我们的**应用对于写入的要求极高**,那么这样的单主机写入操作支持的主从复制架构对于`Redis Master`主机的**要求就非常高**.
>
> ​		**为了解决`Redis Master`主机负载超额问题,我们就得引入我们的`Redis`集群**

# `Redis`集群的给我们提供的解决方案

- **变单`Redis Master`为多`Redis Master`**
- **变单`Master`数据存储为多`Master`分区存储**

> ​		要解决单`Redis Master`的写入负载超额问题，最佳的办法就是**使得我们的`Redis`架构从单`Master`主机变为多`Master`主机**.**`Redis`集群正是这样一种设计架构**
>
> ​		`Redis`集群在扩展出多个`Redis Master`的基础上**又引入了数据分区**,使得我们`Redis`集群中的**每一个`Redis Master`中存储的数据不相交**(也就是说对于`100`份数据,每一个`Redis Master`存储其中一部分,并且任意两台`Redis Master`存储的数据都是不同的).
>
> ​		这也就使得我们的`Redis`集群**即便某一台`Redis Master`宕机**了,那么**即便在不考虑哨兵机制选举新`Master`的情况下**,我们的`Redis`集群也**依然能够提供除挂掉的`Redis Master`中存储的数据外的其他数据的写入服务**.
>
> ![image-20230310232821197](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303102328253.png)

> **借助于多`Redis Master`与数据分区设计,我们的`Redis`集群做到了较好的高可用性**

# `Redis`集群基础架构

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303102137838.png" alt="image-20230310213743723" style="zoom: 50%;" />

# `CRC16`算法介绍

```c
unsigned int keyHashSlot(char *key, int keylen) {
    int s, e; /* start-end indexes of { and } */

    for (s = 0; s < keylen; s++)
        if (key[s] == '{') break;

    /* No '{' ? Hash the whole key. This is the base case. */
    if (s == keylen) return crc16(key,keylen) & 0x3FFF;

    /* '{' found? Check if we have the corresponding '}'. */
    for (e = s+1; e < keylen; e++)
        if (key[e] == '}') break;

    /* No '}' or nothing between {} ? Hash the whole key. */
    if (e == keylen || e == s+1) return crc16(key,keylen) & 0x3FFF;

    /* If we are here there is both a { and a } on its right. Hash
     * what is in the middle between { and }. */
    return crc16(key+s+1,e-s-1) & 0x3FFF;
}
```

```c
uint16_t crc16(const char *buf, int len) {
    int counter;
    uint16_t crc = 0;
    for (counter = 0; counter < len; counter++)
            crc = (crc<<8) ^ crc16tab[((crc>>8) ^ *buf++)&0x00FF];
    return crc;
}
```

# 三大数据分区算法

> **数据分区就是通过算法,将大量的苹果,根据它们各自的某些特点进行分类,并放置到不同的篮子里面**

> **目前分布式领域常用的数据分区算法有以下三种**
>
> - :one:**哈希取余分区算法**
> - **:two:一致性哈希环分区算法**
> - :three:**哈希槽分区算法**
>     - `Redis`集群使用的数据分区算法

## 哈希取余分区算法

### 算法原理

- :one:哈希取余算法在最开始集群搭建时,就需要确定好我们的集群节点的数量`N`
- :two:然后我们设置一个`Hash`函数,每当我们需要读写我们的`Redis`数据库中的数据时,就获取该数据的`Redis Key`,然后交给`Hash`函数获取该`Redis Key`对应的`Hash`值`HashNumber`
- :three:然后我们计算得到`HashNumber Mod N`的结果为`0-(N-1)`中的某一个值
    - 若为`0`,那么我们就将去第`0`台`Redis Master`读写数据
    - 若为`i`,那么我们就将去第`i`台`Redis Master`读写数据

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303102352827.png" alt="image-20230310235244768" style="zoom:67%;" />

### 算法存在的问题

- 我们前面对算法原理进行介绍的时候知道我们的数据分区是基于`HashNumber Mod N`来实现的,当我们**需要对我们的`Redis`集群进行扩容或缩容**的时候,我们的**`N`的值就必须要改变**.而`N`的改变就会使得`Hash Number Mod N`的值发生巨大的改变,从而**导致如果我们希望`Redis`集群正常基于哈希分析算法运行**,就需要**对整个`Redis`集群**中每一台`Redis`主机上的**数据进行大量的调整**,这一过程由于**涉及到大量的数据的传输,应用,移除**会导致我们的`Redis`集群每一次**扩容缩容都会带来极大的时间以及资源开销**
    - **为什么会需要对数据做极大调整**
        - `6 mod 2 = 0`而`6 mod 3 = 0`但是`8 mod 2 = 0`而`8 mod 3 = 2`
        - 也就是说对于`6`号数据而言其**扩容前后都是存储在`0`号主机**上,但**`8`号数据**则在**扩容前存储在`0`号**主机上,**扩容后却存储到了`2`号**主机上.

## 一致性哈希环分区算法

> 前面我们提到哈希取余分区算法由于决定数据的分区时**使用到了我们当前`Redis`集群中的主机的数量**这一信息而导致对`Redis`集群的**扩容与缩容造成了较大的影响**
>
> ​		因此我们的**一致性哈希环分区算法将数据的分区与我们的`Redis`集群中的主机数进行了无关化设计**

### 算法原理

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303102156907.png" alt="image-20230310215614755" style="zoom: 50%;" />

![image-20230310215838884](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303102158003.png)

![image-20230310220018313](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303102200393.png)

### 算法的优点

- **容错性强**

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303110018992.png" alt="image-20230311001847947" style="zoom: 67%;" />

- **可动态扩展**

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303110019931.png" alt="image-20230311001913879" style="zoom: 67%;" />

### 算法存在的问题

#### 数据倾斜问题

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303110023355.png" alt="image-20230311002318256" style="zoom: 50%;" />

### 算法的改进

#### 使用虚拟节点解决数据倾斜问题

## `Redis`哈希槽分区算法

> 前面我们提到**一致性哈希环分区算法存在数据倾斜**的问题,而且**基于虚拟节点的解决方案**并非想象中那么有效,因此为了更进一步解决数据分区的种种问题设计出**哈希槽分区算法**

### 算法原理

> 首先我们的哈希槽分区算法由以下三部分组成
>
> - 数据
> - 哈希槽
> - `Redis`节点

- :one:首先我们的`Redis`集群会在启动时协调好,每一个`Redis`主机负责哪些槽位(`Slot`)

    - 如`A`负责`0-5460`槽位,`B`负责`5461-10922`槽位,`C`负责`10923-16384`槽位

- :two:然后当我们要对数据进行读写时,我们就会先获取到我们要读取的数据的`Redis Key`,然后根据下面的公式计算该数据所属的槽位(`slot`)

    - $slot=CRC16(Redis\ Key)\quad mod\quad 16384$

    :three:计算获得了我们的数据对应的槽位(`slot`)之后,通过获取控制该槽位`slot`的`redis`节点就能够知道我们的数据存储在那一一台`redis`节点上

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303110044544.png" alt="image-20230311004412481" style="zoom: 67%;" />

### 算法优点

- **容错性强**
    - 当我们某一台`Redis`主机宕机，只会导致它**所存储的槽**对应的数据**无法进行写入操作**，而其他的`Redis`主机是不会受到影响的,依然可以正常地运行,并提供其存储的数据的写入服务

- **可动态扩展**
    - 当我们需要向我们的`Redis`集群中再添加一台`Redis`主机时,我们只要协调好`4`台`Redis`主机每台负责哪些槽,然后修改槽`slot`与`Redis`主机的从属关系,将原有的数据做好迁移就能够成功完成我们的`Redis`集群扩容.并且在整个过程中由于我们的数据的`Key`总是映射到`16384`个槽`slot`上,完全不会受到我们的`Redis`集群中主机的数量的影响,因此可以很好地避免大规模的数据迁移
        - **如`A`负责`0-4095`,`B`负责`4096-8191`,`C`负责`8192-12287`,新增加的`D`负责`12288-16383`**

- **能够以很简单的方式避免数据倾斜**
    - 上面我们说可动态扩展的时候就提到了,我们对`Redis`集群进行扩容时,只需要将`16384`个槽`slot`以合理的方式兼顾均衡与最少数据迁移的方式分配给每台`Redis`主机即可,显然**这一过程是可以由我们人为控制的**,不同于**一致性哈希环的不可控性**

# `Redis`集群配置

## 集群配置流程

- :one:**配置第一台`Redis`的`redis.conf`配置文件(`vim /opt/MyRedis7/redis.conf`)**

    - **注意**:第`2-6`台参照下面的方式如法炮制即可
    
    - ```shell
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
        masterauth 58828738
        
        cluster-enabled yes
        cluster-config-file nodes-6379.conf
        cluster-node-timeout 15000
        #**********************************主要修改的部分************************************#
        ```
    
    - ```shell
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
    
- :two:**开启第一台`Redis`主机**

    - ```SHELL
        redis-server /opt/MyRedis7/redis.conf
        ```

    - **注意**:第`2-6`台参照着如法炮制即可

- :three:**借助`redis-cli`使用我们的六台`Redis`集群节点构建`Redis`集群**

    - > **注意**:我们的这个指令可以**由任何一个可以与我们的六台`Redis`集群节点联通,并且装有`Redis`的主机执行**(**就算不是`Redis`集群节点都可以**)

    - ```shell
        redis-cli -a 58828738 --cluster create --cluster-replicas 1 192.168.184.160:6379 192.168.184.161:6380 192.168.184.162:6381 192.168.184.163:6382 192.168.184.164:6383 192.168.184.165:6384
        ```

    - `--cluster-replicas 1`表示分别为每一台`Redis Master`构建一个`Slave`

    - > **注意**:我们这里给出了六个`Redis`主机的`IP端口套接字`,但是其并不是按照`Master1 Slave1 Master2 Slave2 Master3 Slave3`规则排列的这只是指定出了六台主机,而谁来作为`Master`,谁来作为`Slave`实际上还是会由我们的`Redis`通过六台机器之间进行合理的协调之后自动分配的.

        <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303111740618.png" alt="image-20230311174021562" style="zoom: 67%;" />

- :four:**经过:three:我们的六台`Redis`主机就通过相互之间的沟通组成了一个`3`个`Master`,`3`个`Slave`的集群架构**

- :five:**在任意一台集群节点上使用`redis-cli -a 58828738 -p 对应的端口`连接到该节点的`Redis-server`上,然后我们就可以使用`CLUSTER NODES`以及`CLUSTER INFO`来查看我们当前集群的各个集群节点**

    - ```shell
        CLUSTER NODES
        ```

    - ![image-20230311203742318](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112037386.png)

    - ```shell
        CLUSTER INFO
        ```

    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112039959.png" alt="image-20230311203923907" style="zoom: 80%;" />


## `Redis`集群相关的`redis.conf`配置项

### 重要部分

```shell
#是否让当前Redis-server在启动时以集群方式开启
	#常规的Redis-server是无法充当集群节点的,只有以集群节点方式开启的Redis-server才能充当集群节点
cluster-enabled yes

#指定当前Redis-server的集群相关配置文件的名称
	#注意:这个配置文件不是用来让我们的开发者进行配置的,而是我们的集群节点在运行过程中会去自动写入的.
cluster-config-file nodes-6379.conf

#用于指定集群节点的连接超时时间,单位为毫秒
cluster-node-timeout 15000
```

### 非常规部分

```shell
# The cluster port is the port that the cluster bus will listen for inbound connections on. When set to the default value, 0, it will be bound to the command port + 10000. Setting this value requires you to specify the cluster bus port when executing cluster meet.
cluster-port 0

# A replica of a failing master will avoid to start a failover if its data
# looks too old.
#
# There is no simple way for a replica to actually have an exact measure of
# its "data age", so the following two checks are performed:
#
# 1) If there are multiple replicas able to failover, they exchange messages
#    in order to try to give an advantage to the replica with the best
#    replication offset (more data from the master processed).
#    Replicas will try to get their rank by offset, and apply to the start
#    of the failover a delay proportional to their rank.
#
# 2) Every single replica computes the time of the last interaction with
#    its master. This can be the last ping or command received (if the master
#    is still in the "connected" state), or the time that elapsed since the
#    disconnection with the master (if the replication link is currently down).
#    If the last interaction is too old, the replica will not try to failover
#    at all.
#
# The point "2" can be tuned by user. Specifically a replica will not perform
# the failover if, since the last interaction with the master, the time
# elapsed is greater than:
#
#   (node-timeout * cluster-replica-validity-factor) + repl-ping-replica-period
#
# So for example if node-timeout is 30 seconds, and the cluster-replica-validity-factor
# is 10, and assuming a default repl-ping-replica-period of 10 seconds, the
# replica will not try to failover if it was not able to talk with the master
# for longer than 310 seconds.
#
# A large cluster-replica-validity-factor may allow replicas with too old data to failover
# a master, while a too small value may prevent the cluster from being able to
# elect a replica at all.
#
# For maximum availability, it is possible to set the cluster-replica-validity-factor
# to a value of 0, which means, that replicas will always try to failover the
# master regardless of the last time they interacted with the master.
# (However they'll always try to apply a delay proportional to their
# offset rank).
#
# Zero is the only value able to guarantee that when all the partitions heal
# the cluster will always be able to continue.
#
cluster-replica-validity-factor 10

# Cluster replicas are able to migrate to orphaned masters, that are masters
# that are left without working replicas. This improves the cluster ability
# to resist to failures as otherwise an orphaned master can't be failed over
# in case of failure if it has no working replicas.
#
# Replicas migrate to orphaned masters only if there are still at least a
# given number of other working replicas for their old master. This number
# is the "migration barrier". A migration barrier of 1 means that a replica
# will migrate only if there is at least 1 other working replica for its master
# and so forth. It usually reflects the number of replicas you want for every
# master in your cluster.
#
# Default is 1 (replicas migrate only if their masters remain with at least
# one replica). To disable migration just set it to a very large value or
# set cluster-allow-replica-migration to 'no'.
# A value of 0 can be set but is useful only for debugging and dangerous
# in production.
#
cluster-migration-barrier 1

# Turning off this option allows to use less automatic cluster configuration.
# It both disables migration to orphaned masters and migration from masters
# that became empty.
#
# Default is 'yes' (allow automatic migrations).
#
cluster-allow-replica-migration yes

# By default Redis Cluster nodes stop accepting queries if they detect there
# is at least a hash slot uncovered (no available node is serving it).
# This way if the cluster is partially down (for example a range of hash slots
# are no longer covered) all the cluster becomes, eventually, unavailable.
# It automatically returns available as soon as all the slots are covered again.
#
# However sometimes you want the subset of the cluster which is working,
# to continue to accept queries for the part of the key space that is still
# covered. In order to do so, just set the cluster-require-full-coverage
# option to no.
#
cluster-require-full-coverage yes

# This option, when set to yes, prevents replicas from trying to failover its
# master during master failures. However the replica can still perform a
# manual failover, if forced to do so.
#
# This is useful in different scenarios, especially in the case of multiple
# data center operations, where we want one side to never be promoted if not
# in the case of a total DC failure.
#
cluster-replica-no-failover no

# This option, when set to yes, allows nodes to serve read traffic while the
# cluster is in a down state, as long as it believes it owns the slots.
#
# This is useful for two cases.  The first case is for when an application
# doesn't require consistency of data during node failures or network partitions.
# One example of this is a cache, where as long as the node has the data it
# should be able to serve it.
#
# The second use case is for configurations that don't meet the recommended
# three shards but want to enable cluster mode and scale later. A
# master outage in a 1 or 2 shard configuration causes a read/write outage to the
# entire cluster without this option set, with it set there is only a write outage.
# Without a quorum of masters, slot ownership will not change automatically.
#
cluster-allow-reads-when-down no

# This option, when set to yes, allows nodes to serve pubsub shard traffic while
# the cluster is in a down state, as long as it believes it owns the slots.
#
# This is useful if the application would like to use the pubsub feature even when
# the cluster global stable state is not OK. If the application wants to make sure only
# one shard is serving a given channel, this feature should be kept as yes.
#
cluster-allow-pubsubshard-when-down yes

# Cluster link send buffer limit is the limit on the memory usage of an individual
# cluster bus link's send buffer in bytes. Cluster links would be freed if they exceed
# this limit. This is to primarily prevent send buffers from growing unbounded on links
# toward slow peers (E.g. PubSub messages being piled up).
# This limit is disabled by default. Enable this limit when 'mem_cluster_links' INFO field
# and/or 'send-buffer-allocated' entries in the 'CLUSTER LINKS` command output continuously increase.
# Minimum limit of 1gb is recommended so that cluster link buffer can fit in at least a single
# PubSub message by default. (client-query-buffer-limit default value is 1gb)
#
cluster-link-sendbuf-limit 0
 
# Clusters can configure their announced hostname using this config. This is a common use case for 
# applications that need to use TLS Server Name Indication (SNI) or dealing with DNS based
# routing. By default this value is only shown as additional metadata in the CLUSTER SLOTS
# command, but can be changed using 'cluster-preferred-endpoint-type' config. This value is 
# communicated along the clusterbus to all nodes, setting it to an empty string will remove 
# the hostname and also propagate the removal.
#
cluster-announce-hostname ""

# Clusters can advertise how clients should connect to them using either their IP address,
# a user defined hostname, or by declaring they have no endpoint. Which endpoint is
# shown as the preferred endpoint is set by using the cluster-preferred-endpoint-type
# config with values 'ip', 'hostname', or 'unknown-endpoint'. This value controls how
# the endpoint returned for MOVED/ASKING requests as well as the first field of CLUSTER SLOTS. 
# If the preferred endpoint type is set to hostname, but no announced hostname is set, a '?' 
# will be returned instead.
#
# When a cluster advertises itself as having an unknown endpoint, it's indicating that
# the server doesn't know how clients can reach the cluster. This can happen in certain 
# networking situations where there are multiple possible routes to the node, and the 
# server doesn't know which one the client took. In this case, the server is expecting
# the client to reach out on the same endpoint it used for making the last request, but use
# the port provided in the response.
#
cluster-preferred-endpoint-type ip

# In order to setup your cluster make sure to read the documentation
```



<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303111704946.png" alt="image-20230311170403839" style="zoom:80%;" />

# `Redis`集群的容错切换机制

## `Redis`的集群节点宕机自动错误处理机制

- **:one:`Master`宕机`Slave`上位直接顶替为新`Master`**
    - ![image-20230311212249547](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112122625.png)
- **:two:`Master`重新上线降级为`Slave`,从属于它宕机时上位的新`Master`**
    - ![image-20230311212546766](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112125831.png)

## `Redis`的集群节点主从关系手动切换

- **`Cluster Failover`**

    - > **作用**:若当前`Redis-client`连接的`Redis`节点为`Slave`节点,那么调用这个命令之后`Redis`集群就会在一定时间后自动地将该`Slave`节点和其对应的`Master`节点进行主从关系切换,切换完成后我们的这个节点就会变成`Master`节点,原来的`Master`节点就会变成`Slave`节点

    - ![](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112129430.png)

# `Redis`集群扩容与缩容

## 扩容的基本步骤(从`3`主`3`从扩容为`4`主`4`从)

- :one:**参照前面的配置步骤,开启两个`Redis`集群节点**

- :two:**在任意一台可以连接到我们的所有`8`台`Redis`主机且装了`Redis`的主机上使用如下命令将我们的扩容的两个节点中我们要作为`Master`的节点先添加到我们原先的`3`主`3`从的`Redis`集群中**

    - ```shell
        redis-cli -a 58828738 --cluster add-node 扩容Master主机的IP:Port 原始Redis集群中任意一个集群节点的IP:Port
        
        redis-cli -a 58828738 --cluster add-node 192.168.184.180:6385 192.168.184.160:6379
        ```

    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112141563.png" alt="image-20230311214117470" style="zoom: 80%;" />

- :three:**上面的步骤仅仅只是把我们的两个要添加的节点中的要作为新`Redis Master`的集群节点添加到我们的集群中,我们的集群并不会自动地为它分配`Slot`,需要我们自己手动来进行分配**

    - > **注意**:这里的`IP:Port`可以是我们的`4`主`3`从这**`7`个`Redis`集群节点中的任意一个的`IP:Port`**(`7`个是因为到这一步我们还没有把最后一个`Slave`节点添加到我们的`Redis`集群中)

    - ```shell
        redis-cli -a 58828738 --cluster reshard IP:Port
        ```

- :four:**我们知道,前面我们只是把我们的要作为一个新的`Master`节点的`Redis`主机添加到了我们的`3`主`3`从的`Redis`集群中,现在我们还剩下一个`Slave`节点要添加到我们的集群中作为新`Master`节点的`Slave`**

    - ```shell
        redis-cli -a 58828738 --cluster add-node Slave的IP:Port 从属的Master的IP:Port --cluster-slave --cluster-master-id 从属的Master节点在集群中的ID(可以通过Cluster Nodes获取)
        ```

****

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112358183.png" alt="image-20230311235847096" style="zoom:120%;" />

## 缩容(从`4`主`4`从缩容为`3`主`3`从)

- **:one:从我们的`Redis`集群中删除掉我们要缩容的`Master`节点的所有`Slave`节点**

    - ```shell
        redis-cli -a 58828738 --cluster del-node Redis集群中任意一个主机的IP:Port 我们要删除的第一个Slave节点的Redis集群中的ID(可以通过Cluster Nodes获取) [我们要删除的第二个Slave节点的Redis集群中的ID...]
        ```

- :two:**重新对我们的`Redis`集群的各个`Master`管控的槽`slot`进行分配,使得我们将要被删除的`Master`节点所管控的槽`slot`清零,并同时让其存储的数据合理地转移到剩下的`Master`节点中存储.**

    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303120006083.png" alt="image-20230312000644997" style="zoom: 80%;" />

- :three:**到现在我们已经把要删除的`Master`节点所管控的`slot`转移了,我们可以正式开始删除这个`Master`节点了**

    - ```shell
        redis-cli -a 58828738 --cluster del-node Redis集群中任意一个主机的IP:Port 我们要删除的Master节点的Redis集群中的ID(可以通过Cluster Nodes获取)
        ```

    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303120010205.png" alt="image-20230312001026141" style="zoom:80%;" />

# `Redis`集群相关命令

- **:one:`redis-cli -a 58828738 --cluster check IP:Port`**

    - > **作用**:通过指定的`Redis`集群中的节点获取该`Redis`集群中各个`Master`所控制的槽`slot`的情况

    - ![image-20230311214515927](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112145997.png)

- **:two:`redis-cli -a 58828738 --cluster add-node IP:Port IP:Port [IP:Port...] 原始集群中的任意一个主机的IP:Port`**

    - > **作用**:通过指定的`Redis`集群中的节点,向我们的`Redis`集群添加一个或多个`Master`节点

- **:three:`redis-cli -a 58828738 --cluster create --cluster-replicas IP:Port [IP:Port IP:Port...]`** 

    - > **作用**:根据我们给出的`Redis`节点,让我们的`Redis`自动地构建起一个`Redis`集群来

- **:four:`redis-cli -a 58828738 --cluster reshard 集群中的任意一个节点的IP:Port`**

    - > **作用**:通过指定的`Redis`集群中的节点,启动槽`slot`重新分配机制

- :five:**`Cluster Failover`命令**

    - > **作用**:启动集群节点的主从切换机制
        >
        > - 若当前`redis-client`连接的`redis-server`为一个`Slave`节点,那么`Redis`集群在收到这个指令后就会将这个`Slave`节点升级为`Master`,而原本的`Master`则降级为这个新`Master`的`Slave`

    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112129430.png" alt="image-20230311212938320" style="zoom:67%;" />

- :six:**`Cluster Nodes`命令**

    - > **作用**:用于获取当前`redis-client`连接到的`Redis`集群中各个节点的情况

    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112037386.png" alt="image-20230311203742318" style="zoom: 74%;" />

- :seven:**`Cluster Info`命令**

    - > **作用**:用于获取当前`redis-client`连接到的`Redis`集群中的对应`Redis`节点的集群相关信息

    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112039959.png" alt="image-20230311203923907" style="zoom: 80%;" />

- :nine:**`Cluster KeySlot Key名`**

    - > **作用**:获取指定的某个`Key`经过**$slot=CRC16(Key)\quad mod\quad 16384$**后得到的结果`slot`

    - ![image-20230311205229579](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112052629.png)

- **`Cluster CountKeysInSlot {0-16383}中的任意一个值`**

    > - **作用**:用于获取指定的槽`slot`的使用情况,如果当前`Redis`集群的这个槽已经存储了数据那么返回`1`,若没有则返回`0`

# `Redis`集群相关问题汇总

## :one:`CRC16`算法的生成范围为`0-65535`为什么`Redis`集群的最大槽位`slot`数要取`Mod 16384`使其设置为`16384`呢?

- :one:**`Redis Master`节点必须按照一定的时间间隔持续发送心跳数据包给其他的一些节点,设置为`16384`可以缩减这个心跳数据包的大小从而减轻网络负担**
    - 这个心跳数据包中有一个重要的部分,`unsigned char myslots[CLUSTER_SLOTS/8]`,这个部分被实现为一个`Redis BitMap`其作用为存储当前`Redis`节点所管控的槽`slot`,例如若当前`Redis`节点负责`0-4095`范围内的槽位,那么就可以让该`BitMap`的前`4096`个`bit`为`1`这样就能够很好地呈现出该`Redis`节点所负责的槽位.而如果我们设置最大槽位数为`65536`,那么显然我们的`unsigned char myslots[CLUSTER_SLOTS/8]`这个`BitMap`将要拥有`65536`个`bit`,也就是$\frac{65536\ bit}{8\ bit}÷1024\ byte=8\ KB$.而如果我们只设置为`16384`,那么`BitMap`只需要拥有`16384`个`bit`也就是$\frac{16384\ bit}{8\ bit}÷1024\ byte=2\ KB$.也就是说如果我们设置为`16384`可以节约`6 KB`的空间

- :two:**设置为`16384`能够保证`Redis`集群中节点数不超过`1000`的情况下每台`Redis`主机能够分到一个不小的槽位数**
    - `Redis`官方的建议中提到`Redis`集群中的主机数最大应为`1000`,如果我们的`Redis`集群中刚好有`1000`个`Redis`主机的话,那么如果平均分配的话,那么我们的每一台`Redis`主机大概分配到$\frac{16384}{1000}≈16$个槽位.

- :three:**槽位数越小在`Redis`主机数`N`固定的情况下`slot/N`越小,`BitMap`的压缩率越高**
    - 对于用于指示当前`Redis`主机所控制的槽位这一任务而言,我们的`Redis`可以在传输前对我们的`BitMap`进行压缩.比如在`16384`的情况下若一台主机控制的是`0-4095`这个范围内的槽位,那么我们就可以只用一个`4096`个`bit`的全为`1`的位图进行表示,而如果是`4095-8191`,那么我们可以用`8192`个`bit`的全为`1`的位图进行表示(当其他主机受到心跳数据包后可以直接通过补`0 bit值`的方式将这个`BitMap`填充回`16384`).随着每个`Redis`主机控制的槽位数越小(`slot/N`越小)我们上述的压缩带来的空间节约越多.此时就有一件很显然的事情,在两个`Redis`集群主机数都为`100`的情况下中设置槽位数为`65536`时每个主机要至少要管控`655`个槽位,而`16384`情况下则只有`163`个槽位.很显然我们上述的压缩算法在设置为`16384`的情况下能够带来更多的空间节约


### `Redis`集群的心跳数据包

#### 重要部分

```c
typedef struct {
    unsigned char myslots[CLUSTER_SLOTS/8];
} clusterMsg;
```

#### 完整数据包

```c
typedef struct {
    char sig[4];        /* Signature "RCmb" (Redis Cluster message bus). */
    uint32_t totlen;    /* Total length of this message */
    uint16_t ver;       /* Protocol version, currently set to 1. */
    uint16_t port;      /* TCP base port number. */
    uint16_t type;      /* Message type */
    uint16_t count;     /* Only used for some kind of messages. */
    uint64_t currentEpoch;  /* The epoch accordingly to the sending node. */
    uint64_t configEpoch;   /* The config epoch if it's a master, or the last
                               epoch advertised by its master if it is a
                               slave. */
    uint64_t offset;    /* Master replication offset if node is a master or
                           processed replication offset if node is a slave. */
    char sender[CLUSTER_NAMELEN]; /* Name of the sender node */
    unsigned char myslots[CLUSTER_SLOTS/8];
    char slaveof[CLUSTER_NAMELEN];
    char myip[NET_IP_STR_LEN];    /* Sender IP, if not all zeroed. */
    uint16_t extensions; /* Number of extensions sent along with this packet. */
    char notused1[30];   /* 30 bytes reserved for future usage. */
    uint16_t pport;      /* Sender TCP plaintext port, if base port is TLS */
    uint16_t cport;      /* Sender TCP cluster bus port */
    uint16_t flags;      /* Sender node flags */
    unsigned char state; /* Cluster state from the POV of the sender */
    unsigned char mflags[3]; /* Message flags: CLUSTERMSG_FLAG[012]_... */
    union clusterMsgData data;
} clusterMsg;
```

### `Redis`创始人`Antirez`对这个问题的回复

![image-20230311005856675](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303110058741.png)

## :two:`Redis`集群为什么无法保证强一致性呢？

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303102251261.png" alt="image-20230310225149194" style="zoom:67%;" />

## :three:当我们使用`Redis-client`客户端直接连接到`Redis Master 1`但是却在这个客户端中写入存储在`Redis Master 2`中的数据,这会导致什么现象发生?怎么解决这一问题?

### 现象

![image-20230311204234472](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112042525.png)

### 解决方案

- **通过在我们的`Redis-cli`命令中添加`-c`参数来实现路由式连接**

    - ```shell
        redis-cli -a 58828738 -p 对应的端口 -c
        ```

    - ![image-20230311204522592](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303112045640.png)

## :four:在`Redis`集群环境下,如果我们使用类似于`MSet`,`MGet`这类可以同时操作多个`Key`的操作命令去同时操作一些来自于不同的`Redis Master`主机的`Key`与操作同一个`Redis Master`主机上的`Key`有什么区别?

### 区别

- **操作同一个`Redis Matser`下的多个`Key`是不会出现问题的,但是如果操作的是不同的`Redis Master`上的`Key`则会导致如下问题**

![image-20230312001528848](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303120015903.png)

### 使用通识占位符解决这一问题
