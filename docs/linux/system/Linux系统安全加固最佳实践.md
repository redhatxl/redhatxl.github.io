# Linux 系统安全加固最佳实践

## 前言

当云计算重构IT产业的同时，也赋予了企业崭新的增长机遇。通过充分利用云计算的能力，企业可以释放更多精力专注于自己的业务。云计算极大地降低了企业的数字化转型成本，释放更多效能进行业务创新，云计算为企业业务创新带来无限可能。但是当人们在享受使用云计算带来的便利的同时，服务器安全也不容忽视。

Linux是一套免费使用和自由传播的类Unix操作系统，作为一个开放源代码的操作系统，Linux服务器以其安全、高效和稳定的显著优势而得以广泛应用，但如果不做好权限的合理分配，Linux系统的安全性还是会得不到更好的保障。

不过经过安全加固后的Linux系统可达到B1的安全级别，相信大多数同学都有服务器被黑客入侵的经历，通过本文Linux系统从不同维度进行安全加固，一定程度增加系统的安全性，构筑更加坚实的安全壁垒。

## 一 身份鉴别

### 1.1 密码安全策略

操作系统和数据库系统管理用户身份鉴别信息应具有不易被冒用的特点，口令应有复杂度要求并定期更换。

设置有效的密码策略，防止攻击者破解出密码

* 查看空口令帐号并为弱/空口令帐号设置强密码

```
# awk -F: '($2 == ""){print $1}' /etc/shadow
```

可用离线破解、暴力字典破解或者密码网站查询出帐号密钥的密码是否是弱口令

* 修改vi /etc/login.defs配置密码周期策略

![image-20210623141920629](/Users/xuel/Library/Application Support/typora-user-images/image-20210623141920629.png)

此策略只对策略实施后所创建的帐号生效， 以前的帐号还是按99999天周期时间来算。

* /etc/pam.d/system-auth配置密码复杂度：

```shell
password requisite  pam_cracklib.so retry=3 difok=2 minlen=8 lcredit=-1 dcredit=-1

```

参数含义如下所示：

difok：本次密码与上次密码至少不同字符数

minlen：密码最小长度，此配置优先于login.defs中的PASS_MAX_DAYS

ucredit：最少大写字母

lcredit：最少小写字母

dcredit：最少数字

retry：重试多少次后返回密码修改错误

【注】用root修改其他帐号都不受密码周期及复杂度配置的影响。

### 1.2 登录失败策略

 应启用登录失败处理功能，可采取结束会话、限制非法登录次数和自动退出等措施。

遭遇密码破解时，暂时锁定帐号，降低密码被猜解的可能性

* 方法一：/etc/pam.d/login中设定控制台；/etc/pam.d/sshd中设定SSH

  /etc/pam.d/sshd中第二行添加下列信息

```
auth required  pam_tally2.so deny=5 lock_time=2 even_deny_root unlock_time=60
```

\# pam_tally2 --user root

解锁用户

\# pam_tally2 -r -u root

even_deny_root    也限制root用户(默认配置就锁定root帐号)；   
deny     设置普通用户和root用户连续错误登陆的最大次数，超过最大次数，则锁定该用户   
unlock_time     设定普通用户锁定后，多少时间后解锁，单位是秒； 

root_unlock_time      设定root用户锁定后，多少时间后解锁，单位是秒；

### 1.3 安全的远程管理方式

当对服务器进行远程管理时，应采取必要措施，防止鉴别信息在网络传输过程中被窃听。

防止远程管理过程中，密码等敏感信息被窃听

执行如下语句，查看telnet服务是否在运行

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210623171230.png)

禁止telnet运行，禁止开机启动，如下图：

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210623171252.png)

## 二 访问控制

### 2.1 账号及密码

应及时删除多余的、过期的帐户，避免共享帐户的存在。

删除或禁用临时、过期及可疑的帐号，防止被非法利用。

主要是管理员创建的普通帐号，如：test

``` shell
# usermod -L user 禁用帐号，帐号无法登录，/etc/shadow第二栏显示为!开头

# userdel user 删除user用户

# userdel -r user将删除user用户，并且将/home目录下的user目录一并删除
```

### 2.2 检查特殊账号

检查是否存在空口令和root权限的账号。

**操作步骤**

1. 查看空口令和root权限账号，确认是否存在异常账号：
   - 使用命令 `awk -F: '($2=="")' /etc/shadow` 查看空口令账号。
   - 使用命令 `awk -F: '($3==0)' /etc/passwd` 查看UID为零的账号。
2. 加固空口令账号：
   - 使用命令 `passwd <用户名>` 为空口令账号设定密码。
   - 确认UID为零的账号只有root账号。

### 2.3 添加密码策略

加强口令的复杂度等，降低被猜解的可能性。

**操作步骤**

1. 使用命令

   ```
   vi /etc/login.defs 
   ```

   修改配置文件。

   - `PASS_MAX_DAYS 90 #新建用户的密码最长使用天数`
   - `PASS_MIN_DAYS 0 #新建用户的密码最短使用天数`
   - `PASS_WARN_AGE 7 #新建用户的密码到期提前提醒天数`

2. 使用chage命令修改用户设置。
   例如，`chage -m 0 -M 30 -E 2000-01-01 -W 7 <用户名>`表示将此用户的密码最长使用天数设为30，最短使用天数设为0，密码2000年1月1日过期，过期前七天警告用户。

3. 设置连续输错三次密码，账号锁定五分钟。使用命令 `vi /etc/pam.d/common-auth`修改配置文件，在配置文件中添加 `auth required pam_tally.so onerr=fail deny=3 unlock_time=300`。

### 2.4 限制能su到root的用户及禁用root登陆

使用命令 `vi /etc/pam.d/su`修改配置文件，在配置文件中添加行。例如，只允许test组用户su到root，则添加 `auth required pam_wheel.so group=test`。

禁止root直接登陆。

1. 创建普通权限账号并配置密码,防止无法远程登录;
2. 使用命令 `vi /etc/ssh/sshd_config`修改配置文件将PermitRootLogin的值改成no，并保存，然后使用`service sshd restart`重启服务。

## 三 安全审计

### 3.1 审核策略开启

审计范围应覆盖到服务器和重要客户端上的每个操作系统用户和数据库用户；

开启审核策略，若日后系统出现故障、安全事故则可以查看系统日志文件，排除故障、追查入侵者的信息等。

查看rsyslog与auditd服务是否开启

rsyslog一般都会开启，auditd如没开启，执行如下命令：

 ```shell
# systemctl start auditd

 ```

auditd服务开机启动

```shell
# systemctl start auditd

```

### 3.2 日志属性设置

应保护审计记录，避免受到未预期的删除、修改或覆盖等。

防止重要日志信息被覆盖

让日志文件转储一个月，保留6个月的信息，先查看目前配置，

```
# more /etc/logrotate.conf | grep -v "^#\|^$"
```

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20210623171533.png)

### 3.3 启用syslog

启用日志功能，并配置日志记录。

**操作步骤**

Linux系统默认启用以下类型日志：

- 系统日志（默认）/var/log/messages
- cron日志（默认）/var/log/cron
- 安全日志（默认）/var/log/secure

**注意**：部分系统可能使用syslog-ng日志，配置文件为：/etc/syslog-ng/syslog-ng.conf。

您可以根据需求配置详细日志。

## 四 入侵防御

操作系统遵循最小安装的原则，仅安装需要的组件和应用程序，并通过设置升级服务器等方式保持系统补丁及时得到更新。

关闭与系统业务无关或不必要的服务，减小系统被黑客被攻击、渗透的风险。

禁用蓝牙服务

```shell
# systemctl stop bluetooth
```

## 五 系统资源控制

### 5.1 访问控制

应通过设定终端接入方式、网络地址范围等条件限制终端登录。

对接入服务器的IP、方式等进行限制，可以阻止非法入侵。

* 在/etc/hosts.allow和/etc/hosts.deny文件中配置接入限制

最好的策略就是阻止所有的主机在“/etc/hosts.deny”文件中加入“ ALL:ALL@ALL, PARANOID ”，然后再在“/etc/hosts.allow” 文件中加入所有允许访问的主机列表。如下操作：

编辑 hosts.deny文件（vi /etc/hosts.deny），加入下面该行：

```
# Deny access to everyone. 
ALL: ALL@ALL, PARANOID
```



编辑hosts.allow 文件（vi /etc/hosts.allow），加入允许访问的主机列表，比如：

ftp: 202.54.15.99 foo.com    //202.54.15.99是允许访问 ftp 服务的 IP 地址

//foo.com 是允许访问 ftp 服务的主机名称。

* 也可以用iptables进行访问控制 

### 5.2 **超时锁定**

应根据安全策略设置登录终端的操作超时锁定。

设置登录超时时间，释放系统资源，也提高服务器的安全性。

/etc/profile中添加如下一行

```
exprot TMOUT=900  //15分钟# source /etc/profile
```

改变这项设置后，必须先注销用户，再用该用户登录才能激活这个功能。

如果有需要，开启屏幕保护功能

设置屏幕保护：设置 -> 系统设置 -> 屏幕保护程序，进行操作

## 六  最佳经验实践 

对Linux系统的安全性提升有一定帮助。

### 6.1 DOS攻击防御

防止拒绝服务攻击

TCP SYN保护机制等设置

* 打开 syncookie：

```shell
# echo“1”>/proc/sys/net/ipv4/tcp_syncookies  //默认为1，一般不用设置
```

表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；

* 防syn 攻击优化

用vi编辑/etc/sysctl.conf，添加如下行：

```shell
net.ipv4.tcp_max_syn_backlog = 2048
```

### 6.2 历史命令

为历史的命令增加登录的IP地址、执行命令时间等

* 保存1万条命令

```
# sed -i 's/^HISTSIZE=1000/HISTSIZE=10000/g' /etc/profile
```

* 在/etc/profile的文件尾部添加如下行数配置信息：

```
USER_IP=`who -u am i 2>/dev/null| awk '{print $NF}' |sed -e 's/[()]//g'`if [ "$USER_IP" = "" ]thenUSER_IP=`hostname`fiexport HISTTIMEFORMAT="%F %T $USER_IP `whoami` "shopt -s histappendexport PROMPT_COMMAND="history -a"
```

## 参考链接

* https://mp.weixin.qq.com/s/Sq1Kdl6OfIR_0VM7IXCHOA