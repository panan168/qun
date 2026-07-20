# Athena

> 🚩 **Athena** — TryHackMe
> 
> [👉 点击进入靶场房间](https://tryhackme.com/room/4th3n4)
> 
---

### 1. 信息收集 (Recon)
执行端口扫描：
```bash
nmap -sC -sV 10.10.x.x
```
用Nmap扫描目标时，我们可以发现有四个开放端口，每个端口运行不同的服务。端口22为SSH，端口为80，在139/445端口为SMB

![扫描结果](./assets/Athena_01.png)

执行目录爆破工具:
```bash
gobuster dir -u 10.48.155.161 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```
好像没有我们想要的目录

![扫描结果](./assets/Athena_02.png)

访问网站时，我们只看到一个静态页面，没有可用的链接。

![扫描结果](./assets/Athena_03.png)

查看源代码，并没有什么值得关注

![扫描结果](./assets/Athena_04.png)

