---
  title:  Docker中安装nginx
  date: {{ date }}   
  categories: ['lunix'] 
  tags: ['lunix','centsos7','docker','nginx']       
  comments: true    
  img:             
---
## 准备
1.拉取nginx镜像

直接去dockerhub官网然后搜索ngnix就行了
>docker pull nginx:1.15.11(这个是我在用的一个版本)

2.科普
>科普下因为docker中所有项目都是一个容器所以这边运行nginx后他的配置文件也是在容器中的修改不方便所以我们要把他挂载出来。
docker中
 nginx的配置文件路径为
 /etc/nginx/nginx.conf 
 日志的路径为
 /var/log/nginx
资源路径为
/etc/nginx/html
以上为docker中运行nginx的默认配置路径

3.创建挂载的目录
>mkdir -p /usr/local/nginx/{conf,conf.d,html,logs}
(ps:创建的目录html下默认 conf 下是没文件的所以这边需要自己找配置文件)建议跟容器中配置文件保持一致（玩的很熟练的可无视）

4.配置文件详情
>nginx.conf(nginx默认配置)注意**资源路径**要改成nginx**容器**的资源路径而不是主机的资源路径

>举例:比如你把容器中某个目录/etc/html 映射到本机中的 /www  这时在配置文件中还是需要使用/etc/html而不是/www这个你配置文件**挂载**只是把容器中/etc/html **映射**到/www，所以容器中只能识别/etc/html所以在配置文件中的一些路径依然要为容器中的路径。



这个文件是我从容器中拷贝出来的，直接复制过去就好了。
```
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


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

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root  /usr/share/nginx/html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}
```
## 运行镜像
```
docker run -p 80:80 --name mynginx -v /usr/local/ngnix/logs/:/var/log/nginx -v /usr/local/ngnix/nginx.conf:/etc/nginx/nginx.conf  -v /usr/local/ngnix/html/:/etc/nginx/html -d nginx:1.15.11
 ```
如果出现一串密码样的就运行成功
dasdaf444115445112445412454
