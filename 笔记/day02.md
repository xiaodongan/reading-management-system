# 项目需求分析

## 项目技术架构

#### 应用

- 积云图书管理后台

  

#### ![1611887545764](C:\Users\an\AppData\Roaming\Typora\typora-user-images\1611887545764.png)

### 项目目标

- 完全在本地搭建开发环境
- 贴近企业真实应用场景

> 依赖别人提供的 API 将无法真正理解项目的运作逻辑

## 技术难点分析

### 登录

- 用户名密码校验
- token 生成、校验和路由过滤
- 前端 token 校验和重定向

### 电子书上传

- 文件上传
- 静态资源服务器

### 电子书解析

- epub 原理
- zip 解压
- xml 解析

### 电子书增删改

- mysql 数据库应用
- 前后端异常处理

## epub 电子书

epub 是一种电子书格式，它的本质是一个 zip 压缩包

## Nginx 服务器搭建

### 安装 nginx

- windows 通过下载官网安装包，下载地址：http://nginx.org/en/download.html
- mac 通过 brew 安装，参考：https://www.jianshu.com/p/c3294887c6b6

> 使用 macOS 的同学可能会碰到无法写入 /usr/local 问题，后面会提供解决方法

### 修改配置文件

打开配置文件 nginx.conf：

- windows 位于安装目录下
- macOS 位于：`/usr/local/etc/nginx/nginx.conf`

修改一：添加当前登录用户为owner     

```sh
user an owner;  //  an 为当前电脑的登录用户
```

修改二：在结尾大括号之前添加：

```sh
include D:/upload/upload.conf;   
```

这里 `D:/upload` 是资源文件路径，` `D:/upload/upload.conf` 是额外的配置文件，当前把 ` `D:/upload/upload.conf` 配置文件的内容加入 nginx.conf 也是可行的！

> 使用 windows 的同学可能会碰到路径配置错误导致 500 的情况，最后一节有解法

修改三：添加 ` `D:/upload/upload.conf` 文件，配置如下：

```sh
server
{ 
  charset utf-8;
  listen 8089;
  server_name http_host;
  root  D:/upload/;
  autoindex on;
  add_header Cache-Control "no-cache, must-revalidate";
  location / { 
    add_header Access-Control-Allow-Origin *;
  }
}
```

如果需要加入 https 服务，可以再添加一个 server：  但是需要申请一些域名证书等，教学中如果没有不推荐使用

```sh
server
{
  listen 443 default ssl;
  server_name https_host;
  root  D:/upload/;
  autoindex on;
  add_header Cache-Control "no-cache, must-revalidate";
  location / {
    add_header Access-Control-Allow-Origin *;
  }
  ssl_certificate /Users/an/Desktop/https/book_an_xyz.pem;
  ssl_certificate_key /Users/an/Desktop/https/book_an_xyz.key;
  ssl_session_timeout  5m;
  ssl_protocols  SSLv3 TLSv1;
  ssl_ciphers  ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
  ssl_prefer_server_ciphers  on;
}
```

- https证书：/Users/an/Desktop/https/book_an_xyz.pem
- https:私钥：/Users/an/Desktop/https/book_an_xyz.key

### 启动服务

进入到nginx对应的目录下：运行cmd

启动 nginx 服务：  

```
//mac
sudo nginx

//windows
start nginx
```

重启 nginx 服务：

```
sudo nginx -s reload

nginx.exe -s reload
```

停止 nginx 服务：

```
sudo nginx -s stop

nginx.exe -s stop
```

检查配置文件是否存在语法错误：

```
sudo nginx -t

nginx -t
```

访问地址：

- http: `http://localhost:8089`
- https: `https://localhost`

> https 会提示证书有风险，不用理会，直接选择继续访问即可



## MySQL 数据库搭建

### 安装 MySQL

地址：https://dev.mysql.com/downloads/mysql/

### 安装 Navicat

地址：https://www.navicat.com.cn/products

### 启动 MySQL

windows 同学参考：https://blog.csdn.net/ycxzuoxin/article/details/80908447

mac 同学参考：https://blog.csdn.net/qq_25628891/article/details/88431942

```sh
cd /usr/local/mysql-8.0.13-macos10.14-x86_64/bin
./mysqld
```

### 初始化数据库

创建数据库 book

执行 book.sql 导入数据

#### 购买HTTPS证书

