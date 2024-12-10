---
title: Ambari接入LDAP流程
date: 2024-08-12 16:09 +0800
categories: [大数据, ambari]
tags: [ambari]
---

# Ambari接入LDAP流程
## 背景
等保要求，需要周期修改 Ambari 的管理账户密码，但是当前 Ambari 的管理账户是内置的，无法实现接口自动修改密码，所以需要接入 LDAP，利用 AD 的自动密码过期机制来实现密码的周期性修改。这里记录下接入流程。

## 接入流程
Ambari Server 的 LDAP 的配置实际是存储在数据库中的，所以无法直接通过修改配置文件的方式补充 LDAP 配置，而需要走 ambari-server 的 cli daemon 命令来修改配置。实例命令和配置如下：
```shell
# 执行ambari-server setup-ldap
# 这里我们写好LDAP的Hosts，如果是域名的话，需要注意配置好DNS解析或者/etc/hosts文件
Fetching LDAP configuration from DB.
Primary LDAP Host (addnstest001.xxxx.cn):
# 走389端口，不用走SSL，走 SSL 的话还需要配置证书
Primary LDAP Port (389):

# 这里可以直接跳过Secondary LDAP 配置如果有 HA 的话，可以补充上
Secondary LDAP Host <Optional>:
Secondary LDAP Port <Optional>:
# 不用走SSL，走 SSL 的话还需要配置证书
Use SSL [true/false] (false):false
# 下面是 AD 的一些基础设置
User object class (person):person
User ID attribute (sAMAccountName):sAMAccountName
Group object class (group):group
Group name attribute (cn):cn
Group member attribute (member):member
Distinguished name attribute (distinguishedName):distinguishedName
# 这里 Search 就是同步账号的范围，需要适当设置，绑定的账号需要有这个 OU 的读权限
Search Base (OU=xxxx,DC=xxxx,DC=cn): ou=service_account,dc=xxxx,dc=cn
Referral method [follow/ignore] (follow):follow
Bind anonymously [true/false] (false):false
# 这里绑定管理账号来同步数据
Bind DN (cn=hdtest_admin,ou=hadoop_hdtest,dc=wesuretest,dc=cn):
Enter Bind DN Password:
Confirm Bind DN Password:
# 这里是同步账号的过滤条件，可以根据实际情况设置，一般保持默认即可
Handling behavior for username collisions [convert/skip] for LDAP sync (skip):
Force lower-case user names [true/false] (false):
Results from LDAP are paginated when requested [true/false] (false):
```

## 验证
配置完成后，可以通过 ambari-server sync-ldap --all -v 命令来同步 LDAP 的账号信息，然后在 Ambari 的用户管理界面可以看到同步的账号信息。

这里需要注意，如果 LDAP 的账号信息有变动，需要周期性的同步 LDAP 的账号信息。可以配置一个定时任务来执行 ambari-server sync-ldap --all -v 命令。

## 注意
LDAP 账号加入后，默认有只读登录权限，默认账户是 Active 状态，但无法访问和调整任何配置。
原local的账户是不会删除的，可以通过原有Ambari Admin账户给 ldap 同步进来的账户赋权。
local也建议保留，这样如果有脚本使用到老账号可以不用周期修改账户密码。

