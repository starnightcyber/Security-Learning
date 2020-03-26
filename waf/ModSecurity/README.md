# 开源WAF工具ModSecurity

## 前言
ModSecurity是一个开源的跨平台Web应用程序防火墙（WAF）引擎，用于Apache，IIS和Nginx，由Trustwave的SpiderLabs开发。
作为WAF产品，ModSecurity专门关注HTTP流量，当发出HTTP请求时，ModSecurity检查请求的所有部分，如果请求是恶意的，它会被阻止和记录。　

### 优势
* 完美兼容nginx，是nginx官方推荐的WAF
* 支持OWASP规则
* 3.0版本比老版本更新更快，更加稳定，并且得到了nginx、Inc和Trustwave等团队的积极支持
* 免费

#### 功能
* SQL Injection (SQLi)：阻止SQL注入
* Cross Site Scripting (XSS)：阻止跨站脚本攻击
* Local File Inclusion (LFI)：阻止利用本地文件包含漏洞进行攻击
* Remote File Inclusione(RFI)：阻止利用远程文件包含漏洞进行攻击
* Remote Code Execution (RCE)：阻止利用远程命令执行漏洞进行攻击
* PHP Code Injectiod：阻止PHP代码注入
* HTTP Protocol Violations：阻止违反HTTP协议的恶意访问
* HTTPoxy：阻止利用远程代理感染漏洞进行攻击
* Shellshock：阻止利用Shellshock漏洞进行攻击
* Session Fixation：阻止利用Session会话ID不变的漏洞进行攻击
* Scanner Detection：阻止黑客扫描网站
* Metadata/Error Leakages：阻止源代码/错误信息泄露
* Project Honey Pot Blacklist：蜜罐项目黑名单
* GeoIP Country Blocking：根据判断IP地址归属地来进行IP阻断

### 劣势

* 不支持检查响应体的规则，如果配置中包含这些规则，则会被忽略，nginx的的sub_filter指令可以用来检查状语从句：
  重写响应数据，OWASP中相关规则是95X。
* 不支持OWASP核心规则集DDoS规则REQUEST-912-DOS- PROTECTION.conf,nginx本身支持配置DDoS限制
* 不支持在审计日志中包含请求和响应主体

以上内容摘自：[ModSecurity：一款优秀的开源WAF](https://www.freebuf.com/sectool/211354.html)。

## 安装
### 安装nginx依赖
```
# yum install epel-release -y
# yum install gcc-c++ flex bison yajl yajl-devel curl-devel curl GeoIP-devel doxygen zlib-devel pcre pcre-devel libxml2 libxml2-devel autoconf automake lmdb-devel ssdeep-devel ssdeep-libs lua-devel libmaxminddb-devel git apt-utils autoconf automake build-essential git libcurl4-openssl-dev libgeoip-dev liblmdb-dev ibpcre++-dev libtool libxml2-dev libyajl-dev pkgconf wget zlib1g-dev -y
```

### 编译ModSecurity

我们用的是v3版本，我们在/opt目录下进行安装。

```
# cd /opt/　　　　# 切换到/opt
# git clone --depth 1 -b v3/master --single-branch https://github.com/SpiderLabs/ModSecurity　　　　# 下载
# cd ModSecurity/
# git submodule init　　　　  # 初始化
# git submodule update　　　　# 更新
# ./build.sh
# ./configure
# make
# make install
```

【注】在执行build.sh会出现如下错误，可忽略。
```
fatal: No names found, cannot describe anything
```

### ModSecurity-nginx连接器

我们现在需要将ModSecurity-nginx模块编入nginx。
```
# cd /opt/
# git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git
```
```
# nginx -v　　　　　　　　　　　　# 查看当前nginx版本
nginx version: nginx/1.17.5
# wget http://nginx.org/download/nginx-1.17.5.tar.gz
# tar -xvf nginx-1.17.5.tar.gz
# ls
ModSecurity  ModSecurity-nginx  nginx-1.17.5  nginx-1.17.5.tar.gz
# cd nginx-1.17.5/
# ./configure --with-compat --add-dynamic-module=../ModSecurity-nginx      # 如果出现不兼容的问题，请去掉--with-compat参数
# make modules                                  # 会生成如下*.so
# ls ./objs/ngx_http_modsecurity_module.so 
./objs/ngx_http_modsecurity_module.so　　　　　　　# 查看
# cp ./objs/ngx_http_modsecurity_module.so /etc/nginx/modules/   　　# 移动位置
```
```
# vim /etc/nginx/nginx.conf        
load_module /etc/nginx/modules/ngx_http_modsecurity_module.so;     # 添加到配置文件首行
# nginx -t　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　 # 测试通过
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

## 测试

### ECHO测试 

新增配置文件：```/etc/nginx/conf.d/echo.conf``` ：
```
# service nginx start　　　# 启动nginx
Redirecting to /bin/systemctl start nginx.service
# vim /etc/nginx/conf.d/echo.conf 
server {
    listen localhost:8085;
    location / {
        default_type text/plain;
        return 200 "Thank you for requesting ${request_uri}\n";
    }
}
# nginx -s reload　　　　# 重载配置
# nginx -t　　　　　　　　# 检测
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

使用curl命令测试：

```
[root@localhost ~]# curl -D - http://localhost:8085
HTTP/1.1 200 OK
Server: nginx/1.17.5
Date: Mon, 18 Nov 2019 05:35:40 GMT
Content-Type: text/plain
Content-Length: 27
Connection: keep-alive

Thank you for requesting /
[root@localhost ~]# curl -D - http://localhost:8085/notexist
HTTP/1.1 200 OK
Server: nginx/1.17.5
Date: Mon, 18 Nov 2019 05:35:49 GMT
Content-Type: text/plain
Content-Length: 35
Connection: keep-alive

Thank you for requesting /notexist
```

可以看到echo正常。

### 配置反向代理

新增配置文件：```/etc/nginx/conf.d/proxy.conf ```，内容如下：

```
[root@localhost ~]# cat /etc/nginx/conf.d/proxy.conf 
server {
    listen 80;
    location / {
        proxy_pass http://localhost:8085;
        proxy_set_header Host $host;
    }
}
```

因为正常安装后，nginx是有默认配置的：/etc/nginx/conf.d/default.conf，这个会影响到上面的正常生效。

```
[root@localhost ~]# mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.bak
[root@localhost ~]# nginx -s reload
[root@localhost ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

再次试用curl测试：

```
[root@localhost ~]# curl -D - http://localhost
HTTP/1.1 200 OK
Server: nginx/1.17.5
Date: Mon, 18 Nov 2019 05:43:05 GMT
Content-Type: text/plain
Content-Length: 27
Connection: keep-alive

Thank you for requesting /
[root@localhost ~]# curl -D - http://localhost/noexist
HTTP/1.1 200 OK
Server: nginx/1.17.5
Date: Mon, 18 Nov 2019 05:44:06 GMT
Content-Type: text/plain
Content-Length: 34
Connection: keep-alive

Thank you for requesting /noexist
[root@localhost ~]#
```

可以看到访问默认的80端口，会反向代理到8085端口。

### 启用WAF

配置NGINX WAF以通过阻止某些请求来保护演示web应用程序。

```
[root@localhost ~]# mkdir /etc/nginx/modsec
[root@localhost ~]# cd /etc/nginx/modsec
[root@localhost modsec]# sudo wget https://raw.githubusercontent.com/SpiderLabs/ModSecurity/v3/master/modsecurity.conf-recommended
[root@localhost modsec]# sudo mv modsecurity.conf-recommended modsecurity.conf
```

修改```modsecurity.conf```配置文件
```
[root@localhost modsec]# vim  modsecurity.conf 
# -- Rule engine initialization ----------------------------------------------
...
SecRuleEngine On    <== 设置为On
```

修改nginx waf配置文件：```/etc/nginx/modsec/main.conf``` ，添加相应规则。

```
# cat /etc/nginx/modsec/main.conf 
# Include the recommended configuration
Include /etc/nginx/modsec/modsecurity.conf
# A test rule
SecRule ARGS:testparam "@contains test" "id:1234,deny,log,status:403"
```
其中：
* Include：包括modsecurity.conf文件中建议的配置。
* SecRule：创建一个规则，当查询字符串中的testparam参数包含字符串test时，通过阻止请求并返回状态代码403来保护应用程序。修改nginx配置文件，来启用WAF防护。

```
# cat /etc/nginx/conf.d/proxy.conf 
server {
    listen 80;
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;
    location / {
        proxy_pass http://localhost:8085;
        proxy_set_header Host $host;
    }
}
```
其中：
* modsecurity on：启用Nginx WAF；
* modsecurity_rules_file：指定规则文件路径。

```
[root@localhost modsec]# cp /opt/ModSecurity/unicode.mapping /etc/nginx/modsec/　　　　# 需要先拷贝下unicode.mapping文件
[root@localhost modsec]# nginx -s reload　　　　　　　　# 重载配置
[root@localhost modsec]# nginx -t　　　　　　　　　　　　# 测试
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

测试参数中带有test，会被禁止。
```
[root@localhost modsec]# curl -D - http://localhost/foo?testparam=thisisatestofmodsecurity
HTTP/1.1 403 Forbidden
Server: nginx/1.17.5
Date: Mon, 18 Nov 2019 05:59:10 GMT
Content-Type: text/html
Content-Length: 153
Connection: keep-alive

<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.17.5</center>
</body>
</html>
```

### 日志记录功能

修改nginx配置文件：```/etc/nginx/nginx.conf```，

```
# vim /etc/nginx/nginx.conf 
load_module /opt/nginx-1.17.5/objs/ngx_http_modsecurity_module.so;　　　　# 加载模块
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log info;　　　　　　# 将错误日志设置为info级别
```

测试：
```
[root@localhost modsec]# nginx -s reload　　　　　　 # 重载配置
[root@localhost modsec]# nginx -t　　　　　　　　　　 # 测试
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@localhost modsec]# curl -D - http://localhost/foo?testparam=thisisatestofmodsecurity　　　　　　# 再次访问
HTTP/1.1 403 Forbidden
Server: nginx/1.17.5
Date: Mon, 18 Nov 2019 06:02:09 GMT
Content-Type: text/html
Content-Length: 153
Connection: keep-alive

<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.17.5</center>
</body>
</html>
[root@localhost modsec]# tail -5 /var/log/nginx/error.log 　　　　　　# 查看错误日志文件
2019/11/18 14:01:57 [notice] 24845#24845: worker process 25847 exited with code 0
2019/11/18 14:01:57 [notice] 24845#24845: signal 29 (SIGIO) received
2019/11/18 14:01:59 [notice] 25880#25880: ModSecurity-nginx v1.0.0 (rules loaded inline/local/remote: 0/7/0)
2019/11/18 14:02:09 [error] 25879#25879: *11 [client 127.0.0.1] ModSecurity: Access denied with code 403 (phase 1). Matched "Operator `Contains' with parameter `test' against variable `ARGS:testparam' (Value: `thisisatestofmodsecurity' ) [file "/etc/nginx/modsec/main.conf"] [line "4"] [id "1234"] [rev ""] [msg ""] [data ""] [severity "0"] [ver ""] [maturity "0"] [accuracy "0"] [hostname "127.0.0.1"] [uri "/foo"] [unique_id "157405692985.199277"] [ref "o7,4v19,24"], client: 127.0.0.1, server: , request: "GET /foo?testparam=thisisatestofmodsecurity HTTP/1.1", host: "localhost"
2019/11/18 14:02:09 [info] 25879#25879: *11 client 127.0.0.1 closed keepalive connection
```

