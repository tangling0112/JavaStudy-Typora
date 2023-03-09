# 动静分离

> - 
>
> <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281929010.png" alt="image-20230228192907841" style="zoom: 67%;" />

## `Nginx`动静分离的配置

> **`location`属性的匹配原则**
>
> - `location`的匹配遵循**精确匹配原则**,精确匹配规则即如果客户端发起的请求有一个完全对应的`location`与之匹配,那么就访问这个完全对应的`location`,如果找不到完全对应的`location`,那么就根据我们的`location`在配置文件中定义时的顺序依次遍历这些`location`找到第一个能够处理这个请求的`location`(假设客户端发起的请求为`A=www.baidu.com/cs`)
>     - 如果我们的`location`有`/,/img,/js,/c,/cs,/css`,那么最终会由我们的`/cs`的`location`来处理我们的用户请求.
>     - 如果我们的`location`有`/,/img,/js,/c,/css`,那么最终会由我们的`/`的`location`来处理我们的用户请求.

### 基本的动静分离配置

> **注意**
>
> - 

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281934447.png" alt="image-20230228193428396" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281935149.png" alt="image-20230228193545073" style="zoom:67%;" />

### 使用正则表达式指定`location`的动静分离配置

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302281940358.png" alt="image-20230228194017304" style="zoom: 85%;" />
