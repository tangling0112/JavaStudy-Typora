

# 1 `location`属性的使用示例

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302282034922.png" alt="image-20230228203446883" style="zoom: 90%;" />

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302282035014.png" alt="image-20230228203522962" style="zoom: 67%;" />

# 2 `location`属性各个设置的含义

## 2.1 **映射路径配置**

### **示例**

- :one:`location =/ {}`
- :two:`location =/css {}`
- :three:`location =/cs {}`

### **作用**

- 这种方式设置了我们的当前`location`所能处理的`Request`请求
    - 第一种设置就只能处理`http://Host/?请求参数`或`http://Host/静态文件?请求参数`这样两种`Request`请求
    - 第二种设置就只能处理`http://Host/css?请求参数`或`http://Host/css/静态文件?请求参数`这两种`Request`请求
    - 第三种就只能处理只能处理`http://Host/cs?请求参数`或`http://Host/cs/静态文件?请求参数`这两种`Request`请求

## 2.2 `root`

### **示例**

- :one:`root html`
    - 相对路径(**没有以`/`开始**被判定为相对路径)
- :two:`root /home/tangling/html`
    - 绝对路径(**以`/`开始**被判定为绝对路径)

### **作用**

- 这种方式设置了我们的`Request`将要访问的资源所在的目录
    - 假设我们的`location`的映射路径为`/css`,用户的请求为`/css/css.html`
        - 对于**第一种**而言我们的`Request`将要访问的静态资源为**`nginx根目录/html/css/css.html`**
        - 对于**第二种**而言我们的`Request`将要访问的静态资源为**`/home/tangling/html/css/css.html`**

## 2.3 `index`

### **示例**

- :one:`index index.html`
- :two:`index css.html`
- :three:`index index.html css.html`(先**看看`index.html`有没有**,有就返回`index.html`,**没有就看`css.html`有没有**,有就返回`css.html`,**没有就返回`404`**)

### **作用**

- 这种方式设置了当我们的`Request`请求**没有指定要访问哪个静态资源**时我们的`nginx`返回给我们的客户端的静态资源
    - 假设我们的`location`的**映射路径为`/css`**,用户的**请求为`/css`**,**`root`被设置为`html`**
        - :one:对于**第一种**而言,如果`nginx根目录/html/css/index.html`**存在**就将这个静态文件返回给我们的客户端,如果**不存在**就返回`404`
        - :two:对于**第二种**而言,如果`nginx根目录/html/css/css.html`**存在**就将这个静态文件返回给我们的客户端,如果**不存在**就返回`404`
        - :three:对于**第三种**而言,如果`nginx根目录/html/css/index.html`**存在**就将这个静态文件返回给我们的客户端,如果**不存在**就看看`nginx根目录/html/css/css.html`**存不存在**,**存在**就将这个静态文件返回给我们的客户端,如果**不存在**就返回`404`

## 2.4 `proxy_pass`

### **示例**

- :one:`proxy_pass www.baidu.com`
- :two: `proxy_pass 192.168.184.129`

### **作用**

- 这种方式用于反向代理的实现,**具体作用见反向代理**

# 3 `location`属性的匹配原则

> - ``
> - `=`
>     - 精确匹配
> - `~`
>     - 正则匹配
> - `无`
>     - 最长前缀匹配
> - `^~`
> - `~*`

> - `location`的匹配遵循**精确的完全匹配原则**
>     - 精确匹配规则即如果客户端发起的请求
>         - **有一个完全对应**的`location`与之匹配,那么就访问这个完全对应的`location`
>         - 如果**找不到完全对应**的`location`,那么我们的`nginx`就认为没有`location`能够处理这个请求.此时通常情况下我们的客户端就会接到`404`的`Response`响应
>     - **完全对应是指**:假设用户的请求为`/css`那么这个请求**只匹配`/css`**的`location`,其他的`/,/cs,/c`等等`location`都不能匹配

# 4 基于正则表达式的`location`映射路径配置
