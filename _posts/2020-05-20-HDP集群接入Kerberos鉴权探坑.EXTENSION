---
title: HDP集群接入Kerberos鉴权探坑
date: 2020-05-20 20:15:03 +0800
categories: [大数据, 鉴权]
tags: [kerberos]
---

为了保证集群的运行安全，需要将部署的集群接入kerberos，此处kerberos是内建再域控AD内的，ambari直接使用服务，无需额外部署。因为kerberos的特殊性，集群接入时注意的要点相当多，在我们为数不多的两次实践中，确实踩了很多坑。这里整理这篇文章，一来本着救死扶伤的精神，让后来的同学能够躲过这些坑的摧残，二来固化记忆，未来需要进行相关配置的时候能有个参照，三来就是抛砖引玉，希望不对的地方能够得到指正，缺乏描述之处也能有所补充。


1.AD准备
因为集群内kerberos Client接入的是AD，因此前期的配置需要在AD上进行，这里可以参考[IBM的一篇文章](https://www.ibm.com/docs/en/spectrum-scale-bda?topic=support-enable-kerberos-existing-active-directory)。这篇讲了从创建OU到配置读写权限账号等AD内的操作。重点是交予Ambari的OU管理员账号，需要对OU有创建及读写权限，以便自动创建集群各服务所需的认证账号。这一步有问题的话，Ambari接入Kerberos的测试会不通过，提示管理员账号无权限创建用户。


2.证书配置
因为与AD通信走的是SSL，需要将SSL证书导入ambari中。AD的证书是公司内部CA自己签发的证书，可以应用于prd.com等域名的认证。从AD获取根证书，然后导入到ambari中去。这里我们同时使用了两种方式导入证书，一种是直接将证书导入java的cacerts中（参见上点的超链接内相关内容），另一种是创建新的keystore，然后利用amabri-server的setup-security配置工具来导入证书（参见[这篇文档](https://docs.cloudera.com/HDPDocuments/Ambari-2.7.5.0/administering-ambari-views/content/amb_configure_a_trust_store_for_the_ambari_views_server.html)，以及[这篇问答](https://community.cloudera.com/t5/Support-Questions/Failed-to-connect-to-KDC-Failed-to-communicate-the-Active/td-p/218270)）。因为这里遇到了阻碍，因此两个步骤都执行了，目前不清楚哪个真正有效。


3.hosts配置
HDP集群搭建时，ambari推荐为各节点配置FQDN（Fully Qualified Domain Name），以保证节点的服务发现和通信正常。kerberos的认证也需要FQDN的支持，但是在集群配置之初，集群并未配置合理的FQDN，因此这部分也进行了修改，具体步骤可以参考[[为HDP集群机器更改hostname]]这篇文章。另外，在/etc/hosts内，要配置上各节点的IP，机器名以及FQDN。


4.域名解析要求
在Ambari kerberos wizard中进行配置的时候，直接kdc host一开始填写的是AD的IP，连接正常，但是在验证时还是报SSL证书找不到的错误，这一点困扰了我们一些时间。在集群机器上curl访问AD的时候，发现有域名和ssl证书不匹配的报错，联想到SSL开启后，受认证的通信必须由域名来接入，直接在kdc host内填写IP会导致类似的错误，只不过在ambari内没有正确的提示出来，这时证书是正确导入的，只不过证书域名不匹配而被ambari-server忽略了。
解决方法很简单，在Ambari kerberos wizard配置时，kdc host以及相关配置填写AD的域名就可以了。这里我们集群没办法正常解析AD的域名，于是选择在/etc/hosts里添加配置来解析。


5.UDP vs TCP
到这里Kerberos终于验证成功开始安装了，但是安装完毕启动组件的时候，zookeeper却出现了认证不成功的问题。这里仔细再网上搜索了各种问答，主要集中在keytab的加密方式，以及UDP通信配置两方面，在对比了之前生产环境的配置后，锁定了关闭UDP通信的配置项。kerberos库在鉴权的时候，对于信息不大的请求采用UDP的形式发送，而超过`udp_preference_limit`规定大小的信息才采用TCP方式发送，虽然UDP不通时也会采用TCP形式，这点可能造成zookeeper认证失败，而不能启动组件。集群内网络配置可能也限制了UDP的请求。在添加了`udp_preference_limit = 1`这行配置到krb5.conf的`[libdefaults]`模块后，zookeeper可以正常启动了。这个配置相当于默认所有请求走TCP。


6.重启再重启
Zookeeper的小坑跨过去了，Amabri-server获取集群jmx信息的时候却也出现了认证失败的问题，在确认各服务配置正常的情况下，考虑是ambari-server的问题，在kerberos配置成功后没有重启Ambari-server，重启直接解决。