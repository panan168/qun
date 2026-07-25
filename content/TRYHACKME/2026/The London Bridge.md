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

### 3. 探索隐藏参数
