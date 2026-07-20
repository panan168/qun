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