# Nginx

## 简介

1. 是什么?  
    反向代理服务器, 负载均衡, 动静分离, 健康检查


2. 有什么功能?  
    项目部署在不同的服务器上，但是通过统一的域名进入，nginx则对请求进行分发，减轻服务器的压力

> 1. 高并发。静态小文件
> 2. 占用资源少。2万并发、10个线程，内存消耗几百M。
> 3. 功能种类比较多。web,cache,proxy。每一个功能都不是特别强。
> 4. 支持epoll模型，使得nginx可以支持高并发。
> 5. nginx 配合动态服务和Apache有区别。（FASTCGI 接口）
> 6. 利用nginx可以对IP限速，可以限制连接数。
> 7. 配置简单，更灵活。


3. 应用场合

> 1. 静态服务器。（图片，视频服务）另一个lighttpd。并发几万，html，js，css，flv，jpg，gif等。
> 2. 动态服务，nginx——fastcgi 的方式运行PHP，jsp。（PHP并发在500-1500，MySQL 并发在300-1500）。
> 3. 反向代理，负载均衡。日pv2000W以下，都可以直接用nginx做代理。
> 4. 缓存服务。类似 SQUID,VARNISH。


4. 目录所在?  

```nginx
/usr/sbin/nginx       主程序
/etc/nginx            存放配置文件
/usr/share/nginx      存放静态文件
/var/log/nginx        存放日志
```

## 安装

1. 依赖包

- nginx安装依赖GCC、openssl-devel、pcre-devel和zlib-devel软件库。
- Pcre全称（Perl Compatible Regular Expressions）

```
yum install  pcre pcre-devel -y 
yum install openssl openssl-devel -y 
```

   

2. linux安装Nginx

```nginx
# 1.解压
tar -zxvf nginx-1.18.0.tar.gz

# 2.进入nginx目录
cd nginx-1.18.0 

# 3.配置
./configure --prefix=/data/nginx-1.18 --user=nginx --group=nginx  --with-http_ssl_module  --with-http_stub_status_module

useradd nginx -M -s /sbin/nologin 
# 4.make
make && make install

ln -s /data/nginx-1.10.1 /data/nginx
```

如果出现错误：

```nginx
make: *** 没有规则可以创建“default”需要的目标“build”。 停止。
```

缺少依赖包：

`yum install pcre-devel zlib zlib-devel openssl openssl-devel`

## 启动

#### 正常启动

```nginx
/data/nginx-1.18/sbin/nginx -t	## 检查配置文件
/data/nginx-1.18/sbin/nginx     ## 确定nginx服务

nginx: the configuration file /data/nginx-1.18/conf/nginx.conf syntax is ok
nginx: configuration file /data/nginx-1.18/conf/nginx.conf test is successful

netstat -lntup |grep nginx      ## 检查进程是否正常

shell> curl -I http://localhost	## 确认结果
HTTP/1.1 200 OK
```



#### 其他命令

```nginx
nginx -s signal
signal：
    stop — fast shutdown
    quit — graceful shutdown
    reload — reloading the configuration file
    reopen — reopening the log files # 用来打开日志文件，这样nginx会把新日志信息写入这个新的文件中
```



## 配置

1. 如何配置?

```
    1. 进入hosts文件:
    sudo vim /etc/hosts
    
    2. 内容 ip映射
    127.0.0.1  8080.www.demo.com
    127.0.0.1  8081.www.demo.com
```

5. 配置反向代理(基本)

```nginx
    ＃ 基本反向代理配置
    server {
        listen       80;
        # 请求域名地址 8080.www.demo.com;
        server_name  8080.www.demo.com;
        #charset utf8;
        #access_log  logs/host.access.log  main;
        # 匹配url拦截地址
        location / {
            # 根路径，相对路径
            root   html;
            # 监听8080端口,　转发到http://127.0.0.1:8080
            proxy_pass   http://127.0.0.1:8080;
            # 默认首页
            index  index.html index.htm;
            # 允许cros跨域访问 
            add_header 'Access-Control-Allow-Origin' *;
        }
    }
  
    server {
        listen       80;
        # 请求域名地址 8081.www.demo.com;
        server_name  8081.www.demo.com;
        #charset utf8;
        #access_log  logs/host.access.log  main;
        location / {
            # 根路径，相对路径
            root   html;
            # 监听8081端口
            proxy_pass   http://127.0.0.1:8081;
            # 默认首页
            index  index.html index.htm;
            # 允许cros跨域访问 
            add_header 'Access-Control-Allow-Origin' *;
        }
    }

```

5-1. 配置反向代理(路径匹配)

```nginx
    ＃ 基本反向代理配置
    server {
        listen       80;
        # 请求域名地址 8080.www.demo.com;
        server_name  8080.www.demo.com;
        #charset utf8;
        #access_log  logs/host.access.log  main;
        # 匹配url拦截地址, order开头的
        location ~ /order/ {
            # 根路径，相对路径
            root   html;
            # 监听8080端口,　转发到http://127.0.0.1:8080
            proxy_pass   http://127.0.0.1:8080;
            # 默认首页
            index  index.html index.htm;
            # 允许cros跨域访问 
            add_header 'Access-Control-Allow-Origin' *;
        }
    }
  
    server {
        listen       80;
        # 请求域名地址 8081.www.demo.com;
        server_name  8081.www.demo.com;
        #charset utf8;
        #access_log  logs/host.access.log  main;
        # 匹配url拦截地址, user开头的
        location ~ /user/ {
            # 根路径，相对路径
            root   html;
            # 监听8081端口
            proxy_pass   http://127.0.0.1:8081;
            # 默认首页
            index  index.html index.htm;
            # 允许cros跨域访问 
            add_header 'Access-Control-Allow-Origin' *;
        }
    }

```

6. 配置负载均衡的服务, 在http模块中添加如下配置

```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    # nacos配置
    upstream nacos {
          # ip地址
          server 192.168.107.128:8848  weight=3;
          server 192.168.107.129:8848;
          server 192.168.107.130:8848;
    }
    # weight;   权重
    # ip_hash;　ip绑定
    # fair: 后端响应时间
    # 设置负载均衡-1, weight权重
    upstream tomcatserver1 {
        # ip地址
        server 127.0.0.1:8082 weight=3;
        server 127.0.0.1:8083;
    }
    
    # 设置负载均衡-2　ip_hash
    upstream tomcatserver1 {
        server 127.0.0.1:8082;
        server 127.0.0.1:8083;
        # ip绑定,访问一次, 固定到ip,　可以解决session问题
        ip_hash;
        # fair
    }
	server {
        listen       80;
        server_name  localhost;
        #charset koi8-r;
        #access_log  logs/host.access.log  main;
        location = /nacos {
            root   html;
            index  index.html index.htm;
            # 负载均衡upstream别名
            proxy_pass   http://nacos;
        }
	}
}
```

参数说明:  
1）down
    表示单前的server暂时不参与负载  
2）Weight
    默认为1.weight越大，负载的权重就越大  
3）max_fails
    允许请求失败的次数默认为1.当超过最大次数时，返回proxy_next_upstream 模块定义的错误  
4）fail_timeout
    max_fails 次失败后，暂停的时间  
5）Backup
    其它所有的非backup机器down或者忙的时候，请求backup机器。所以这台机器压力会最轻  

7. 配置反向代理

```nginx
    server {
        listen       80;
        server_name  80.www.demo.com;
        #charset utf8;
        #access_log  logs/host.access.log  main;
        location / {
            # 负载均衡upstream别名
            proxy_pass   http://tomcatserver1;
            index  index.html index.htm;
            # nginx跟后端服务器连接超时时间(nginx连接超时)
            proxy_connect_timeout 10s;
            # nginx发送(真实服务器tomcat)数据响应时间(nginx发送超时)
            proxy_send_timeout 3s;
            # 连接成功后，nginx接收(真实服务器tomcat)数据响应时间(nginx接收超时)
            proxy_read_timeout 3s;
        }
     }
```

8. 配置文件

+ 全局块

```nginx
    #全局设置
    main
    
    # 运行用户
    user www-data;
    
    # 启动进程,通常设置成和cpu的数量相等, 值越大,　可以支持的并发处理数越大
    worker_processes  1;
    
    # 全局错误日志及PID文件
    error_log  /var/log/nginx/error.log;
    pid        /var/run/nginx.pid;
```

+ events块,　与用户的网络连接

```nginx
    # 工作模式及连接数上限
    events {
        #　epoll是多路复用IO(I/O Multiplexing)中的一种方式,但是仅用于linux2.6以上内核,可以大大提高nginx的性能
        use epoll;
        #　单个后台worker process进程的最大并发链接数
        worker_connections 1024; 
        # multi_accept on;
    }
```

+ http块(**重要**)

1. http全局块

```nginx
http {
    # 设定mime类型,类型由mime.type文件定义
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    # 设定日志格式
    access_log    /var/log/nginx/access.log;

    #sendfile 指令指定 nginx 是否调用 sendfile 函数（zero copy 方式）来输出文件，对于普通应用，
    # 必须设为 on,如果用来进行下载等应用磁盘IO重负载应用，可设置为 off，以平衡磁盘与网络I/O处理速度，降低系统的uptime.
    sendfile        on;
    # 将tcp_nopush和tcp_nodelay两个指令设置为on用于防止网络阻塞
    tcp_nopush      on;
    tcp_nodelay     on;
    # 连接超时时间
    keepalive_timeout  65;

    # 开启gzip压缩
    gzip  on;
    gzip_disable "MSIE [1-6]\.(?!.*SV1)";

    # 设定请求缓冲
    client_header_buffer_size    1k;
    large_client_header_buffers  4 4k;

    # 启动配置
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}

```

2. server块, 可以包含多个server块

```nginx
http {   
    #　设定负载均衡的服务器列表
    upstream mysvr {
        # weigth参数表示权值，权值越高被分配到的几率越大
        # 本机上的Squid开启3128端口
        server 192.168.8.1:3128 weight=5;
        server 192.168.8.2:80  weight=1;
        server 192.168.8.3:80  weight=6;
    }


    server {
        #侦听80端口
        listen       80;
        #定义使用www.xx.com访问
        server_name  www.xx.com;

        #设定本虚拟主机的访问日志
        access_log  logs/www.xx.com.access.log  main;

        #默认请求
        location / {
            root   /root;      #定义服务器的默认网站根目录位置
            index index.php index.html index.htm;   #定义首页索引文件的名称

            fastcgi_pass  www.xx.com;
            fastcgi_param  SCRIPT_FILENAME  $document_root/$fastcgi_script_name;
            include /etc/nginx/fastcgi_params;
        }

        # 定义错误提示页面
        error_page   500 502 503 504 /50x.html;
            location = /50x.html {
            root   /root;
        }

        # 静态文件, nginx自己处理
        location ~ ^/(images|javascript|js|css|flash|media|static)/ {
            # 存放静态文件的目录
            root /var/www/virtual/htdocs;
            # 浏览器过期30天，静态文件不怎么更新，过期可以设大一点，如果频繁更新，则可以设置得小一点。
            expires 30d;
        }
        #　PHP 脚本请求全部转发到 FastCGI处理. 使用FastCGI默认配置.
        location ~ \.php$ {
            root /root;
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME /home/www/www$fastcgi_script_name;
            include fastcgi_params;
        }
        # 设定查看Nginx状态的地址
        location /NginxStatus {
            stub_status            on;
            access_log              on;
            auth_basic              "NginxStatus";
            auth_basic_user_file  conf/htpasswd;
        }
        #禁止访问 .htxxx 文件
        location ~ /\.ht {
            deny all;
        }

    }

    #　第一个虚拟服务器
    server {
        #侦听192.168.8.x的80端口
        listen       80;
        server_name  192.168.8.x;

        #　对aspx后缀的进行负载均衡请求
        location ~ .*\.aspx$ {
            root   /root;#定义服务器的默认网站根目录位置
            index index.php index.html index.htm;#定义首页索引文件的名称
            # 请求转向mysvr 定义的服务器列表
            proxy_pass  http://mysvr;

            #以下是一些反向代理的配置可删除.
            proxy_redirect off;

            #后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            client_max_body_size 10m;            #允许客户端请求的最大单文件字节数
            client_body_buffer_size 128k;        #缓冲区代理缓冲用户端请求的最大字节数，
            proxy_connect_timeout 90;            #nginx跟后端服务器连接超时时间(代理连接超时)
            proxy_send_timeout 90;               #后端服务器数据回传时间(代理发送超时)
            proxy_read_timeout 90;               #连接成功后，后端服务器响应时间(代理接收超时)
            proxy_buffer_size 4k;                #设置代理服务器（nginx）保存用户头信息的缓冲区大小
            proxy_buffers 4 32k;                 #proxy_buffers缓冲区，网页平均在32k以下的话，这样设置
            proxy_busy_buffers_size 64k;         #高负荷下缓冲大小（proxy_buffers*2）
            proxy_temp_file_write_size 64k;      #设定缓存文件夹大小，大于这个值，将从upstream服务器传
        }
    }
}

```

8. 动静分离配置

```nginx
http {
    server {
        # 静态文件, nginx自己处理
        location ~ ^/(images|javascript|js|css|flash|media|static)/ {
            # 存放静态文件的目录
            root /var/www/virtual/htdocs;
            # 浏览器过期30天，静态文件不怎么更新，过期可以设大一点，如果频繁更新，则可以设置得小一点。
            expires 30d;
            autoindex  on;
        }     
    } 
}

```

9. 高可用集群

nginx宕机, keepalived, master和backup


```
yum install keepalived.x86_64 -y

​```nginx
yum install keepalived -y
#　配置文件
/etc/keepalived/keepalived.conf

# 修改配置文件
<<<<<<< Updated upstream

# 全局定义
=======
! Configuration File for keepalived

>>>>>>> Stashed changes
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
<<<<<<< Updated upstream
   # 重要, 主机名称
=======
>>>>>>> Stashed changes
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

<<<<<<< Updated upstream
＃ 检测脚本
vrrp_script chk_nginx {
        script "/usr/local/sbin/nginx.sh"               #定义服务检查脚本用于服务异常挂起时尝试启动的操作
        # 检测间隔 2s
        interval 3
        weight -20
}

vrrp_instance VI_1 {
    # 充当角色,　备份服务器上, 为BACKUP
    state MASTER
    # 网卡
    interface eth0
    # 主 备机的virtual_router_id必须相同
    virtual_router_id 51
    # 优先级,　主机值较大
    priority 100
    # 心跳检测
=======
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
>>>>>>> Stashed changes
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
<<<<<<< Updated upstream
    # 虚拟IP地址
=======
>>>>>>> Stashed changes
    virtual_ipaddress {
        192.168.200.16
        192.168.200.17
        192.168.200.18
    }
}

virtual_server 192.168.200.100 443 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.201.100 443 {
        weight 1
        SSL_GET {
            url {
              path /
              digest ff20ad2481f97b1754ef3e12ecd3a9cc
            }
            url {
              path /mrtg/
              digest 9b3a0c85a887a256d6939da88aabd8cd
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}

virtual_server 10.10.10.2 1358 {
    delay_loop 6
    lb_algo rr 
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    sorry_server 192.168.200.200 1358

    real_server 192.168.200.2 1358 {
        weight 1
        HTTP_GET {
            url { 
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl3/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.200.3 1358 {
        weight 1
        HTTP_GET {
            url { 
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334c
            }
            url { 
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334c
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}

virtual_server 10.10.10.3 1358 {
    delay_loop 3
    lb_algo rr 
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.200.4 1358 {
        weight 1
        HTTP_GET {
            url { 
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl3/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.200.5 1358 {
        weight 1
        HTTP_GET {
            url { 
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl3/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
<<<<<<< Updated upstream

```

master和worker进程(nginx热部署),　每个worker是独立进程, worker数量: cpu, worker_connection连接数: 发起请求,　占用worker2个或4个连接数 ;
一个mater, 有4个worker，每个worker支持最大的连接数1024，　支持的最大并发数: 4 * 1024/2(4(作为反向代理))
=======
```

>>>>>>> Stashed changes

语法规则： location [=|~|~*|^~] /uri/ { … }

​```nginx
	= 开头表示精确匹配
	^~ 开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。nginx不对url做编码，因此请求为/static/20%/aa，可以被规则^~ /static/ /aa匹配到（注意是空格）。
	~ 开头表示区分大小写的正则匹配
	~*  开头表示不区分大小写的正则匹配
	!~和!~*分别为区分大小写不匹配及不区分大小写不匹配 的正则
	/ 通用匹配，任何请求都会匹配到。
	首先匹配 =, 其次匹配^~, 其次是按文件中顺序的正则匹配，最后是交给 / 通用匹配。当有匹配成功时候，停止匹配，按当前匹配规则处理请求。
```