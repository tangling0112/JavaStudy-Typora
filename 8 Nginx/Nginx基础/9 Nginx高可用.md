# 高可用的诞生背景

> ​		我们前面提到，我们借助于`Nginx`服务器能够实现反向代理，负载均衡等一系列功能。我们的外网客户端都先访问我们的`Nginx`服务器，然后再由我们的`Nginx`服务器将我们的`Request`请求转发给我们的内网中的应用服务器进行处理。
>
> ​		此时我们就会遇到一个问题，假如我们的`Nginx`服务器因为某些原因发生了故障，从而导致无法实现请求的接收，请求的转发。那么此时我们的网络应用服务显然无法正常工作。必须等到我们的该`Nginx`服务器正确地恢复过来。
>
> ​		高可用正是面向这一问题诞生的。我们希望可以有多台`Nginx`服务器，通常状态下只有其中一台负责请求转发与请求接收，当我们的这台`Nginx`服务器发生故障的时候，迅速地将另一台`Nginx`服务器投入使用，使得新的请求由我们的这台新的`Nginx`服务器所接收，转发。通过这种方式我们就保证了即便我们的工作`Nginx`服务器发生了故障，借助于另一台`Nginx`服务器的迅速投入使用，可以最大程度上做到对我们的网络应用服务最小的影响。而这就被称为我们的高可用。
>
> ​		`keepalived`就是我们的一个优秀地用于实现`Nginx`服务器高可用的组件。

# `Nginx`高可用配置

## `Nginx`高可用的基本实现方法

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303021215109.png" alt="image-20230302121502986" style="zoom: 67%;" />

## `keepalived`的安装配置

- `sudo apt-get install keepalived`

### `keepalived`相关文件所在位置

- **`.conf`配置文件**
    - `/etc/keepalived/`
- ``

## `keepalived`配置文件详解

```shell
! Configuration File for keepalived 

#vrrp 实例部分定义，tangling自定义名称
vrrp_instance tangling {     

	#指定 keepalived 的角色，必须大写 可选值：MASTER|BACKUP
    state MASTER               
    
    #网卡设置，lvs需要绑定在网卡上，realserver绑定在回环口。区别：lvs对访问为外，realserver为内不易暴露本机信息
    interface ens33     
    
    #虚拟路由标识，是一个数字，同一个vrrp 实例使用唯一的标识，MASTER和BACKUP 的 同一个 vrrp_instance 下 这个标识必须保持一致
    virtual_router_id 51   
    
    #定义优先级，数字越大，优先级越高。
    priority 100                                  
    
    #设定 MASTER 与 BACKUP 负载均衡之间同步检查的时间间隔，单位为秒，两个节点设置必须一样
    advert_int 1                                
    
    #设置验证类型和密码，两个节点必须一致
    authentication {                              
        auth_type PASS                        
        auth_pass 1111                        
    }         
    
    #设置虚拟IP地址，可以设置多个虚拟IP地址，每行一个
    virtual_ipaddress {                           
        192.168.184.200                       
    }
    
    #脚本监控状态
    track_script {                                
    	#可加权重，但会覆盖声明的脚本权重值。chk_nginx_service weight -20
        chk_nginx_service                         
    }
    
        notify_master "脚本路径"  #当前节点成为master时，通知脚本执行任务
        notify_backup "脚本路径"   #当前节点成为backup时，通知脚本执行任务
        notify_fault  "脚本路径"   #当当前节点出现故障，执行的任务; 
}      

#定义RealServer对应的Virtual IP及服务端口，IP和端口之间用空格隔开
virtual_server 192.168.184.200 80  {          
    delay_loop 6                              #每隔6秒查询realserver状态
    lb_algo rr                                #后端调试算法（load balancing algorithm）
    lb_kind DR                                #LVS调度类型NAT/DR/TUN
    #persistence_timeout 60                   同一IP的连接60秒内被分配到同一台realserver
    protocol TCP                              #用TCP协议检查realserver状态
    
    real_server 192.168.184.128 80 {          
        weight 1                              #权重，最大越高，lvs就越优先访问
        
        #keepalived的健康检查方式HTTP_GET | SSL_GET | TCP_CHECK | SMTP_CHECK | MISC
        TCP_CHECK {                           
            connect_timeout 10                #10秒无响应超时
            retry 3                           #重连次数3次
            delay_before_retry 3              #重连间隔时间
            connect_port 80                   #健康检查realserver的端口
        }                                     
    }              
    
    real_server 192.168.184.129 80 {          
        weight 1                              #权重，最大越高，lvs就越优先访问
        
        #keepalived的健康检查方式HTTP_GET | SSL_GET | TCP_CHECK | SMTP_CHECK | MISC
        TCP_CHECK {                           
            connect_timeout 10                #10秒无响应超时
            retry 3                           #重连次数3次
            delay_before_retry 3              #重连间隔时间
            connect_port 80                   #健康检查realserver的端口
        }                                     
    }                                         
}                                             
 
vrrp_instance VI_2 {                          #vrrp 实例部分定义，VI_1自定义名称
    
    #指定 keepalived 的角色，必须大写 可选值：MASTER|BACKUP 分别表示（主|备）
    state   BACKUP                            
    
    #网卡设置，绑定vip的子接口，lvs需要绑定在网卡上，realserver绑定在回环口。区别：lvs对访问为外，realserver为内不易暴露本机信息
    interface ens33                           
    
    #虚拟路由标识，是一个数字，同一个vrrp 实例使用唯一的标识，MASTER和BACKUP 的 同一个 vrrp_instance 下 这个标识必须保持一致
    virtual_router_id 52       
    
    #定义优先级，数字越大，优先级越高。
    priority 90                               
    
    #设定 MASTER 与 BACKUP 负载均衡之间同步检查的时间间隔，单位为秒，两个节点设置必须一样
    advert_int 1     
    
    #设置验证类型和密码，两个节点必须一致
    authentication {                          
        auth_type PASS                        
        auth_pass 1111                        
    }
    
    #设置虚拟IP地址，可以设置多个虚拟IP地址，每行一个
    virtual_ipaddress {                       
        192.168.184.201                       
    }                                         
}                                             
 
#定义RealServer对应的VIP及服务端口，IP和端口之间用空格隔开
virtual_server 192.168.184.201 80 {           
    delay_loop 6                              #每隔6秒查询realserver状态
    lb_algo rr                                #后端调试算法（load balancing algorithm）
    lb_kind DR                                #LVS调度类型NAT/DR/TUN
    #persistence_timeout 60                   #同一IP的连接60秒内被分配到同一台realserver
    protocol TCP                              #用TCP协议检查realserver状态
    real_server 192.168.184.128 80 {          
        weight 1                              #权重，最大越高，lvs就越优先访问
        
        #keepalived的健康检查方式HTTP_GET | SSL_GET | TCP_CHECK | SMTP_CHECK | MISC
        TCP_CHECK {                           
            connect_timeout 10                #10秒无响应超时
            retry 3                           #重连次数3次
            delay_before_retry 3              #重连间隔时间
            connect_port 80                   #健康检查realserver的端口
        }                                     
    }                                         
    real_server 192.168.184.129 80 {          
        weight 1                              #权重，最大越高，lvs就越优先访问
        
        #keepalived的健康检查方式HTTP_GET | SSL_GET | TCP_CHECK | SMTP_CHECK | MISC
        TCP_CHECK {                           
            connect_timeout 10                #10秒无响应超时
            retry 3                           #重连次数3次
            delay_before_retry 3              #重连间隔时间
            connect_port 80                   #健康检查realserver的端口
        }
    }
```

## `keepalived`配置示例

### 主机

```shell
! Configuration File for keepalived 
vrrp_instance tangling1 {     
    state MASTER               
    interface ens33     
    virtual_router_id 51   
    priority 50                                  
    advert_int 1                                       
    authentication {                             
        auth_type PASS                        
        auth_pass 1111                        
    }
    virtual_ipaddress {                           
        192.168.184.200                       
    }
}      
```

### 备用机

```shell
! Configuration File for keepalived 
vrrp_instance tangling1 {     
    state BACKUP               
    interface ens33     
    virtual_router_id 51   
    priority 50                                  
    advert_int 1                                       
    authentication {                             
        auth_type PASS                        
        auth_pass 1111                        
    }
    virtual_ipaddress {                           
        192.168.184.200                       
    }
}      
```

## `keepalive`完整`.conf`配置文件

```shell
! Configuration File for keepalived
global_defs {                                     #全局定义部分
    notification_email {                          #设置报警邮件地址，可设置多个
        acassen@firewall.loc                      #接收通知的邮件地址
    }                        
    notification_email_from test0@163.com         #设置 发送邮件通知的地址
    smtp_server smtp.163.com                      #设置 smtp server 地址，可是ip或域名.可选端口号 （默认25）
    smtp_connect_timeout 30                       #设置 连接 smtp server的超时时间
    router_id LVS_DEVEL                           #主机标识，用于邮件通知
    vrrp_skip_check_adv_addr                   
    vrrp_strict                                   #严格执行VRRP协议规范，此模式不支持节点单播
    vrrp_garp_interval 0                       
    vrrp_gna_interval 0     
    script_user keepalived_script                 #指定运行脚本的用户名和组。默认使用用户的默认组。如未指定，默认为keepalived_script 用户，如无此用户，则使用root
    enable_script_security                        #如过路径为非root可写，不要配置脚本为root用户执行。
}       
 
vrrp_script chk_nginx_service {                   #VRRP 脚本声明
    script "/etc/keepalived/chk_nginx.sh"         #周期性执行的脚本
    interval 3                                    #运行脚本的间隔时间，秒
    weight -20                                    #权重，priority值减去此值要小于备服务的priority值
    fall 3                                        #检测几次失败才为失败，整数
    rise 2                                        #检测几次状态为正常的，才确认正常，整数
    user keepalived_script                        #执行脚本的用户或组
}                                             
 
vrrp_instance VI_1 {                              #vrrp 实例部分定义，VI_1自定义名称
    state MASTER                                  #指定 keepalived 的角色，必须大写 可选值：MASTER|BACKUP
    interface ens33                               #网卡设置，lvs需要绑定在网卡上，realserver绑定在回环口。区别：lvs对访问为外，realserver为内不易暴露本机信息
    virtual_router_id 51                          #虚拟路由标识，是一个数字，同一个vrrp 实例使用唯一的标识，MASTER和BACKUP 的 同一个 vrrp_instance 下 这个标识必须保持一致
    priority 100                                  #定义优先级，数字越大，优先级越高。
    advert_int 1                                  #设定 MASTER 与 BACKUP 负载均衡之间同步检查的时间间隔，单位为秒，两个节点设置必须一样
    authentication {                              #设置验证类型和密码，两个节点必须一致
        auth_type PASS                        
        auth_pass 1111                        
    }                                         
    virtual_ipaddress {                           #设置虚拟IP地址，可以设置多个虚拟IP地址，每行一个
        192.168.119.130                       
    }
    track_script {                                #脚本监控状态
        chk_nginx_service                         #可加权重，但会覆盖声明的脚本权重值。chk_nginx_service weight -20
    }
        notify_master "/etc/keepalived/start_haproxy.sh start"  #当前节点成为master时，通知脚本执行任务
        notify_backup "/etc/keepalived/start_haproxy.sh stop"   #当前节点成为backup时，通知脚本执行任务
        notify_fault  "/etc/keepalived/start_haproxy.sh stop"   #当当前节点出现故障，执行的任务; 
}                                             
 
virtual_server 192.168.119.130 80  {          #定义RealServer对应的VIP及服务端口，IP和端口之间用空格隔开
    delay_loop 6                              #每隔6秒查询realserver状态
    lb_algo rr                                #后端调试算法（load balancing algorithm）
    lb_kind DR                                #LVS调度类型NAT/DR/TUN
    #persistence_timeout 60                   同一IP的连接60秒内被分配到同一台realserver
    protocol TCP                              #用TCP协议检查realserver状态
    real_server 192.168.119.120 80 {          
        weight 1                              #权重，最大越高，lvs就越优先访问
        TCP_CHECK {                           #keepalived的健康检查方式HTTP_GET | SSL_GET | TCP_CHECK | SMTP_CHECK | MISC
            connect_timeout 10                #10秒无响应超时
            retry 3                           #重连次数3次
            delay_before_retry 3              #重连间隔时间
            connect_port 80                   #健康检查realserver的端口
        }                                     
    }                                         
    real_server 192.168.119.121 80 {          
        weight 1                              #权重，最大越高，lvs就越优先访问
        TCP_CHECK {                           #keepalived的健康检查方式HTTP_GET | SSL_GET | TCP_CHECK | SMTP_CHECK | MISC
            connect_timeout 10                #10秒无响应超时
            retry 3                           #重连次数3次
            delay_before_retry 3              #重连间隔时间
            connect_port 80                   #健康检查realserver的端口
        }                                     
    }                                         
}                                             
 
vrrp_instance VI_2 {                          #vrrp 实例部分定义，VI_1自定义名称
    state   BACKUP                            #指定 keepalived 的角色，必须大写 可选值：MASTER|BACKUP 分别表示（主|备）
    interface ens33                           #网卡设置，绑定vip的子接口，lvs需要绑定在网卡上，realserver绑定在回环口。区别：lvs对访问为外，realserver为内不易暴露本机信息
    virtual_router_id 52                      #虚拟路由标识，是一个数字，同一个vrrp 实例使用唯一的标识，MASTER和BACKUP 的 同一个 vrrp_instance 下 这个标识必须保持一致
    priority 90                               #定义优先级，数字越大，优先级越高。
    advert_int 1                              #设定 MASTER 与 BACKUP 负载均衡之间同步检查的时间间隔，单位为秒，两个节点设置必须一样
    authentication {                          #设置验证类型和密码，两个节点必须一致
        auth_type PASS                        
        auth_pass 1111                        
    }                                         
    virtual_ipaddress {                       #设置虚拟IP地址，可以设置多个虚拟IP地址，每行一个
        192.168.119.131                       
    }                                         
}                                             
 
virtual_server 192.168.119.131 80 {           #定义RealServer对应的VIP及服务端口，IP和端口之间用空格隔开
    delay_loop 6                              #每隔6秒查询realserver状态
    lb_algo rr                                #后端调试算法（load balancing algorithm）
    lb_kind DR                                #LVS调度类型NAT/DR/TUN
    #persistence_timeout 60                   #同一IP的连接60秒内被分配到同一台realserver
    protocol TCP                              #用TCP协议检查realserver状态
    real_server 192.168.119.120 80 {          
        weight 1                              #权重，最大越高，lvs就越优先访问
        TCP_CHECK {                           #keepalived的健康检查方式HTTP_GET | SSL_GET | TCP_CHECK | SMTP_CHECK | MISC
            connect_timeout 10                #10秒无响应超时
            retry 3                           #重连次数3次
            delay_before_retry 3              #重连间隔时间
            connect_port 80                   #健康检查realserver的端口
        }                                     
    }                                         
    real_server 192.168.119.121 80 {          
        weight 1                              #权重，最大越高，lvs就越优先访问
        TCP_CHECK {                           #keepalived的健康检查方式HTTP_GET | SSL_GET | TCP_CHECK | SMTP_CHECK | MISC
            connect_timeout 10                #10秒无响应超时
            retry 3                           #重连次数3次
            delay_before_retry 3              #重连间隔时间
            connect_port 80                   #健康检查realserver的端口
        }
    }
```

