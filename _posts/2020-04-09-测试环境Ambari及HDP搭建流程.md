---
title: 测试环境Ambari及HDP搭建流程
date: 2020-04-09 00:00:00 +0800
categories: [大数据, 运维]
tags: [hdp]
---

# 测试环境Ambari及HDP搭建流程

## 参考文章
可以参考[CSDN的文章](https://blog.csdn.net/Happy_Sunshine_Boy/article/details/86595945)
也可以参考[官方文档](https://docs.cloudera.com/HDPDocuments/Ambari-2.7.3.0/bk_ambari-installation/content/ch_Getting_Ready.html)


## 所需软件包下载地址
Aambari-2.7.3.0：
http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.3.0/ambari-2.7.3.0-centos7.tar.gz

HDP-3.1.0：
http://public-repo-1.hortonworks.com/HDP/centos7/3.x/updates/3.1.0.0/HDP-3.1.0.0-centos7-rpm.tar.gz

HDP-UTILS-1.1.0.22：
http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/centos7/HDP-UTILS-1.1.0.22-centos7.tar.gz

另外根据[这篇文章](https://docs.cloudera.com/HDPDocuments/Ambari-2.7.3.0/bk_ambari-installation/content/install-mysql.html )最好选择MySQL5.5之后的版本作为Ambari及各组件的元数据库或者直接使用云数据库，而不是使用内部默认的Postgres数据库。本地安装的话，最好准备好rpm bundle包，而不是使用Oracle的源来安装，因为网速往往很慢很慢。


## 机器准备
机器准备的要点包括：
1. 统一hostname，可以使用`hostnamectl`命令来为集群设定有意义的统一格式的hostname，方便识别机器。
2. 主机向集群内的从机配置免密，使用`ssh-keygen`生成公私钥，将公钥的`id-rsa.pub`导入`authorized_keys`里，再使用`scp`或者`ssh-copy-id`拷贝至各从机。
3. 集群内所有机器开启NTP服务，确保机器时间一致。我司的机器默认安装了ntp包，不过需要手动开启ntpd，并配置开机自启动。
4. 修改所有机器的/etc/hosts，需要包涵有效的FQDN。
5. 在 `/etc/sysconfig/network` 内配置有效的FQDN。
6. 因为内网机器的网络权限由网络的同学管理，从其他机器复制网络权限要求即可。
7. 集群机器默认禁止SELinux，因此这一步不用处理。
8. 检查`Process`和`max open file`的限制。
```
 `ulimit -Sn` 
 `ulimit -Hn` 
```如果限制过小的话，修改成集群统一的100001，可以在`/etc/limits.conf`，或者`/etc/limits.d/`里面修改


## 安装本地MySQL库（可选）
1. 使用scp将oracle的rpm bundle包拷贝到目标机器上，使用`rpm -ivh`命令按照正确顺序安装所需包，可以参考[这篇文章](https://blog.csdn.net/xiangxianghehe/article/details/77427077)
2. 配置好mysql-connection-java：`yum install mysql-connector-java`. 安装好的包在`/usr/share/java`路径下。
3. 配置mysql，添加utf8编码，并重启服务。


## 在数据库创建相应的用户和DB
1. 需要创建ambari，hive，ranger等所需组件的数据库及专属用户。可以选择先创建ambari的，后续数据库在ambari的webUI提供root及密码ambari自动创建（root需要有远程登陆权限）。
2. 确保用户有对应数据库的权限并可以远程登陆。
```
mysql> create database ambari character set utf8;
mysql> CREATE USER 'ambari'@'%'IDENTIFIED BY 'sometoughpass';
mysql> GRANT ALL PRIVILEGES ON ambari.* TO 'ambari'@'%';
mysql> FLUSH PRIVILEGES;
```


## 配置本地源
1. 安装yum相关工具
```
[root@yum ~]# yum install yum-utils -y
[root@yum ~]# yum repolist
[root@yum ~]# yum install createrepo -y
[root@yum ~]# yum install httpd -y
```
2. 上传下载好的ambari及hdp包。
3. 在`/var/www/html`目录下配置ambari和hdp的目录，来存放安装文件。
```
[root@yum ~]# mkdir /var/www/html/ambari
[root@yum ~]# mkdir /var/www/html/hdp
[root@yum ~]# mkdir /var/www/html/hdp/HDP-UTILS-1.1.0.22
[root@yum ~]# tar -zxvf ambari-2.7.3.0-centos7.tar.gz -C /var/www/html/ambari/
[root@yum ~]# tar -zxvf HDP-3.1.0.0-centos7-rpm.tar.gz -C /var/www/html/hdp/
[root@yum ~]# tar -zxvf HDP-UTILS-1.1.0.22-centos7.tar.gz -C /var/www/html/hdp/HDP-UTILS-1.1.0.22/
```
4. 启动httpd服务
```
[root@yum ~]# systemctl start httpd		# 启动httpd
[root@yum ~]# systemctl status httpd		# 查看httpd状态
[root@yum ~]# systemctl enable httpd	# 设置httpd开机自启
```
5. 在默认的80端口查看apache http server是否正常运行。
6. 配置ambari的本地repo，可以先下载官方的repo配置。（注意版本）
```
wget -O /etc/yum.repos.d/ambari.repo http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.3.0/ambari.repo
```
7. 修改配置文件：`vim /etc/yum.repos.d/ambari.repo`，将baseurl和gpgkey修改为httpd对应目录下的url。
8. 配置HDP和HDP-UTILS的repo。在`/etc/yum.repos.d/`下添加HDP.repo文件。添加以下内容（注意版本及baseurl和gpgkey的配置）。
```
# VERSION_NUMBER=3.1.0.0-78
[HDP-3.1.0.0]
name=HDP Version - HDP-3.1.0.0
baseurl=http://192.168.121.77/hdp/HDP/centos7
gpgcheck=1
gpgkey=http://192.168.121.77/hdp/HDP/centos7/3.1.0.0-78/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1

[HDP-UTILS-1.1.0.22]
name=HDP-UTILS Version - HDP-UTILS-1.1.0.22
baseurl=http://192.168.121.77/hdp/HDP-UTILS-1.1.0.22
gpgcheck=1
gpgkey=http://192.168.121.77/hdp/HDP-UTILS-1.1.0.22/HDP-UTILS/centos7/1.1.0.22/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
```
9. 将ambari.repo及HDP.repo分发到各节点的相同目录下。
10. 使用`createrepo`命令生成`/var/www/html/hdp/HDP/centos7/`及`/var/www/html/hdp/HDP-UTILS-1.1.0.22/`下的本地源，自动生成元数据。


## 安装配置Ambari-server
1. yum安装ambari-server，如有报错，可以检查下repo文件内的路径是否配置正确。安装完毕后配置ambari-server。
```
[root@hdptest ~]# yum install ambari-server
[root@hdptest ~]# ambari-server setup
```
2. 配置ambari-server：
```
[root@nd-00 ~]# yum install ambari-server
[root@nd-00 ~]# ambari-server setup
Using python  /usr/bin/python
Setup ambari-server
Checking SELinux...
SELinux status is 'disabled'
Customize user account for ambari-server daemon [y/n](n)? y
Enter user account for ambari-server daemon (root):root		# 用户
Adjusting ambari-server permissions and ownership...
Checking firewall status...
Checking JDK...
[1] Oracle JDK 1.8   Java Cryptography Extension (JCE) Policy Files 8
[2] Custom JDK
==============================================================================
Enter choice (1): 2		# 选择自定义jdk
WARNING: JDK must be installed on all hosts and JAVA_HOME must be valid on all hosts.
WARNING: JCE Policy files are required for configuring Kerberos security. If you plan to use Kerberos,please make sure JCE Unlimited Strength Jurisdiction Policy Files are valid on all hosts.
Path to JAVA_HOME: /usr/java/jdk1.8.0_77		# jdk安装路径
Validating JDK on Ambari Server...done.
Check JDK version for Ambari Server...
JDK version found: 8
Minimum JDK version is 8 for Ambari. Skipping to setup different JDK for Ambari Server.
Checking GPL software agreement...
GPL License for LZO: https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html
Enable Ambari Server to download and install GPL Licensed LZO packages [y/n](n)? y
Completing setup...
Configuring database...
Enter advanced database configuration [y/n](n)? y
Configuring database...
==============================================================================
Choose one of the following options:
[1] - PostgreSQL (Embedded)
[2] - Oracle
[3] - MySQL / MariaDB
[4] - PostgreSQL
[5] - Microsoft SQL Server (Tech Preview)
[6] - SQL Anywhere
[7] - BDB
==============================================================================
Enter choice (1): 3		# 选择安装的mysql
Hostname (localhost): nd-00		# 配置hostname
Port (3306): 		# 默认
Database name (ambari): 
Username (ambari): 
Enter Database Password (bigdata): 		# 密码不显示
Re-enter password: 
Configuring ambari database...
Should ambari use existing default jdbc /usr/share/java/mysql-connector-java.jar [y/n](y)? y
Configuring remote database connection properties...
WARNING: Before starting Ambari Server, you must run the following DDL directly from the database shell to create the schema: /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql		# 此处需注意，启动ambari之前需要执行此句
Proceed with configuring remote database connection properties [y/n](y)? y
Extracting system views...
ambari-admin-2.7.3.0.139.jar
....
Ambari repo file contains latest json url http://public-repo-1.hortonworks.com/HDP/hdp_urlinfo.json, updating stacks repoinfos with it...
Adjusting ambari-server permissions and ownership...
Ambari Server 'setup' completed successfully.		# 安装成功
```
这里用mac配置的时候有个小bug，输入delete会专成^H，上下左右也会变成字符，因此只能先在其他地方输入好再复制上去。

3. 在mysql内执行ambari-server安装过程中提示的sql文件。
```
[root@nd-00 ~]# mysql -u ambari -p -h nd-00
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.25 MySQL Community Server (GPL)
Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
 -------------------- 
| Database           |
 -------------------- 
| information_schema |
| ambari             |
 -------------------- 
2 rows in set (0.03 sec)

mysql> use ambari;
Database changed
mysql> source /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql;
# 执行完，查看有无报错信息，并查看数据表
```
4. 启动Ambari-Server： `ambari-server start`
5. 在各节点安装ambari-agent：`yum -y install ambari-agent`


## Ambari界面操作
在Ambari server安装完成后，可以在对应服务器的8080端口访问WEBUI，使用默认用户名及密码admin/admin登陆Ambari界面。之后按照页面的提示新建新集群即可。