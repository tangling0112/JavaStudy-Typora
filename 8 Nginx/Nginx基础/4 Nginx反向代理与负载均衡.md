# 反向代理与负载均衡概念的产生背景

> - 我们知道，我们使用`SprignBoot`来开发`Web`项目,这个`Web`项目可以运行在我们的某个主机上,这个主机就是我们的一个服务端,然后我们的客户端可以向我们的服务端的这个主机发起`Request`请求.**这种情况下只有一台服务端主机**,当我们的客户端**访问量较小**时,不会带来什么问题.但是当我们的**客户端访问量巨大**时,一台服务端主机就会**明显难以实时地为我们的客户端提供服务**.
> - 这个时候,我们就会想,**可不可以去多加几台服务端主机**,让他们**一起来处理**我们的客户端`Request`请求呢?答案当然是**可以的**.但是此时我们又**面临着两个问题**
>     - :one:常规情况下,我们的这些主机都**必须有一个自己的公网`IP`**,然后我们的客户端通过不同的公网`IP`去访问不同的服务端主机获取服务.
>     - :two:通常情况下,我们希望的是,客户端只需要通过一个域名就可以直接访问我们的服务端主机.如`www.baidu.com`,但是呢很显然,我们的**一个域名只能绑定到一个公网`IP`上**
> - **反向代理**——那么这些问题有没有什么办法可以解决呢?当然也有方法可以解决,那就是**构建一个特殊的有着公网`IP`的服务端主机**,然后让**这个服务端主机接收所有的`Request`请求**,在接收到请求后将我们的`Request`**请求转发到我们的内网上的某个提供服务的服务器主机**,让这个服务器主机来实际处理业务.但是这样呢又带来了一个问题
>     - :one:我们内网有**多台服务器主机,都可以提供服务**,那么我们如何**保证**每一台服务器主机在一定的时间段内**处理的`Request`请求数都均衡**呢.
> - **负载均衡**——那么这个问题有没有什么办法解决呢?当然有,那就是我们`Nginx`的**负载均衡机制**
>
> ![image-20230228135638865](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281356937.png)

# 反向代理

## 什么是反向代理?

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281011826.png" alt="image-20230228101122721" style="zoom: 55%;" />

## 正向代理是什么?

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281012525.png" alt="image-20230228101257453" style="zoom:67%;" />

## `Nginx`反向代理在企业中的应用

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281023480.png" alt="image-20230228102301414" style="zoom: 61%;" />

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281022417.png" alt="image-20230228102247323" style="zoom: 52%;" />

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281023466.png" alt="image-20230228102335383" style="zoom:67%;" />

## 基于`nginx.conf`配置文件的`Nginx`反向代理的配置

![image-20230228103951556](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281039621.png)

![image-20230228131032884](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281310950.png)

## `Nginx`反向代理配置的注意事项

> ​		如果我们的**`proxy_pass`属性**指定的**不是**某个我们的`Nginx`服务器**组网内的服务器的内网`IP`**,而是**一个公网`IP`**或者**域名**,那么我们的`Nginx`就不会反向代理来**将我们的请求转发给我们的内网的某个服务器**,而是**直接告知**我们的**客户端启动重定向**,重新向到我们使用`proxy_pass`指定的公网`IP`或者域名

- :one:我们设置**`proxy_pass=http://192.168.184.129`**,那么我们最终的结果如下,**我们的请求是被转发了,而不是被重定向了**
    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281342363.png" alt="image-20230228134159982" style="zoom: 56%;" />
- :two:我们设置**`proxy_pass=http://www.baidu.com`**,那么最终的结果如下，**我们的请求被直接重定向了，而非转发**
    - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281345704.png" alt="image-20230228134527439" style="zoom:56%;" />

# 负载均衡

## 负载均衡的作用

> ​		**借助于负载均衡,可以使得我们的服务端的大量主机处理的客户端`Request`请求的工作量维持在一个合适的水平.**
>
> - 也就是说假如大家的工作量平均数为`40`
>     - 那么**在不负载均衡的情况下**,工作量最大的服务端主机的工作为`100`,而工作量最小的主机的工作为`10`
>     - 在**负载均衡的情况下**,工作了最大的服务端主机的工作为`50`,工作量最小的主机的工作为`30`

## 基于`nignx.conf`配置文件的`Nginx`负载均衡的配置

> **常见的负载均衡策略**
>
> - :one:轮询式负载均衡
> - :two:基于权重的负载均衡
>
> **不常用的负载均衡策略**
>
> - :one:基于`IP_Hash`的负载均衡
>     - 根据客户端`IP`地址来进行请求转发.
>     - 客户端第一次发送请求过来按照常规负载均衡策略找到处理它的服务端主机,然后服务端会将这个客户端的`IP`地址以及其被哪个服务端主机处理这两大信息保存下来.然后当同一客户端再次发送请求时,就会依旧由上一次处理它的服务端主机进行处理
> - :two:基于`Least_conn`的负载均衡
>     - 最少连接访问
> - :three:基于`URL_Hash`的负载均衡
>     - 根据客户端访问的`URL`进行请求转发.
>     - 我们在服务端给我们的各个服务端主机每一个指定一些需要它来进行处理的`URL`,当客户端发起`Request`请求时,我们的`Nginx`服务器就会查看这个`Request`请求的`URL`然后找到专门处理这个`URL`的服务端主机,将请求转发给这个服务端主机
> - :four:基于`Fair`的负载均衡
>     - 根据后端服务器的响应时间来进行请求转发

### 轮询式负载均衡

![image-20230228151007128](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281510361.png)

### 基于权重的负载均衡(`weight`)

![image-20230228151323861](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281513315.png)

### `down`,`backup`属性的使用

#### `down`

![image-20230228151459897](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281514968.png)

#### `backup`

![image-20230228151540336](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281515390.png)

# 负载均衡反向代理给我们的常规`Web`应用开发带来的问题及其解决方案

## 带来的问题

![image-20230228153817126](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281538468.png)

## 解决方案

### 使用`Redis`等来存储我们的`Session`

### 使用`Token`令牌机制
