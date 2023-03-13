# 微服务的基本概念

> ​		微服务是一种架构模式，它提倡将单一的一个应用程序拆分为多个子服务模块，子服务模块之间互相协调，互相配合，为用户提供最终的价值。每一个子服务模块都运行在其独立的进程中，它们之间采用轻量级的通信机制互相协作（如基于`HTTP`协议的`RESTful API`）。每个服务都围绕着具体的业务进行构建，并且能够被独立的部署到生产环境，类生成环境等等。另外，微服务应尽量避免统一的集中式的服务管理机制，对具体的一个服务而言，应根据其业务上下文，选择合适的语言，工具对其进行开发构建

# 微服务的基础组成

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303131827221.png" alt="image-20230313182723090" style="zoom:67%;" />

# `SpringCloud`技术栈

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303131832692.png" alt="image-20230313183238623" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303131833669.png" alt="image-20230313183337585" style="zoom:67%;" />

# `SpringCloud`技术选型

> [``SpringCloud技术选型``](https://start.spring.io/actuator/info)

- 服务开发
    - **`SpringBoot`**
- 服务注册中心
    - **`Nacos`**
    - `Consul`
    - `Zookeeper`
- 负载均衡服务调用
    - `Ribbon`
    - **`LoadBalance`**

- 服务接口调用

    - `Feign`

    - **`OpenFeign`**

- 服务降级
    - **`Sentinel`**
    - `Hystrix`
    - `Resilience4j`
- 服务网关
    - **`Gateway`**
    - `Zuul`/`Zuul2`
- 服务配置
    - **`Nacos`**
    - `Config`

- 服务总线
    - **`Nacos`**
