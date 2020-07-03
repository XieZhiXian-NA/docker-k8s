# nginx

## IO多路复用

1. 多线程处理 资源消耗过多
   让一个socket连接并行处理多个IO流请求，再分发到不同的线程中。

2. IO流主动上报
   复用一个线程，在一个线程内并发交替地顺序完成
   内核会将io流（文件描述符）主动上报给应用程序

3. cpu亲和
   并发处理能力
   把cpu核心和Nginx工作进程绑定，把每个worker进程固定在一个cpu核心上执行，减少cpu切换的性能消耗

4. /etc/logrotate.d/nginx
   Nginx日志轮转，用于logrotate服务的日志切割

5. /etc/systemd 
   配置系统守护进程管理器管理方式

6. /etc/sbin/nginx
   nginx服务的启动管理的终端命令

7. /var/cache/nginx 
  开启缓存后缓存存放的位置

8. /var/log/nginx
  存放nginx log的目录

## log日志分类

- access.log
- error.log 
  
## 模块

nginx -t 验证配置文件是否正确
nginx -s reload /stop 快速停止 / quit正常停止或关闭
nginx -c xxx.conf使用另一个配置文件
start nginx 启动

+ --with-http_stub_status_module nginx客户端状态

```js
  //包含基本状态数据的简单网页
  location /mystatus{
      stub_status;
  }
```

+ --with-http_random_index_module 在目录中选择一个随机主页

```js
 //不会选择 .的隐藏文件
  location /{
      root /opt;
      random_index on;
  }
```

+ --width-http_sub_module http内容替换 默认没有安装

+ limit_conn_module 连接频率的限制 tcp三次握手
  
```js
  // 存储连接数的区域 key-value的形式存储，$remote_addr--ip $server_name--服务器名
  // name 存储区域的名字
  limit_conn_zone key zone=name:size
  limit_conn name number;//number 同一时刻一个key允许连接的数量
```

+ limit_req_module 请求频率的限制  一次连接上多次发送请求

```js
//rate  查询该区域的速率 1r/s 1s一个
  limit_req_zone key zone=name:size rate=rate;
//burst 当速率超过了1r/s，将x个请求延迟到下一秒
//nodelay 其余的请求不延迟
  limit_req zone=name [burst=number] [nodelay]

```

+ nginx访问控制

  - http_acccess_module 基于IP的访问控制
    局限性 只是基于$remote_addr,直接与服务器交互的ip地址，若经过了代理，则识别不出来了。

     ```js
        allow address | CIDR(网段) | all;
        deny address | CIDR | all;

        location /{
            root /opt;
            index index.html index.html;
        }

        location ~ ^/admin.html{
           //允许除了120.198.22.24以外的所有ip访问
           root /opt;
           //deny 120.198.22.24;
           //allow all;
           allow 120.198.22.24/24;//允许该网段的ip可以访问
           deny all;
           index index.html index.html;
        }

        = 严格匹配。如果这个查询匹配，那么将停止搜索并立即处理此请求。
        ~ 为区分大小写匹配(可用正则表达式)
        !~为区分大小写不匹配
        ~* 为不区分大小写匹配(可用正则表达式)
        !~*为不区分大小写不匹配
        ^~ 如果把这个前缀用于一个常规字符串,那么告诉nginx 如果路径匹配那么不测试正则表达式。
     ```

  - http_auth_basic_module基于用户的信任登录
    局限性：依赖文件，效率不高

    ```js
        //string会返回给客户端
        auth_basic string | off;
        //file存放的是用户名和密码的文件，具有一定的格式,密码
        auth_basic_user_file file;
        location ~ ^/admin.html{
            root /opt;
            index index.html index.html; 
            auth_basic  "Auth access test!input your password";
            auth_basic_user_file /etc/nginx/auth_conf;
        }

    ```

  - nginx+LUA高效验证

## nginx静态资源

+ 静态资源WEB服务
  CDN 传输延时的最小化

  ```js
     //文件读取
     sendfile on | off;
     //sendfile开启的情况下，提高网络包的传输效率
     //对包进行整理，一次性发送多个包，大文件下
     tcp_nopush on | off;
     //实时性的文件 keepalive连接下，提高网络包的传输实时性
     tcp_nodelay on | off;
     //传输的压缩
     gzip on | off;
     //预读gzip功能，先去找是否有已经压缩了的文件
     gzip_static on | off | always;

     location ~ .*\.(jpg|gif|png)${
         gzip on;
         gzip_http_version 1.1;
         gzip_comp_level 2;
         gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpg image/gif image/png;
         root /opt/images;
     }

     location ~.*\.(txt|xml)${
         gzip on;
         gzip_http_version 1.1;
         gzip_comp_level 2;
         gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/gif image/png;
         root /opt/doc;
     }

     location ~^/download{
         gzip_static on;
         tcp_nopush on;
         root /opt/code;
     }

  ```

+ 动态缓存
  校验本地缓存文件是否过期，根据文件的http的响应头头中的Expires Cache-Control(max-age) 在request-header中会自动加 max-age<=0 则客户端每次都会向服务端发起校验，
  Etag 会向服务器进项校验，一串特殊的字符串（）304
  Last-Modified 会向服务器进项校验 具体的时间格式--只精确到秒，不准确（当服务器中的文件发生了更改，则会更新服务器中的该文件的Last-Modified，则两个请求一对比，不一样，重新返回新的文件）。

  ```js
     //给http头信息添加Cache-Control Expires
    expires [modified] time;
    expires epoch | max | off;
    location ~ .*\.(html | htm)${
        expires 24h;// 在服务端的响应头中添加max-age，但是浏览器会自动在请求头中添加max-age=0，请求遵循的是这个规则，不会走服务端的设置。
        root /opt/code
    }
  ```

+ 跨站访问
  
  ```js
     //name---Access-Control-Allow-Origin 跨域访问
     // Access-Control-Allow-Methods
     add_header name value [always]

  ```

+ 防盗链
  也就是让服务端识别指定的Referer，在服务端接收到请求时，通过匹配referer头域与配置，对于指定放行，对于其他referer视为盗链。防止资源被盗用。
  http_refer 上一次请求的地址。

  ```js
  //允许哪些地址过来访问，值为1
    valid_referers none blocked 116.62.103.228;
    if($valid_referers) return 403
  ```

+ 代理服务
  正向代理对象--客户端--翻墙
  反向代理对象--服务端
  
  ```js
     proxy_pass URL
     //nginx会尽可能多的将服务器response放到该缓冲区，再返回给客户端
     //禁用缓冲后，响应一收到就立即同步传递到客户端。nginx不会尝试从代理服务器读取整个响应
     pass_buffering on | off
     //跳转重定向，当服务端返回304并且客户端处理不了重定向时
     proxy_redirect default;
     //重新对请求头设置一些变量,携带给后端
     proxy_set_header Host $proxy_host
     proxy_set_header X-Real-IP $remote_addr;//用户真实的ip地址
     //将该请求转到8080端口上---反向代理，客户端不关心具体访问的服务器地址
     location ~/test_Proxy.html${
       proxy_pass http://127.0.0.1:8080;
     }
     //正向代理,当某个网站只允许111.134.233.228IP访问，若我需要访问，则将我的访问转为可以访问的IP地址，再将结果扔给我，翻墙
     location /{//228Ip的服务器上
       proxy_pass http://$http_host$http_request_url;
     }
  
  ```

+ 负载均衡
  虚拟服务组

  ```js
  //在http层级 weight权重越大，访问概率越大
  //对后端服务端设置
  upstream xlx {
    server 116.62.10.228:8001 weight = 5;
    server 116.62.10.228:8002;
    server 116.62.10.228:8003;
  }
  location /{
    //这里的域名与upstream的name一致
    proxy_pass http://xlx;
  }
  down:当前server暂时不参与负载均衡，不对外提供服务。
  backup:预留的备份服务器，当其他所有节点都不能提供服务时，它启动服务。
  max_fails: 允许请求失败的次数，当向后台查询失败时，会再次查询，直到到达该次数。
  fail_timeout: 经过max_fails失败后，服务暂停的时间，不再去查询，等待该时间后，再次查询，默认10s。
  max_conns:限制最大的接收的连接数
  ```

  轮询策略，调度算法
  
  ```js
   加权轮询: weight值越大，分配到的访问几率越高
   ip_hash: 每个请求访问IP的hash结果分配，这样来自同一个IP的固定访问一个后端服务器-----[cookie session]一致,可能会拿不到真实的ip
   url_hash: 按照访问的URL的hash结果来分配请求，是每个URL定向到同一个后端服务器 url_hash key 
   //url_hash $request_uri; jeson.immoc.io/jecore.html是根据jecore.html来只定向到同一个服务器上。
   //当对于http://shu/dhfue?name=sss&&age=17等复杂的后缀时，正则提取，再自定义key值
   least_conn: 最少连接数，哪个机器数连接数少就分发给谁
   hash关键数值: hash自定义的key
  ```

  缓存处理
  减少后端的压力
  服务端缓存：redis等
  代理缓存：从服务端获取到的内容，再存到内存
  客户端缓存：存储到本地
  ![代理缓存](./代理缓存.jpg)

  ```js
  proxy_cache_path path; //存放缓存的文件路径
  proxy_cache zone | off; //缓存区
  proxy_cache_valid  [code] time;//time时间内过期
  proxy_cache_key $host$uri$is_args$args;
  // levels 两层目录分级，缓存没有都放置在同一个文件夹下。10兆 8万个key
  // 若缓存文件在1h内没有被访问过，就将其清除掉
  proxy_cache_path /opt/app/cache levels=1:2 keys_zone = xzx_cache:10m max_size=10g inactive=60m use_temp_path=off
  if($request_uri ~^/(url3 | login | register |password )){
    //当请求的路径以这些开头时，这些页面就不会被缓存
    set $cookie_nocache 1;
  }
  location /{
        proxy_cache xzx_cache;
        proxy_pass http://xlx;
        proxy_cache_valid 200 304 12h;//对状态码为200 304的请求进行缓存,过期时间为12h
        proxy_cache_valid any 10m;//除了其他以外的缓存10m
        proxy_cache_key $scheme$proxy_host$request_uri;//作为缓存的内容
        proxy_no_cache $cookie_nocache;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;//当有一台服务器返回这些错误时，直接跳过该台服务器，
  }
  ```

+ 大文件分片请求
  slice size; http_slice_module

## 动静分离

通过中间件将动态请求和静态请求分离，

+ rewrite
  实现url重写以及重定向

  ```js
   rewrite regex replacement[flag];
   flag
   last: 停止rewrite检测
   break: 停止rewrite检测
   redirect: 返回302临时重定向，地址栏会显示跳转后的地址,发起两次请求
   permanent: 返回301永久重定向，地址栏会显示跳转后的地址,当清除缓存以后，才不会重定向，否则即使服务器关了，也会重定向
   //正则匹配() 用于匹配括号之间的内容，将匹配的结果放在通过$1,$2调用
   //把所有的请求都重定向到maintain.html下
   rewrite ^(.*)$ /pages/maintain.html break;
  
   root /opt/app/code
   location ~^/break{
     //此时当访问该路径，会重写路径，但是由于该/code路径下没有test文件，则直接就返回了404,不会再去匹配后面的location
     rewrite ^/break /test/ break;
   }
   location ~^/last{
     //此时请求该路径，尽管路径下没有该文件，会重新发起一次http://test/的请求,去匹配后面的location
     rewrite ^/last /test/ last;
   }
   location ~^/test{
     return 200 '{"state":"success"}';
   }

  ```

+ secure_link_module 
  下载资源后端加密再返给前端,防止dist目录下的静态资源被随意的下载访问
  制定并允许检查请求的链接的真实行以及保护资源免遭未经授权的访问。
  限制链接生效周期
  ![验证链接](./secure_link.jpg)

  ```js
    secure_link expression
    secure_link_md5 expression//判断拿到的参数变量是否合法，根据与后端约定的加密方式，重新加密 xzx加密密串
    //下载地址 
    /download?md5=sdjfwejfiwjfiwej&expires=12314242543
    location /{
      //取地址中的参数名
      secure_link $arg_md5,$arg_expires;
      secure_link_md5 "$secure_link_expires$uri xzx"
      if($secure_link = '') return 403
      if($secure_link = '0') return 410
    }
  ```

+ geoip 读取IP所在的地域信息
 区别国内国外做HTTP访问规则 
 区别国内其他城市地域做http访问规则

```js
   //该模块没有加载配置，需要引入该模块
   load_module "modules/ngx_http_geoip_module.so";
   load_module "modules/ngx_stream_geoip_module.so";
```

+ location匹配顺序
相同server_name的匹配顺序，默认优先读取第一个
location优先级
= 优先级最高 放置最高级
^~ 优先级高 前缀匹配 匹配到 不会继续向下寻找
~\~* 优先级不高，匹配到，还会继续往下去寻找是否有更加匹配的路径
~区分大小写 ~*不区分大小写

+ try_files

```js
  root F:\\showwrite\\dist;
  location /xzx/{
    try_files $uri  /xzx/index.html;
  }  
  http://localhost:6888/xzx/js/app.056a014a.js
  资源路径是是两者的拼接 F:\showwrite\dist\xzx\js\app.056a014a.js目录的拼接
  当请求的路径是静态资源时，直接访问dist/xzx/目录下的文件，若不是，回退显示/xzx/index.html文件
```