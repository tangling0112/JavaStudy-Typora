# `UrlRewrite`的诞生背景

> - 通常情况下,当我们的客户端想要访问我们的`Tomcat`服务器上的某个静态文件(如`hello.html`),并传递一个`pageNumber=5`的请求参数时,我们的**客户端会发起`http://192.168.184.128/hello.html?pageNumber=5`的`Request`请求**
> - 这种情况下,我们的客户端发起的请求`URI`我们可能**觉得过于冗杂**了,我们希望让我们的客户端的`Request`请求`URI`**变得更加简洁**.此时我们就需要**用到我们的`UrlRewrite`机制**了
> - 对于`UrlRewrite`而言,我们可以**设置一个客户端请求`URI`与一个特殊的`URI`的映射**
>     - 如将`/1.html`映射为`/hello.html?pageNumber=5`.
>     - 这样我们的客户端只用访问`http://192.168.184.128/1.html`就能够**借助于`Nginx`的代理以及`UrlRewrite`机制**顺利地以`http://192.168.184.128/hello.html?pageNumber=5`这个`URL`**访问我们的`Tomcat`服务器**
>
> **注意**:`UrlRewrite`**只是修改我们的请求`URI`**,像**`referer`请求头这些`request`请求实例的数据是不会修改**的

# `UrlRewrite`配置

## `UrlRewrite`的基本语法格式

```
rewrite 正则表达式 要替换成的内容 标记
```

### 正则表达式

> 用于匹配我们的`URI`,当客户端发起的`Request`请求的`URI`能够与我们的这个正则表达式相匹配,那么就会触发我们当前的这个`Rewrite`

### 要替换的内容

> 用于给出,当我们的客户端发起的`Request`请求的`URI`与我们的当前正则表达式相匹配时,我们要用于替换原来的旧`URI`的新`URI`

### 标记

> 用于指定我们当前的`Rewrite`的遵循的规则,类似于我们设置负载均衡时给`upstream`中的`server`加的`down,backup`等等关键字

## `UrlRewrite`的四大标记

![image-20230301114100067](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/image-20230301114100067.png)

### `last`

### `break`

### `redirect`

### `permanent`

## `UrlRewrite`配置示例

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303011038540.png" alt="image-20230301103854366" style="zoom:67%;" />

# 注意事项

> - 若我们的客户端`Request`请求触发了`UrlRewrite`,那么我们的**这个请求的`request`实例的`URI`就会被修改**,也就是说**这之后**我们服务端**任何人**获取到该`Request`请求的`request`实例后,**获取该`request`实例的`URI`时**获取到的会**是被重写后的`URI`**而**不是重写前的`URI`**
> - 常规情况下,此时**重写前的`URI`只有我们的客户端存储有**,**服务端已经不再存储有**了.
>
> **注意**:我们`UrlRewrite`默认情况下仅仅**只会修改`request`请求实例的`URI`**,**其他的**像`referer`请求头等等一系列的**`request`实例的数据都不会修改**
