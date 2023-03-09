# 什么是持久化？

# `RDB RedisDataBase`持久化

## `RDB`持久化的基本原理

> ​		我们知道`Redis`是一个基于内存的数据库,所有的数据都被保存在内存中, `RDB`作为一种持久化方式,其原理就是**每隔指定的时间间隔**,将`Redis`运行时所占用的内存上的所有的数据库相关的**数据全部持久化存储到我们的硬盘**上,这些数据在文件系统下的表现为**一个`dump.rdb`文件**.当我们的服务器宕机后,我们就可以使**用特定的命令去根据这个`dump.rdb`文件**中存储的数据,将我们的`Redis`数据库**恢复到宕机之前的状态.**
>
> <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303061526138.png" alt="image-20230306152609075" style="zoom:80%;" />

****

## `RDB`持久化的配置

### `redis.conf`配置文件中的`RDB`持久化相关配置部分

```shell
################################ SNAPSHOTTING  ################################

# Save the DB to disk.
#
# save <seconds> <changes> [<seconds> <changes> ...]
#
# Redis will save the DB if the given number of seconds elapsed and it
# surpassed the given number of write operations against the DB.
#
# Snapshotting can be completely disabled with a single empty string argument
# as in following example:
#
# save ""
#
# Unless specified otherwise, by default Redis will save the DB:
#   * After 3600 seconds (an hour) if at least 1 change was performed
#   * After 300 seconds (5 minutes) if at least 100 changes were performed
#   * After 60 seconds if at least 10000 changes were performed
#
# You can set these explicitly by uncommenting the following line.
#
# save 3600 1 300 100 60 10000

# By default Redis will stop accepting writes if RDB snapshots are enabled
# (at least one save point) and the latest background save failed.
# This will make the user aware (in a hard way) that data is not persisting
# on disk properly, otherwise chances are that no one will notice and some
# disaster will happen.
#
# If the background saving process will start working again Redis will
# automatically allow writes again.
#
# However if you have setup your proper monitoring of the Redis server
# and persistence, you may want to disable this feature so that Redis will
# continue to work as usual even if there are problems with disk,
# permissions, and so forth.
stop-writes-on-bgsave-error yes

# Compress string objects using LZF when dump .rdb databases?
# By default compression is enabled as it's almost always a win.
# If you want to save some CPU in the saving child set it to 'no' but
# the dataset will likely be bigger if you have compressible values or keys.
rdbcompression yes

# Since version 5 of RDB a CRC64 checksum is placed at the end of the file.
# This makes the format more resistant to corruption but there is a performance
# hit to pay (around 10%) when saving and loading RDB files, so you can disable it
# for maximum performances.
#
# RDB files created with checksum disabled have a checksum of zero that will
# tell the loading code to skip the check.
rdbchecksum yes

# Enables or disables full sanitization checks for ziplist and listpack etc when
# loading an RDB or RESTORE payload. This reduces the chances of a assertion or
# crash later on while processing commands.
# Options:
#   no         - Never perform full sanitization
#   yes        - Always perform full sanitization
#   clients    - Perform full sanitization only for user connections.
#                Excludes: RDB files, RESTORE commands received from the master
#                connection, and client connections which have the
#                skip-sanitize-payload ACL flag.
# The default should be 'clients' but since it currently affects cluster
# resharding via MIGRATE, it is temporarily set to 'no' by default.
#
# sanitize-dump-payload no

# The filename where to dump the DB
dbfilename dump.rdb

# Remove RDB files used by replication in instances without persistence
# enabled. By default this option is disabled, however there are environments
# where for regulations or other security concerns, RDB files persisted on
# disk by masters in order to feed replicas, or stored on disk by replicas
# in order to load them for the initial synchronization, should be deleted
# ASAP. Note that this option ONLY WORKS in instances that have both AOF
# and RDB persistence disabled, otherwise is completely ignored.
#
# An alternative (and sometimes better) way to obtain the same effect is
# to use diskless replication on both master and replicas instances. However
# in the case of replicas, diskless is not always an option.
rdb-del-sync-files no

# The working directory.
#
# The DB will be written inside this directory, with the filename specified
# above using the 'dbfilename' configuration directive.
#
# The Append Only File will also be created inside this directory.
#
# Note that you must specify a directory here, not a file name.
dir ./
```

****

### 相关配置项介绍

|            配置项             |                             作用                             |                             示例                             |                             备注                             |
| :---------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|            `save`             | 用于**设置**`RDB`持久化写`dump.rdb`持久化文件的**时间间隔**  | `save 300 10 3000 1 20 500`<br />这个命令的含义为若任意的`300`秒以内有`10`次写入操作**或者**`3000`秒内有`1`次写入操作**或者**`20`秒内有`500`次写入操作,那么就**触发一次持久化写入操作** | 若我们配置文件中**没有设置`save`属性**,那么我们的`Redis`会使用其**默认的`RDB`持久化时间间隔设置** |
| `stop-writes-on-bgsave-error` |                                                              |                                                              |                                                              |
|       `rdbcompression`        | 用于指定**是否**使用`LZF`算法对我们的`dump.rdb`文件中的**数据进行压缩**,以**缩小**其所**占据的存储空间** |                     `rdbcompression yes`                     | 如果我们设置为`yes`,那么由于需要使用算法来进行压缩,所以肯定**会带来`CPU`资源的消耗**. |
|         `rdbchecksum`         |     用于指定**是否开启**持久化文件的**`CRC64`检错机制**      | `rdbchecksum yes`<br />开启了检错机制后若我们的持久化文件在存储,复制粘贴等等过程中**出现问题数据发生了改变**,我们的`Redis`就能够**通过检错码发现错误**. |                                                              |
|         `dbfilename`          | 用于指定我们的**持久化文件的名称**,这个属性的默认值为`dump.rdb` |                  `dbfilename tangling.rdb`                   |                                                              |
|     `rdb-del-sync-files`      |                                                              |                                                              |                                                              |
|             `dir`             |        用于**指定**用于保存我们的持久化文件的**目录**        |  `dir /opt/mypersistentfile`<br />`dir ./mypersistentfile`   | 如果我们使用的是**相对路径**那么这个相对路径的**根路径为我们的`redis`的`.conf`配置文件所在的目录** |

## 手动触发的`RDB`持久化

### 基于`Save`,`BgSave`命令的手动触发

> 当我们在`Redis`客户端调用了`SAVE`或`BGSAVE`命令,都会马上触发一次持久化`RDB`文件的写入,但是`SAVE`与`BGSAVE`有所不同
>
> - **`SAVE`**
>     - 当我们使用了`SAVE`命令,那么我们的**`Redis-server`就会进入阻塞状态**,**阻塞所有的客户端发起的命令调用**,**直到**我们的`SAVE`命令触发的**持久化写入操作完成**才会**解除阻塞**
>     - 并且**阻塞期间**客户端发起的**所有命令都会直接被丢弃**,即便**阻塞解除也不会重新执行**
> - **`BGSAVE`**
>     - 当我们调用了`BGSAVE`命令,我们的**`Redis-server`也会进入阻塞状态**不过这个阻塞状态是用于**开启一个线程来负责持久化写入操作**,当子进程**开启完毕**后,`Redis-server`就会**解除阻塞**.在**子进程**进行持久化写入期间,我们的**`Redis`客户端依然能够正常地执行命令(除了子进程开辟时)**.
>
> **注意**:`BGSAVE`保存的是**命令调用之前**的`Redis`数据库状态,而我们在`BGSAVE`**执行期间做出的修改**是**不会被持久化**的

****

### `flushdb`,`flushall`,`shutdown`命令的自动触发

> - :one:当我们**使用`flushdb`或`flushall`命令**时,当它们执行完毕后就会**自动触发一次**持久化文件的写入(当然这个**文件中是存储的命令执行完成之后的数据库状态**,而**不是命令执行前的状态**)
> - :two:当我们**使用`ShutDown`命令**关闭我们的`Redis`服务时**会自动在`ShutDown`命令执行前**自动**触发一次持久化文件的写入操作**

## `BGSAVE`的原理

- :one:`Redis`创建子进程以后，**不会**在内存上把`Redis`的数据的额外**完整地复制一份**，主进程与子进程实际上是**共享数据**的。**主进程**继续对外**提供读写服务**,**子进程**负责**持久化**服务。
- :two:虽然`Redis`不会复制一份数据，但是`Redis`会**把主进程中使用到的所有内存页的权限都设为只读**，主进程和子进程访问数据的指针都指向同一内存地址。
- :three:**主进程发生写操作**时，因为权限已经设置为只读了，所以会**触发页异常中断**（`page-fault`）。在**中断处理**中，需要**被写入的内存页面会复制一份**，**复制出来的数据页交给子进程使用**，而**原本的**只读数据页的**权限就会被修改为读写**,然后**主进程**就可以**写入数据**。
- :four:也就是说，在进行`IO`操作写盘的过程中，对于**没有改变的数据**，主进程和子进程**资源共享**；只有在出现了**需要变更的数据**时，才**进行复制**操作。

****

## 基于`RDB`持久化文件恢复`Redis`数据库

> 当我们的**`Redis-server`启动时**会**自动去查看**我们的用于启动的**配置文件所在的目录下是否有`RDB`持久化文件**,如果有就会**根据这个文件来恢复我们的`Redis`数据库**

## `RDB`持久化机制的缺点

> **注意**:**只有`SAVE`命令会阻塞主进程,`BGSAVE`与其他的自动触发都是由子进程处理持久化**

### :one:可能会丢失一些最新的数据

> - `RDB`持久化的**触发方式**有以下三种
>     - `SAVE`,`BGSAVE`命令**手动触发**
>     - `SHUTDOWN`,`FLUSHDB`,`FLUSHALL`**自动触发**
>     - 根据配置文件设置的**规则触发**
> - 这就会导致如果我们的**最新的`x`次写入操作没有触发`RDB`持久化**,而**恰巧**就在这个时间点上我们的`Redis`**服务器宕机**了,这就会导致我们的持久化文件中并**没有包括这最新的`x`次写入操作**

### :two:子进程开辟时耗时

- 我们前面就提到了,我们的`Redis-server`主进程在接到了`BGSAVE`命令后会**阻塞一小段时间**用于**开启一个子进程**用于处理持久化操作.这一阻塞可能**时间不长**,但是如果我们的持久化操作如果做的**过于频繁**,仍然会对我们的`Redis-server`**带来一定的性能问题**

## 基于`redis-check-rdb`应用的``RDB`持久化文件的修复

> ​		我们考虑这样一种场景,我们的`Redis-Server`**正在进行持久化**写入操作,在**写入过程中突然我们的`Redis`服务器宕机**了,那么这种情况下,很可能我们的**`dump.rdb`持久化文件会发生破损**,从而导致我们的`Redis`**无法借助于这个破损的`dump.rdb`持久化文件恢复**我们的`Redis`**数据库**.这个时候**为了能够挽救一些数据**,我们可能就会希望去**对这个`dump.rdb`进行修复**,以**牺牲小部分**数据的代价,**挽回大部分**数据.

```shell
cd /usr/local/bin
./redis-check-rdb /MyRedis/dump.rdb
```

# `RDB`持久化机制的禁用

> **在`redis`的`.conf`配置文件中设置`save ""`**

> **注意**:这里禁用的只是`RDB`持久化机制的自动触发,我们调用`SAVE`或者`BGSAVE`命令依然能够触发`RDB`持久化

```shell
save ""
```

# `AOF`持久化的开启

> **在`redis`的`.conf`配置文件中设置`appendonly yes`**

```shell
appendonly yes
```

# `AOF Append Only File`持久化

## `AOF`持久化的工作原理与工作流程

### 工作原理

> 

### 工作流程

- :one:首先我们的`Redis-server`服务接收到某一个`Redis-client`的**命令调用**
- :two:若这个命令**涉及到数据库数据写入**,那么`Redis-server`就会将这个命令**写入**到我们的**内存上的`AOF缓冲区`**
- :three:一直**重复**上述的两个过程,直到**触发`AOF缓冲区持久化`**
- :four:`AOF缓冲区`中的数据将会被**写入到我们的磁盘**上
    - `Redis`有**三种不同的写入策略**
- :five:**重复**上述过程,直到我们的`appenonly.aof`**持久化文件膨胀到一定程度**,触发持久化文件的**重写压缩机制**
- :six:`Redis`会对我们的`appendonly.aof`持久化文件中的**指令进行缩减**,以达到压缩的目的
    - 如我们有着这样的**命令执行流程**
        - `set k1 v1`
        - `set k2 v2`
        - `set k1 v3`
        - `set k1 v4`
    - 那么我们的三条对于`k1`的写入操作就会被缩减为**只保存最后一条`set k1 v4`**

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303061902872.png" alt="image-20230306190209762"  />

## `AOF`缓冲区的三大持久化策略

> - `Always`
>     - 每执行一个写命令就会阻塞我们的`Redis-server`主进程,然后将我们的这个命令写入到`AOF缓冲区`,并且将`AOF缓冲区`中的数据持久化到磁盘,仅当这两步都完成后我们的`Redis-server`主进程才会解除阻塞
> - `everysec`
>     - `Redis`在启动时,我们的`Redis-server`主进程就会自动开启一个子进程用于处理`AOF缓冲区`中数据的持久化到磁盘的过程.当然命令写入`AOF缓冲区`还是由我们的`Redis-server`主进程负责的
> - `no`
>     - 

## `AOF`对应的`.conf`文件配置项

|             属性              |                             作用                             |
| :---------------------------: | :----------------------------------------------------------: |
|         `appendonly`          |             用于指定**是否开启`AOF`持久化机制**              |
|       `appendfilename`        | 用于指定持久化到磁盘中的`AOF`**持久化**数据在文件系统中对应的**文件的名称** |
|        `appenddirname`        | 用于指定保存`AOF`持久化文件的**存储目录**<br />注意我们的`.conf`配置文件中**还指定了`dir`属性**,因此**实际上**我们的`AOF`持久化文件的存储路径为<br />**`dir/appenddirname/appendfilename后缀`<br />**这个属性`Redis6`没有 |
|         `appendfsync`         | 用于指定**`AOF缓冲区`**中数据**持久化到磁盘**使用的策略<br />**有`Always`,`everysec`,`no`三种选择** |
|  `no-appendfsync-on-rewrite`  | 用于指定当我们的`AOF`持久化文件的重写压缩进程正在运行时<br />我们的`Redis`主进程是**按照原有的规则**将`AOF缓冲区`中的数据**持久化到磁盘**上(**`no`**)<br />还是**暂停**`AOF缓冲区`数据的**持久化**,直到重写压缩执行完毕(**`yes`**) |
| `auto-aof-rewrite-percentage` |  用于**指定一个百分比**,这个百分比关系到自动重写压缩的触发   |
|  `auto-aof-rewrite-min-size`  | 用于**指定一个文件大小**<br />当我们的增量`AOF`文件的大小**大于**我们这里**指定的文件大小**,并且是我们**上一次重写后的`AOF`文件的大小的`percentage`倍**时**触发自动重写压缩** |
|     `aof-load-truncated`      |                                                              |
|    `aof-use-rdb-preamble`     |                                                              |
|    `aof-timestamp-enabled`    |                                                              |

```shell
############################## APPEND ONLY MODE ###############################

# By default Redis asynchronously dumps the dataset on disk. This mode is
# good enough in many applications, but an issue with the Redis process or
# a power outage may result into a few minutes of writes lost (depending on
# the configured save points).
#
# The Append Only File is an alternative persistence mode that provides
# much better durability. For instance using the default data fsync policy
# (see later in the config file) Redis can lose just one second of writes in a
# dramatic event like a server power outage, or a single write if something
# wrong with the Redis process itself happens, but the operating system is
# still running correctly.
#
# AOF and RDB persistence can be enabled at the same time without problems.
# If the AOF is enabled on startup Redis will load the AOF, that is the file
# with the better durability guarantees.
#
# Please check https://redis.io/topics/persistence for more information.

appendonly no

# The base name of the append only file.
#
# Redis 7 and newer use a set of append-only files to persist the dataset
# and changes applied to it. There are two basic types of files in use:
#
# - Base files, which are a snapshot representing the complete state of the
#   dataset at the time the file was created. Base files can be either in
#   the form of RDB (binary serialized) or AOF (textual commands).
# - Incremental files, which contain additional commands that were applied
#   to the dataset following the previous file.
#
# In addition, manifest files are used to track the files and the order in
# which they were created and should be applied.
#
# Append-only file names are created by Redis following a specific pattern.
# The file name's prefix is based on the 'appendfilename' configuration
# parameter, followed by additional information about the sequence and type.
#
# For example, if appendfilename is set to appendonly.aof, the following file
# names could be derived:
#
# - appendonly.aof.1.base.rdb as a base file.
# - appendonly.aof.1.incr.aof, appendonly.aof.2.incr.aof as incremental files.
# - appendonly.aof.manifest as a manifest file.

appendfilename "appendonly.aof"

# For convenience, Redis stores all persistent append-only files in a dedicated
# directory. The name of the directory is determined by the appenddirname
# configuration parameter.

appenddirname "appendonlydir"

# The fsync() call tells the Operating System to actually write data on disk
# instead of waiting for more data in the output buffer. Some OS will really flush
# data on disk, some other OS will just try to do it ASAP.
#
# Redis supports three different modes:
#
# no: don't fsync, just let the OS flush the data when it wants. Faster.
# always: fsync after every write to the append only log. Slow, Safest.
# everysec: fsync only one time every second. Compromise.
#
# The default is "everysec", as that's usually the right compromise between
# speed and data safety. It's up to you to understand if you can relax this to
# "no" that will let the operating system flush the output buffer when
# it wants, for better performances (but if you can live with the idea of
# some data loss consider the default persistence mode that's snapshotting),
# or on the contrary, use "always" that's very slow but a bit safer than
# everysec.
#
# More details please check the following article:
# http://antirez.com/post/redis-persistence-demystified.html
#
# If unsure, use "everysec".

# appendfsync always
appendfsync everysec
# appendfsync no

# When the AOF fsync policy is set to always or everysec, and a background
# saving process (a background save or AOF log background rewriting) is
# performing a lot of I/O against the disk, in some Linux configurations
# Redis may block too long on the fsync() call. Note that there is no fix for
# this currently, as even performing fsync in a different thread will block
# our synchronous write(2) call.
#
# In order to mitigate this problem it's possible to use the following option
# that will prevent fsync() from being called in the main process while a
# BGSAVE or BGREWRITEAOF is in progress.
#
# This means that while another child is saving, the durability of Redis is
# the same as "appendfsync no". In practical terms, this means that it is
# possible to lose up to 30 seconds of log in the worst scenario (with the
# default Linux settings).
#
# If you have latency problems turn this to "yes". Otherwise leave it as
# "no" that is the safest pick from the point of view of durability.

no-appendfsync-on-rewrite no

# Automatic rewrite of the append only file.
# Redis is able to automatically rewrite the log file implicitly calling
# BGREWRITEAOF when the AOF log size grows by the specified percentage.
#
# This is how it works: Redis remembers the size of the AOF file after the
# latest rewrite (if no rewrite has happened since the restart, the size of
# the AOF at startup is used).
#
# This base size is compared to the current size. If the current size is
# bigger than the specified percentage, the rewrite is triggered. Also
# you need to specify a minimal size for the AOF file to be rewritten, this
# is useful to avoid rewriting the AOF file even if the percentage increase
# is reached but it is still pretty small.
#
# Specify a percentage of zero in order to disable the automatic AOF
# rewrite feature.

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# An AOF file may be found to be truncated at the end during the Redis
# startup process, when the AOF data gets loaded back into memory.
# This may happen when the system where Redis is running
# crashes, especially when an ext4 filesystem is mounted without the
# data=ordered option (however this can't happen when Redis itself
# crashes or aborts but the operating system still works correctly).
#
# Redis can either exit with an error when this happens, or load as much
# data as possible (the default now) and start if the AOF file is found
# to be truncated at the end. The following option controls this behavior.
#
# If aof-load-truncated is set to yes, a truncated AOF file is loaded and
# the Redis server starts emitting a log to inform the user of the event.
# Otherwise if the option is set to no, the server aborts with an error
# and refuses to start. When the option is set to no, the user requires
# to fix the AOF file using the "redis-check-aof" utility before to restart
# the server.
#
# Note that if the AOF file will be found to be corrupted in the middle
# the server will still exit with an error. This option only applies when
# Redis will try to read more data from the AOF file but not enough bytes
# will be found.
aof-load-truncated yes

# Redis can create append-only base files in either RDB or AOF formats. Using
# the RDB format is always faster and more efficient, and disabling it is only
# supported for backward compatibility purposes.
aof-use-rdb-preamble yes

# Redis supports recording timestamp annotations in the AOF to support restoring
# the data from a specific point-in-time. However, using this capability changes
# the AOF format in a way that may not be compatible with existing AOF parsers.
aof-timestamp-enabled no
```

## 基于`AOF`持久化文件恢复`Redis`数据库

> 当我们的**`Redis-server`启动时**会**自动去查看**我们的用于启动的**配置文件所在的目录下的`appenddirname`目录下是否有`AOF`持久化文件**,如果有就会**根据这个文件来恢复我们的`Redis`数据库**

## `AOF缓冲区` `EverySec`持久化写入磁盘策略对于`Redis-server`主进程的阻塞问题

- **在`EverySec`策略下**,我们的`AOF`缓冲区的**持久化写入的管理**是由我们的`Redis-server`**主进程开辟的一个子进程进行**的
- 因此按道理来说我们的`AOF缓冲区`的**持久化写入并不会阻塞我们的`Redis-server`主进程**(主进程**只有**在把命令**写入`AOF缓冲区`时**会被**短暂地阻塞**)
- 但是实际上我们的子进程每一次写入

## 基于`redis-check-aof`应用的`AOF`持久化文件的修复

> ​	我们考虑这样一种场景,我们的`Redis-Server`**正在进行`AOF`持久化缓冲区的**持久化写入操作,在**写入过程中突然我们的`Redis`服务器宕机**了,那么这种情况下,很可能我们的**`AOF`持久化文件会发生破损**,从而导致我们的`Redis`**无法借助于这个破损的`AOF`持久化文件恢复**我们的`Redis`**数据库**.这个时候**为了能够挽救一些数据**,我们可能就会希望去**对这个`AOF`持久化文件进行修复**,以**牺牲小部分**数据的代价,**挽回大部分**数据.

> `redis-check-aof [--fix|--truncate-to-timestamp $timestamp] <file.manifest|file.aof>`

```shell
cd /usr/local/bin
./redis-check-aof --fix /MyRedis/appenddirname/appendfilename.aof.1.incr.aof
```

> **注意**:若不添加`--fix`就**只会校验**指定的`AOF`持久化文件**是否正常**,**不会修复**

## `AOF`持久化机制的缺点

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303062049252.png" alt="image-20230306204908152" style="zoom:150%;" />

## 手动启动`AOF`持久化文件的重写压缩

> ​		我们知道我们的`AOF`持久化文件默认情况下会在特定的时候进行**自动的压缩,**但是有时候我们希望**手动**地让`AOF`持久化文件**开启压缩**.**`BgRewriteAof`命令就能够实现这一目的**

```shell
BGREWRITEAOF
```

## `AOF`持久化文件的重写压缩的工作流程

- :one:首先我们的`Redis`**触发**`AOF`持久化文件的**重写压缩机制**,此时我们的`Redis-server`主进程就会**开辟出一个子进程**用于负责**管理重写压缩过程**
- :two:然后我们的子进程会去**读取**我们当前`Redis`数据库中的**所有键值对**,然后**生成能够产生这些键值对的一条条的命令**
    - 这个过程是**边生成**命令**边写入**命令到我们内存中的`AOF重写缓冲区`中的
    - **注意**:在此期间我们的`Redis-server`主进程收到的**命令**会被**同步写入到`AOF缓冲区`以及`AOF重写缓冲区`**以保证**重写压缩完成后**我们的**`AOF重写缓冲区`中的命令与我们的`Redis`数据库当前的状态保持一致**
- :three:当**重写压缩完成**,我们的`Redis-server`**主进程**就会被**阻塞**
- **:four:然后**
    - 我们的子进程就会将**重写产生的`AOF重写缓冲区`中的数据**就会写入大我们的文件系统中**的`appendonly.aof.2.base.aof`文件中**
    - `appendonly.aof.1.base.rdb`文件则**会被`Redis-server`主进程直接删除**.
    - `appendonly.aof.1.incr.aof`文件也会**被`Redis-server`主进程删除**,并被`appendonly.aof.2.incr.aof`这个**空**文件**取代**
- :four:然后我们的`Redis-server`主进程就会**解除阻塞**
- :four:**至此我们的整个重写压缩流程就结束了**

## `Redis7`与`Redis6`的`AOF`机制的区别

- :one:**持久化文件存储路径不同**
    - `Redis6`下没有`appenddirname`属性,因此`AOF`持久化文件的保存路径为`dir/appendfilename`
    - `Redis7`下的`AOF`持久化文件的保存路径为``dir/appenddirname/appendfilename后缀``

- :two:**持久化文件的数目不同**
    - `Redis6`下只有**一个**`appendfilename`持久化文件
    - `Redis7`下则有**三个**持久化文件,它们分别为
        - `appendonly.aof.1.base.rdb`**基本文件**
        - `appendonly.aof.1.incr.aof`**增量文件**
        - `appendonly.aof.manifest`**清单文件**
    - 注意:基本文件和增量文件并非一直是这个名字,随着`Redis`数据库数据的写入,会从`1`变为`2`变为`3`…..

# `AOF`持久化机制的关闭

> **在`redis`的`.conf`配置文件中设置`appendonly no`**

> **注意**:这只是关闭了`AOF`持久化机制的自动触发,关闭后我们调用`BgRewriteAof`依然可以生成`AOF`持久化文件只不过这时`incr`增量文件不再起作用,只有`base`基础文件起作用了

```shell
appendonly no
```

# `RDB`与`AOF`全部关闭

> 同时设置`save ""`与`appendonly no`

```shell
save ""
appendonly no
```

# `AOF`的`aof-use-rdb-preamble`属性

> **设置`aof-use-rdb-preamble yes`会导致`base`基础`AOF`持久化文件发生变化**

## `aof-use-rdb-preamble no`(`.base.aof`)

> 在这种情况下,我们的`base`基础`AOF`持久化文件为**`appendonly.aof.1.base.aof`文件**,经过**一次重写压缩**后变为`appendonly.aof.2.base.aof`文件,经过**`i`次重写压缩**后变为`appendonly.aof.i.base.aof`文件

> `base`基础`AOF`持久化文件中的**数据存储格式遵循`AOF`持久化标准格式**,每一次**触发`AOF`持久化文件重写压缩**生成的都是**大量**可以用于恢复数据库的**`Redis`命令集**

## `aof-use-rdb-preamble yes`(`.base.rdb`)

> 在这种情况下,我们的`base`基础`AOF`持久化文件为**`appendonly.aof.1.base.rdb`文件**,经过**一次重写压缩**后变为`appendonly.aof.2.base.rdb`文件,经过**`i`次重写压缩**后变为`append only.aof.i.base.rdb`文件

> `base`基础`AOF`持久化文件中的**数据存储格式遵循`RDB`持久化标准格式**,每一次**触发**`AOF`持久化文件**重写压缩**都**相当于是触发了一次对数据库的`RDB`快照保存**

# 持久化相关问题汇总

## :one:`Redis`数据库在启动时到底使用`RDB`还是`AOF`进行数据库恢复?

### 当**启动时**指定的配置文件中的`save`没有被设置为`“”`但`appendonly=no`时

> 这种情况下即便我们既存在`RDB`持久化文件也存在`AOF`持久化文件,即便`AOF`持久化文件中的数据比`RDB`持久化文件中的新,**反正就是无论什么情况下都会根据`RDB`持久化文件来对我们的`Redis`数据库进行恢复**,如果即便`RDB`持久化配置文件不存在也不会使用`AOF`持久化文件进行恢复

### 当**启动时**指定的配置文件中的`save`被设置为`“”`但`appendonly=yes`时

> 这种情况下即便我们既存在`RDB`持久化文件也存在`AOF`持久化文件,即便`RDB`持久化文件中的数据比`AOF`持久化文件中的新,**反正就是无论什么情况下都会根据`AOF`持久化文件来对我们的`Redis`数据库进行恢复**,即便`AOF`持久化配置文件不存在也不会使用`RDB`持久化文件进行恢复

### 当**启动时**指定的配置文件中的`save`没有被设置为`“”`且`appendonly=yes`时

> **存在`AOF`持久化文件**就**用**`AOF`持久化文件进行`Redis`数据库恢复,**不存在**`AOF`持久化文件**才**使用`RDB`持久化文件进行`Redis`数据库恢复,**即便`RDB`中的数据比`AOF`中的新**

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303062329450.png" alt="image-20230306232945351" style="zoom: 50%;" />

## :two:如果我们本来`AOF`持久化机制是关闭的,但是突然我们想要开启`AOF`持久化机制,那么我们的`AOF`持久化机制如何保证原本存储在`RDB`中的数据被正确恢复呢?

### `Redis-server`正在运行期间设置

#### `aof-use-rdb-preamble no`情况下

> 当`Redis-server`主进程接收到修改命令时会**直接进入阻塞状态**,然后我们的`Redis`会**根据我们的数据库当前的状态生成一个`appendonly.aof.1.base.aof`文件**,这个文件中就**存储**着可以用来**恢复我们的`Redis`数据库的`AOF`格式的数据**

#### `aof-use-rdb-preamble yes`情况下

> 当`Redis-server`主进程接收到修改命令时会**直接进入阻塞状态**,然后我们的`Redis`会**根据我们的数据库当前的状态生成一个`appendonly.aof.1.base.rdb`文件**,这个文件中就**存储**着可以用来**恢复我们的`Redis`数据库的`RDB`格式的数据**

### `Redis-server`关闭的情况下设置

#### `aof-use-rdb-preamble no`情况下

> 当我们的`Redis`服务器**下次被重新开启时**,会去**读取我们的`RDB`持久化文件**以**恢复我们的`Redis`数据库**,并且在**数据库恢复之后**我们的`Redis`会**根据我们的数据库当前的状态生成一个`appendonly.aof.1.base.aof`文件**,这个文件中就**存储**着可以用来**恢复我们的`Redis`数据库的`AOF`格式的数据**

#### `aof-use-rdb-preamble yes`情况下

> 当我们的`Redis`服务器**下次被重新开启时**,会去**读取我们的`RDB`持久化文件**以**恢复我们的`Redis`数据库**,并且在**数据库恢复之后**我们的`Redis`会**根据我们的数据库当前的状态生成一个`appendonly.aof.1.base.rdb`文件**,这个文件中就**存储**着可以用来**恢复我们的`Redis`数据库的`RDB`格式的数据**

## :three:如果我们本来用的是`RDB`持久化机制,但是突然我们想要关闭``Redis``开启`AOF`使得`RDB`与`AOF`共同作用,那么我们的`AOF`持久化机制如何保证原本存储在`RDB`中的数据被正确恢复呢?

> 当我们的`Redis`服务器重新开启时,会将我们的`RDB`持久化文件中的数据拷贝到`appendonly.aof.1.base.rdb`文件中,然后我们的`Redis`就会根据这个`RDB`持久化文件恢复我们的`Redis`数据库

## :four:假如在`EverySec`策略下当前次我们的`AOF缓冲区`数据触发持久化写入时,我们的上一次`AOF缓冲区`持久化操作还没有完成会导致怎样的效果?
