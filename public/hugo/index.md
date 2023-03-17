# 使用Hugo搭建博客


### 1.系统环境 
CentOS Linux release 8.2.2004 (Core)

### 2.安装Golang
ver>1.18, 用来编译hugo

[Golang安装教程](https://go.dev/doc/install)

安装后设置go proxy

vim ~/.profile
``` shell
# 配置 GOPROXY 环境变量
export GOPROXY=https://proxy.golang.com.cn,direct
# 还可以设置不走 proxy 的私有仓库或组，多个用逗号相隔（可选）
export GOPRIVATE=git.mycompany.com,github.com/my/private

```
source ~/.profile


### 3.安装Hugo

安装gcc
``` shell
yum install -y gcc-c++

```

编译hugo
``` shell
go install -tags extended github.com/gohugoio/hugo@latest
hugo version

```
至此hugo安装完成

### 4.安装主题

创建站点

``` shell
hugo new site quickstart

```

添加主题
``` shell
cd quickstart
git init
git submodule add https://github.com/dillonzq/LoveIt themes/LoveIt
echo "theme = 'LoveIt'" >> config.toml

```

### 4.运行网站

新建about页面
新建blog
``` shell
//about页面
hugo new about.md
//blog
hugo new posts/first.md

```

启动网站, 在网站根目录执行后台运行命令
``` shell
nohup hugo server -b "http://你自己的公网ip:80" -p 80 --bind "0.0.0.0" -D &

```

- `b http://ip:8080`是由于其他人访问的时候，静态资源默认是localhost，所以需要指向自己ip地址和默认端口。
- `p 8080`默认端口1313，指定端口8080
- `-bind "0.0.0.0"`，默认启动只允许本机访问，所以需要bind所有机器都可以访问。

在管理后台设置开放端口80

在浏览器输入,查看网站效果
http://ip:80

### 5.主题优化
#### 5.1 设置icp备案
#### 5.2 设置访问统计

