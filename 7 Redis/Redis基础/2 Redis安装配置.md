# `Redis7`目录结构

![image-20230303134629088](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303031346123.png)

![image-20230303134008123](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303031340250.png)

# `Redis7`安装配置

- `mkdir /opt/MyRedis7`

- `cp /opt/redis-7.0.9/redis.conf /opt/MyRedis7/redis7.conf`

- ![image-20230303144600240](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303031446287.png)

- ```properties
    #bind 127.0.0.1 -::1
    requirepass 58828738
    protected-mode no
    port 6379
    tcp-backlog 511
    timeout 0
    tcp-keepalive 300
    daemonize yes
    pidfile /var/run/redis_6379.pid
    loglevel notice
    logfile ""
    databases 16
    always-show-logo no
    set-proc-title yes
    proc-title-template "{title} {listen-addr} {server-mode}"
    stop-writes-on-bgsave-error yes
    rdbcompression yes
    rdbchecksum yes
    dbfilename dump.rdb
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
    appendonly no
    appendfilename "appendonly.aof"
    appenddirname "appendonlydir"
    appendfsync everysec
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

- `redis-server /opt/MyRedis/redis7.conf`

- `redis-cli -a 58828738 -p 6379`

- ![image-20230303145542424](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303031455466.png)

- `PING`

- ![image-20230303145635330](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303031456361.png)

- `SHUTDOWN`

    - ![image-20230303150617860](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303031506895.png)

- `redis-cli -a 58828738 -p 6379 shutdown`

    - ![image-20230303150453866](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303031504902.png)

- 

# `Redis7`默认配置文件介绍

```properties
bind=127.0.0.1 -::1
protected-mode=yes
port=6379
tcp-backlog=511
timeout=0
tcp-keepalive=300
daemonize=no
pidfile=/var/run/redis_6379.pid
loglevel=notice
logfile=""
databases=16
always-show-logo=no
set-proc-title=yes
proc-title-template="{title} {listen-addr} {server-mode}"
stop-writes-on-bgsave-error=yes
rdbcompression=yes
rdbchecksum=yes
dbfilename=dump.rdb
rdb-del-sync-files=no
dir=./
replica-serve-stale-data=yes
replica-read-only=yes
repl-diskless-sync=yes
repl-diskless-sync-delay=5
repl-diskless-sync-max-replicas=0
repl-diskless-load=disabled
repl-disable-tcp-nodelay=no
replica-priority=100
acllog-max-len=128
lazyfree-lazy-eviction=no
lazyfree-lazy-expire=no
lazyfree-lazy-server-del=no
replica-lazy-flush=no
lazyfree-lazy-user-del=no
lazyfree-lazy-user-flush=no
oom-score-adj=no
oom-score-adj-values=0 200 800
disable-thp=yes
appendonly=no
appendfilename="appendonly.aof"
appenddirname="appendonlydir"
appendfsync=everysec
no-appendfsync-on-rewrite=no
auto-aof-rewrite-percentage=100
auto-aof-rewrite-min-size=64mb
aof-load-truncated=yes
aof-use-rdb-preamble=yes
aof-timestamp-enabled=no
slowlog-log-slower-than=10000
slowlog-max-len=128
latency-monitor-threshold=0
notify-keyspace-events=""
hash-max-listpack-entries=512
hash-max-listpack-value=64
list-max-listpack-size=-2
list-compress-depth=0
set-max-intset-entries=512
zset-max-listpack-entries=128
zset-max-listpack-value=64
hll-sparse-max-bytes=3000
stream-node-max-bytes=4096
stream-node-max-entries=100
activerehashing=yes
client-output-buffer-limit=normal 0 0 0
client-output-buffer-limit=replica 256mb 64mb 60
client-output-buffer-limit=pubsub 32mb 8mb 60
hz=10
dynamic-hz=yes
aof-rewrite-incremental-fsync=yes
rdb-save-incremental-fsync=yes
jemalloc-bg-thread=yes
```

