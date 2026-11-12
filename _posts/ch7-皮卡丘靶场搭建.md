---
layout: post
title: "在 Kali 里搭一个 Pikachu 靶场"
date: 2026-11-11
categories: 实验
---

Pikachu 是一个用 PHP 写的 Web 漏洞练习平台，按漏洞类型分成十几个模块：暴力破解、SQL 注入、XSS、文件包含、命令执行等等。每个模块里既有原理说明，也有一个能实际动手的页面。

它需要三样东西才能跑起来：一个 Web 服务器（Apache）、一个 PHP 解释器、一个数据库（MySQL）。这三样合起来叫 LAMP，是 PHP 网站的常见组合。

所以这个实验就是把这五件事做完：装上三个组件、把靶场代码放进网站目录、建数据库、改好配置、初始化。

装在 Kali 虚拟机里，没有装在 Windows 主机上。

## 环境

| 项目 | 说明 |
|---|---|
| 系统 | Kali Linux 2026.1 虚拟机 |
| 组件 | Apache + PHP 8.4 + MariaDB |
| 靶场 | Pikachu，源码放在 `/var/www/html/pikachu` |
| 访问 | 虚拟机里打开 `http://127.0.0.1/pikachu/` |

## 为什么装在 Kali 而不是 Windows

Windows 那边一般用小皮面板，点鼠标就能装，更省事。但 Web 渗透的题目基本都在 Kali 里做，Burp Suite 也自带，靶场和工具在同一个系统里，不用来回切。

Kali 的代价是没有图形面板，出问题得自己看报错。不过这些报错基本都能查到原因。

## 确认起点

```bash
php -v | head -2
echo "-----"
systemctl is-active apache2 mariadb
```

![图1：PHP 版本](截图/01.png)

![图2：两个服务的状态](截图/02.png)

第一次查的时候 PHP 根本没装，Apache 也没跑。所以下面从装组件开始。

## 装 Apache、PHP 和数据库

```bash
sudo apt update
sudo apt install -y apache2 mariadb-server php libapache2-mod-php php-mysqli php-gd php-mbstring
```

### 装到一半报 404

第一条命令跑完，第二条报了一屏错：

```
Failed to fetch http://http.kali.org/kali/pool/main/p/php8.4/php8.4-readline_8.4.23-1_amd64.deb  404  Not Found
Failed to fetch http://http.kali.org/kali/pool/main/p/php8.4/php8.4-opcache_8.4.23-1_amd64.deb   404  Not Found
```

![图3：php8.4 的几个包下载 404](截图/03.png)

一开始以为是网络问题，但 404 不是连不上，是服务器说“你要的这个文件不存在”。而且 mariadb 那些包都下成功了，只有 php 系列挂掉。

原因是 Kali 是滚动更新的发行版，仓库里的软件一直在换版本。本机的软件索引记的是 `php8.4 = 8.4.23-1`，但官方源上那个版本已经被新版本替换掉了，按老路径去要就得到 404。

解决办法就是刷新索引再装：

```bash
sudo apt update
sudo apt install -y php libapache2-mod-php php-mysqli php-gd php-mbstring
```

看到 `Setting up php (2:8.4+101)` 就装完了。

![图4：PHP 装好](截图/04.png)

装完重启 Apache 让模块生效：

```bash
sudo systemctl restart apache2
sudo systemctl enable apache2 mariadb
```

## 确认 Apache 在跑

```bash
sudo systemctl status apache2 --no-pager | head -5
sudo apache2ctl configtest
sudo ss -tlnp | grep ':80'
```

![图5：active (running)，Syntax OK，80 端口在监听](截图/05.png)

三条分别是：服务在不在跑、配置文件语法对不对、80 端口上有没有人在听。

## 下载靶场源码

```bash
cd /tmp
wget "https://codeload.github.com/zhuifengshaonianhanlu/pikachu/zip/refs/heads/master" -O pikachu.zip
```

解压并放到网站目录：

```bash
unzip -q pikachu.zip
sudo mv pikachu-master /var/www/html/pikachu
sudo chown -R www-data:www-data /var/www/html/pikachu
ls /var/www/html/pikachu/
```

![图6：下载完成，目录里有 index.php、install.php、inc、vul](截图/06.png)

`chown` 是把属主改成 `www-data`。Apache 是用这个账号跑的，文件属主是 root 的话，程序想写缓存、写日志就会失败。

## 建数据库

```bash
sudo systemctl start mariadb

sudo mysql -e "CREATE DATABASE IF NOT EXISTS pikachu DEFAULT CHARSET utf8mb4;
CREATE USER IF NOT EXISTS 'pikachu'@'localhost' IDENTIFIED BY 'pikachu';
GRANT ALL PRIVILEGES ON pikachu.* TO 'pikachu'@'localhost';
FLUSH PRIVILEGES;
SHOW DATABASES;"
```

最后一句 `SHOW DATABASES` 会列出所有库，里面有 `pikachu` 就证明对了。

![图7：数据库列表里有 pikachu](截图/07.png)

## 改配置文件

靶场代码里写死了连数据库用的账号密码，默认是 `root`、密码为空。我建的是 `pikachu` 账号，所以要改过来。

配置文件有两个，都要改：

```bash
sudo sed -i "s/define('DBUSER', 'root')/define('DBUSER', 'pikachu')/" /var/www/html/pikachu/inc/config.inc.php
sudo sed -i "s/define('DBPW', '')/define('DBPW', 'pikachu')/" /var/www/html/pikachu/inc/config.inc.php
sudo sed -i "s/define('DBUSER', 'root')/define('DBUSER', 'pikachu')/" /var/www/html/pikachu/pkxss/inc/config.inc.php
sudo sed -i "s/define('DBPW', 'root')/define('DBPW', 'pikachu')/" /var/www/html/pikachu/pkxss/inc/config.inc.php
```

`pkxss` 是靶场自带的 XSS 后台，练 XSS 那部分要用它收 cookie。两个文件的密码必须一致，否则会出现主站能进、XSS 平台连不上数据库的情况。

注意两条 `sed` 的匹配内容不一样：主站那个文件默认密码是空的（`''`），`pkxss` 那个默认是 `root`。照抄成同一条会有一条改不动，而 `sed` 找不到匹配时不会报错，只是什么都不做。

## 初始化

浏览器打开：

```
http://127.0.0.1/pikachu/install.php
```

点页面中间的「安装/初始化」。它会自动建表、插初始数据。

![图8：四条红字提示，建表成功](截图/08.png)

页面提示「好了，可以开搞了」，下面有个链接进首页。

## 验证

先看表建出来没有：

```bash
sudo mysql -e "USE pikachu; SHOW TABLES;"
```

```
httpinfo
member
message
users
xssblind
```

![图9：五张表](截图/09.png)

再打开首页：

```
http://127.0.0.1/pikachu/
```

![图10：皮卡丘首页，左侧是漏洞菜单](截图/10.png)

左侧十几个模块都能点进去，页面中文正常，没有报错刷屏。

## 两个容易搞混的地方

**`127.0.0.1` 指的是“我这台机器”。** 在 Windows 的浏览器里敲它，访问的是 Windows 本机，跟虚拟机里的 Apache 没关系。要么在 Kali 里开浏览器，要么在 Windows 里用 Kali 的地址 `http://192.168.200.131/`。

**80 端口只能有一个程序用。** 我之前做的 EFSS 实验也占 80。以后要重做那个实验，得先停掉 Apache：

```bash
sudo systemctl stop apache2
```

到这一步靶场就能用了。左侧十几个模块点进去都有页面，但里面的漏洞一个还没打，实战另外再练。



## 用到的命令

- `sudo apt install -y apache2 mariadb-server php libapache2-mod-php php-mysqli`：装 LAMP
- `sudo systemctl enable apache2 mariadb`：设开机自启
- `sudo ss -tlnp | grep ':80'`：看 80 端口谁在听
- `sudo chown -R www-data:www-data 目录`：改属主，让 Web 进程能写
- `sudo mysql -e "..."`：在命令行执行 SQL
- `sudo sed -i "s/旧/新/" 文件`：直接替换文件内容
- `sudo mysql -e "USE pikachu; SHOW TABLES;"`：验证表建出来了
