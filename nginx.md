##　nginx

1. 是什么?  
    反向代理服务器, 负载均衡, 动静分离, 健康检查
    
2. 有什么功能?  
    项目部署在不同的服务器上，但是通过统一的域名进入，nginx则对请求进行分发，减轻服务器的压力
    
3. 目录所在?  

```
    /usr/sbin/nginx       主程序
    /etc/nginx            存放配置文件
    /usr/share/nginx      存放静态文件
    /var/log/nginx        存放日志
```

4. 如何配置?

```
    1. 进入hosts文件:
    sudo vim /etc/hosts
    
    2. 内容 ip映射
    127.0.0.1  8080.www.demo.com
    127.0.0.1  8081.www.demo.com
```

5. 配置反向代理(基本)

```
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

```
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

```
    weight;   权重
    ip_hash;　ip绑定
    fair: 后端响应时间
    
    
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
        #fair
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

```
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

```
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

```
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

```
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

```
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

```
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
yum install keepalived -y
#　配置文件
/etc/keepalived.conf

# 修改配置文件
```

语法规则： location [=|~|~*|^~] /uri/ { … }

```
	= 开头表示精确匹配
	^~ 开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。nginx不对url做编码，因此请求为/static/20%/aa，可以被规则^~ /static/ /aa匹配到（注意是空格）。
	~ 开头表示区分大小写的正则匹配
	~*  开头表示不区分大小写的正则匹配
	!~和!~*分别为区分大小写不匹配及不区分大小写不匹配 的正则
	/ 通用匹配，任何请求都会匹配到。
	首先匹配 =, 其次匹配^~, 其次是按文件中顺序的正则匹配，最后是交给 / 通用匹配。当有匹配成功时候，停止匹配，按当前匹配规则处理请求。
```