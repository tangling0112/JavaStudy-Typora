# `Nginx`的`nginx.conf`配置文件

```shell
#我们前面提到了我们的Nginx的主进程会开启多个worker子进程去处理我们来自于外部的访问，而我们的这个worker_processes属性就是用于指定我们的主进程开启的worker子进程的数量。默认为1,即只开启一个worker子进程
worker_processes  1;	

events {
	#用于指定我们的一个worker子进程同时维持的最大连接数
    worker_connections  1024; 
}

http {

	#借助include指令将我们的nginx.conf所在目录下的mime.types文件中的配置内容导入到了http{}中
	#这个的作用在于当我们的服务端向我们的客户端发送数据时,显然我们的服务端并没有像客户端向服务端发送数据时可以使用请求参数等等方式来告知我们的服务端传输的数据的后缀名,我们的服务器端只能借助于Response响应的Content-Type响应头来告诉我们的客户端我们给它发送的数据是什么后缀名的.
	#这个mime.types就给出了不同的文件后缀名对应的Content-Type的值
	#当我们的服务端要发送文件给我们的客户端时,就会根据文件的后缀名自动地查找mime.type获取到Content-Type请求头的值.从而实现告知客户端传输的数据的后缀名的目的.
    include       mime.types;
    
    #指定当我们的服务端要向我们的客户端传输的数据类型在mime.types中没有匹配的Content-Type时,我们的服务端给Response响应中的Content-Type请求头设置的值
    default_type  application/octet-stream;
	
	#在下面有详细介绍
    sendfile        on;
    
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}

```

## `sendfile`设置为`on/off`的效果

> - 设置为`off`
> - 设置为`on`

![image-20230227204143170](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302272041328.png)

![image-20230227204124370](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302272041684.png)

## `mime.type`文件

```yaml
types {
    text/html                                        html htm shtml;
    text/css                                         css;
    text/xml                                         xml;
    image/gif                                        gif;
    image/jpeg                                       jpeg jpg;
    application/javascript                           js;
    application/atom+xml                             atom;
    application/rss+xml                              rss;

    text/mathml                                      mml;
    text/plain                                       txt;
    text/vnd.sun.j2me.app-descriptor                 jad;
    text/vnd.wap.wml                                 wml;
    text/x-component                                 htc;

    image/avif                                       avif;
    image/png                                        png;
    image/svg+xml                                    svg svgz;
    image/tiff                                       tif tiff;
    image/vnd.wap.wbmp                               wbmp;
    image/webp                                       webp;
    image/x-icon                                     ico;
    image/x-jng                                      jng;
    image/x-ms-bmp                                   bmp;

    font/woff                                        woff;
    font/woff2                                       woff2;

    application/java-archive                         jar war ear;
    application/json                                 json;
    application/mac-binhex40                         hqx;
    application/msword                               doc;
    application/pdf                                  pdf;
    application/postscript                           ps eps ai;
    application/rtf                                  rtf;
    application/vnd.apple.mpegurl                    m3u8;
    application/vnd.google-earth.kml+xml             kml;
    application/vnd.google-earth.kmz                 kmz;
    application/vnd.ms-excel                         xls;
    application/vnd.ms-fontobject                    eot;
    application/vnd.ms-powerpoint                    ppt;
    application/vnd.oasis.opendocument.graphics      odg;
    application/vnd.oasis.opendocument.presentation  odp;
    application/vnd.oasis.opendocument.spreadsheet   ods;
    application/vnd.oasis.opendocument.text          odt;
    application/vnd.openxmlformats-officedocument.presentationml.presentation
                                                     pptx;
    application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
                                                     xlsx;
    application/vnd.openxmlformats-officedocument.wordprocessingml.document
                                                     docx;
    application/vnd.wap.wmlc                         wmlc;
    application/wasm                                 wasm;
    application/x-7z-compressed                      7z;
    application/x-cocoa                              cco;
    application/x-java-archive-diff                  jardiff;
    application/x-java-jnlp-file                     jnlp;
    application/x-makeself                           run;
    application/x-perl                               pl pm;
    application/x-pilot                              prc pdb;
    application/x-rar-compressed                     rar;
    application/x-redhat-package-manager             rpm;
    application/x-sea                                sea;
    application/x-shockwave-flash                    swf;
    application/x-stuffit                            sit;
    application/x-tcl                                tcl tk;
    application/x-x509-ca-cert                       der pem crt;
    application/x-xpinstall                          xpi;
    application/xhtml+xml                            xhtml;
    application/xspf+xml                             xspf;
    application/zip                                  zip;

    application/octet-stream                         bin exe dll;
    application/octet-stream                         deb;
    application/octet-stream                         dmg;
    application/octet-stream                         iso img;
    application/octet-stream                         msi msp msm;

    audio/midi                                       mid midi kar;
    audio/mpeg                                       mp3;
    audio/ogg                                        ogg;
    audio/x-m4a                                      m4a;
    audio/x-realaudio                                ra;

    video/3gpp                                       3gpp 3gp;
    video/mp2t                                       ts;
    video/mp4                                        mp4;
    video/mpeg                                       mpeg mpg;
    video/quicktime                                  mov;
    video/webm                                       webm;
    video/x-flv                                      flv;
    video/x-m4v                                      m4v;
    video/x-mng                                      mng;
    video/x-ms-asf                                   asx asf;
    video/x-ms-wmv                                   wmv;
    video/x-msvideo                                  avi;
}
```

# `Nginx`虚拟主机与域名解析

``
