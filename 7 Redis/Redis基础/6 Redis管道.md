# `Redis`管道的诞生背景

## `Redis-server`与`Redis-client`的通信

> ​		 对于一条命令而言我们的`Redis`工作流程为,`Redis-client`向我们的`Redis-server`发送携带着我们需要调用的命令的数据包,然后经过网络传输由我们的`Redis-server`接收到数据包并解析获取到客户端发起的命令,然后`Redis-server`找到一个合适的时机将`Redis-client`客户端发起的命令执行,并将执行的结果包装起来再又经过网络传输交付给我们的客户端,然后我们的客户端对`Redis-server`服务端发送过来的数据进行解析从而获得命令的执行结果
>
> ​		**上述的过程中的耗时主要在以下几个部分**
>
> - :one:`Redis-client`打包命令数据
> - :two:`Redis-client`到`Redis-server`之间的网络通信
> - :three:`Redis-server`解析命令数据
> - :four:`Redis-server`执行命令
> - :five:`Redis-server`打包命令的执行结果
> - :six:`Redis-server`到`Redis-client`之间的网络通信
> - :seven:`Redis-client`解析结果数据

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303090312570.png" alt="image-20230309031208472" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303090335286.png" alt="image-20230309033555233" style="zoom:80%;" />

## 带来的问题

> ​		上面我们讲述的是`Redis`下的一条命令执行的全过程,如果我们**每一条语句都单独执行**,也就是说每一条语句都要经过上述的步骤,那么我们**在网络通信上肯定会耗费极其多的时间**.如果我们能够在客户端**打包好一系列的命令**,然后**一次性发送给我们的服务端**,并由**服务端一并执行与返回**,那么不谈太过细节的事情,至少来说我们**可以节约大量的网络通信的耗时**.而**`Redis`管道正是为了实现这一目的而设计出来的**.

# `Redis`管道与`Redis`事务的区别

# `Redis`管道与`MySQL`事务的区别

![image-20230309034831990](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303090348045.png)

# `Redis`管道与`Redis`原生的`MSET`,`MGET`这类命令的区别

![image-20230309034855319](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303090348395.png)

![image-20230309035057816](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303090350865.png)

# `Redis`管道的使用

## 基本使用步骤

- :one:`sudo vim pipeTest`新建一个存储文本的文件并编辑
- :two:在这个文件中像在`Redis`客户端中一样写下一条或多条命令调用,然后存储该文件
    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303090342823.png" alt="image-20230309034211778" style="zoom: 50%;" />
- :three:`redis-server /opt/MyRedis/redis7.conf`保证`Redis`服务端已经开启
- :four:`cat 文件名 | redis-cli -a 58828738 –pipe`
    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303090342388.png" alt="image-20230309034231338" style="zoom: 50%;" />

# `Redis`管道使用的一些问题

## :one:同一个管道中的命令是否会被`Redis-server`依次连续地执行,会不会在某两条命令之间去执行一下其他的客户端发起的命令调用?
