# 客户端浏览器向服务端发送请求的全过程

- :one:客户端用户通过浏览器向`www.baidu.com`发起`Request`访问请求
- :two:我们的客户端浏览器接到用户要发送`Request`请求后,去访问我们的客户端浏览器所在的操作系统的`hosts`文件,查看是否存储有`www.baidu.com`所对应的`IP`地址
    - **如果有**,那么我们客户端的`Request`请求就会直接根据我们查询到的这个`IP`地址借助于我们的互联网去访问我们的服务端,并且我们要清楚,我们的`www.baidu.com`这个域名,会被自动地存储到我们的`Request`请求当中,以便于我们的服务端知道我们的客户端是通过什么域名进行的访问.
        - 到这一步对于我们的客户端而言,请求的发送过程就已经完成了.即便我们的请求无法成功访问,对于我们的客户端而言也是结束了一次正确的请求
    - **如果没有**,那么进入第:three:步
    - ![image-20230228094245750](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302280942794.png)
        - 当我们随意修改我们的操作系统的`hosts`文件,并将`www.baidu.com`这个域名设置一个无法访问的`IP`,我们再去访问`www.baidu.com`我们就会发现,我们无法访问到百度搜索了
        - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302280945276.png" alt="image-20230228094551226" style="zoom: 50%;" />
- :three:我们的客户端`hosts`文件中无法查找到我的访问的域名对应的`IP`,因此我们的浏览器会直接将我们的请求发送给我们的`DNS`服务器,然后当我们的`DNS`服务器接收到请求后,就会去查看这个请求访问的`www.baidu.com`这个域名在服务器里面是否存储了其对应的`IP`地址.
    - **如果存储了**,那么我们的`DNS`服务器就会再次将我们客户端发出的`Request`请求向这个查询到的`IP`地址转发
    - **如果没有存储**,那么我们的请求就不会被转发,这样我们的客户端浏览器就会一直处于等待响应状态,经过一定的时间,我们的客户端最大等待时长到达之后,就会告诉我们的用户我们的请求访问失败.
    - **注意**:这个`DNS`服务器提供服务的过程很复杂,我们只要知道我们的请求最终被发送到了一个存储着我们的域名对应的`IP`的`DNS`服务器就可以


 

# 浏览器,`Nginx`,`HTTP`协议的关系

![image-20230227205651701](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302272056047.png)

# 域名解析与泛域名解析

# `Nginx`中虚拟主机的配置

> **什么是`Nginx`虚拟主机,他的作用是什么?**
>
> - 通常情况下,在互联网上,我们的客户端做的事情就是借助于`HTTP`协议发起`Request`请求去访问互联网上某个指定的主机,并获取其`Response`响应.
> - 很显然我们的服务端大多**只有一台**(或**只开放了一台)**用于接收客户端请求的主机,但是呢,我们的开发者又希望客户端可以通过不同的**域名+端口号**的组合访问到我们的服务端这**同一台主机**,并且这一台主机还要能够根据请求的**域名+端口号**的组合的不同**做出不同的处理**
> - 为了实现上面的目的,就设计出了虚拟主机,在我们的服务端`Nginx`服务启动后,`Nginx`会自动维护我们借助于`nginx.conf`配置文件设置的虚拟主机,每一个虚拟主机负责对**一个**或**一系列** **域名+端口号**的组合进行处理
>

## `Nginx`虚拟主机的配置

```yaml
#虚拟主机1
server {
    listen       80;
    server_name  www.tangling.com;

    location / {
        root   html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}

#虚拟主机2
server {
    listen       80;
    server_name  www.tangling.org www.tangling.ink;

    location / {
        root   html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
```

## `server_name`属性的匹配规则

- :two:**当我们的请求匹配不上适用于处理它的虚拟主机时,就会直接使用我们第一个配置的虚拟主机来进行出路**
- :three:**当我们的请求匹配到了第一个适用于处理它的虚拟主机,那么后续的虚拟主机就不会再去查看是否匹配了,而是会直接使用我们这个第一个匹配到的虚拟主机来进行处理**
- :four:**同一个`server_name`属性可以设置多个域名**
    - `server_name vod.tangling.com www.tangling.com`
        - `www.tangling.com`与`vod.tangling.com`两者都可以匹配上
- :five:**通配符匹配**
    - `server_name *.tangling.com`
        - `www.tangling.com`,`vod.tangling.com`,`night.tangling.com`都可以匹配上`*.tangling.com`
- :six:**通配符结束匹配**
    - `server_name www.tangling.*`
        - `www.tangling.org`与`www.tangling.com`,`www.tangling.ink`都可以匹配上`www.tangling.*`
- :seven:**正则匹配**
