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
