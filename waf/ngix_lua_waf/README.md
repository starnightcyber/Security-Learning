# OpenResty + ngx_lua_waf安装使用

本篇介绍在CentOS7.6上安装、测试使用ngx_lua_waf + openresty。

## Preface

```
# yum install epel-release -y
# yum group install "Development Tools" -y　　　　# 安装基本编译工具
```

## 安装Luagit

```
# cd /opt/
# wget http://luajit.org/download/LuaJIT-2.1.0-beta3.tar.gz
# tar -xvf LuaJIT-2.1.0-beta3.tar.gz
# cd LuaJIT-2.1.0-beta3/
# make && make install
# ln -sf luajit-2.1.0-beta3 /usr/local/bin/luajit
```

## 安装OpenResty

### 安装依赖

```
# yum install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel -y
```

### 下载安装

```
# cd /opt/
# wget https://openresty.org/download/openresty-1.13.6.1.tar.gz
# tar -xvf openresty-1.13.6.1.tar.gz
# cd openresty-1.13.6.1/
# ./configure --prefix=/usr/local/openresty --with-luajit --with-http_stub_status_module --with-pcre --with-pcre-jit
# gmake && gmake install
# ln -sf /usr/local/openresty/nginx/sbin/nginx /usr/local/openresty/bin/openresty
```

```
# /usr/local/openresty/bin/openresty   # 启动openresty
# netstat -lntp | grep 80              # 服务运行正常
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN 
```

## 配置ngx_lua_waf

### 下载

```
# cd /usr/local/openresty/nginx/conf　　　　　　　　　　　　　　　　　　# 到openresty配置文件目录
# git clone https://github.com/loveshell/ngx_lua_waf.git　　　　　　# 下载
Cloning into 'ngx_lua_waf'...
# mv ngx_lua_waf/ waf/　　　　　　　　　　　　　　　　　　　　　　　　　　 # 改个简单的名字
```

### 添加配置
修改openresty配置文件。

```
# vim /usr/local/openresty/nginx/conf/nginx.conf
...
user nobody;     # 取消注释  
...
http{            # 在http块下添加如下内
...
lua_package_path "/usr/local/openresty/nginx/conf/waf/?.lua";
lua_shared_dict limit 10m;
init_by_lua_file  /usr/local/openresty/nginx/conf/waf/init.lua;
access_by_lua_file /usr/local/openresty/nginx/conf/waf/waf.lua;
...
```

### 修改ngx_lua_waf配置

```
# cd /usr/local/openresty/nginx/conf/waf/     # ngx_lua_waf目录
# vim config.lua
...
RulePath = "/usr/local/openresty/nginx/conf/waf/wafconf/"    # 规则文件路径
attacklog = "on"                                             # 启用日志
logdir = "/usr/local/openresty/nginx/logs/hack/"             # 日志目录
...
```

### 创建日志目录

```
# mkdir -p /usr/local/openresty/nginx/logs/hack/
# chown -R nobody:nobody  /usr/local/openresty/nginx/logs/hack/
```

### 测试openresty配置是否正常：

```
# /usr/local/openresty/bin/openresty               # 如果没有启动服务，则启动
# /usr/local/openresty/bin/openresty -s reload     # 如果已经启动，则重载配置
# /usr/local/openresty/bin/openresty -t            # 测试配置是否正常
nginx: the configuration file /usr/local/openresty/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/openresty/nginx/conf/nginx.conf test is successful
```

## WAF测试

直接访问站点（此处为虚拟机的ip地址），应该能看到Openresty的欢迎页。



### 攻击测试

 尝试目录遍历攻击，即使用../，跳转目录读取文件。

访问：```http://192.168.139.139//test.php?id=../etc/passwd```

提示，检测到攻击行为，请求被拦截，说明可以正常工作。

##  ngx_lua_waf配置文件说明

```
RulePath = "/usr/local/openresty/nginx/ngx_lua_waf/wafconf/"   ##指定相应位置
attacklog = "on"                            ##开启日志
logdir = "/usr/local/openresty/nginx/logs/hack/"           ##日志存放位置
UrlDeny="on"                              ##是否开启URL防护
Redirect="on"                             ##地址重定向
CookieMatch="on"                           ##cookie拦截
postMatch="on"                            ##post拦截
whiteModule="on"                           ##白名单
black_fileExt={"php","jsp"}                        
ipWhitelist={"127.0.0.1"}                    ##白名单IP
ipBlocklist={"1.0.0.1"}                     ##黑名单IP
CCDeny="on"                             ##开启CC防护        
CCrate="100/60"                          ##60秒内允许同一个IP访问100次
```

## 参考

[centos下安装openresty+ngx_lua_waf防火墙部署](https://stepwen.github.io/2018/08/29/centos%E4%B8%8B%E5%AE%89%E8%A3%85openresty-ngx-lua-waf%E9%98%B2%E7%81%AB%E5%A2%99%E9%83%A8%E7%BD%B2/)

https://github.com/loveshell/ngx_lua_waf

https://blog.51cto.com/tar0cissp/1980249

注：参考链接中都有几个小问题，写的不够清晰。
