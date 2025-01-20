---
title: Atlas-Plugin的升级操作记录，加入Admin-Purge权限
date: 2023-04-12 00:00:00 +0800
categories: [大数据]
tags: [ranger]
---

# 【Ranger】Atlas-Plugin的升级操作记录，加入Admin-Purge权限


# 问题：
Atlas在升级到2.1.0后，历史0.7.0版本对接的Ranger Atlas Plugin已落后多个版本，在进行Admin Purge操作时，发现老版本Ranger Plugin和对应Authorizer缺失新操作的标签，导致无法对新版本Admin Purge操作做鉴权。
检查Ranger的Atlas的Policy Json Cache `/etc/ranger/hdsz_atlas/policycache`，确认在检查3.1.0-307-6版本Ranger代码时发现 ，关键配置`/agents-common/src/main/resources/service-defs/ranger-servicedef-atlas.json`缺失了对应的操作。

# 背景说明：
Apache Ranger是一个开源的安全管理框架，旨在为Hadoop生态系统（如HDFS、Hive、HBase、Kafka等）提供细粒度的访问控制和安全性管理。它提供了集中式的安全策略管理、访问控制和审核功能，可以帮助组织管理和保护其数据资源。
Apache Ranger的主要功能包括：
1. 集中式安全策略管理：Apache Ranger提供了一个集中式的管理控制台，可用于创建、管理和部署安全策略。这些策略可以应用于不同的Hadoop组件，例如HDFS、Hive、HBase、Kafka等，当然也包括我们这次针对处理的Atlas。
2. 细粒度的访问控制：Apache Ranger允许管理员定义细粒度的访问控制策略，以控制用户或组对数据资源的访问权限。这些策略可以基于用户、组、IP地址、时间和其他条件进行定义。
3. 审计功能：Apache Ranger提供了完整的审计功能，可以记录所有访问请求和操作，以便管理员可以监视和审计用户访问行为。
4. 多租户支持：Apache Ranger可以支持多个租户，每个租户可以有自己的安全策略和访问控制规则。

总之，Apache Ranger为Hadoop生态系统提供了一个强大的安全管理框架，可以帮助组织保护其数据资源并满足合规性需求。

Ranger包含以下几个组件：
1. Ranger Admin：Ranger Admin是集中式的管理控制台，可以用于创建、管理和部署安全策略。管理员可以使用Ranger Admin来定义细粒度的访问控制策略，以控制用户或组对数据资源的访问权限，还可以进行审计和报告管理。
2. Ranger Usersync：Ranger Usersync是一个工具，用于同步Ranger Admin中的用户和组信息以及外部目录（如LDAP、Active Directory等）中的用户和组信息。它还可以帮助管理员将外部目录中的用户和组与Ranger策略中的用户和组进行映射。
3. Ranger Plugins：Ranger Plugins是一组插件，可用于将Ranger的访问控制和审计功能集成到不同的Hadoop组件中，例如HDFS、Hive、HBase、Kafka等。这些插件可以与Hadoop组件进行集成，以便管理员可以在Ranger Admin中定义访问控制策略，并将这些策略应用于各个Hadoop组件。
4. Ranger TagSync：Ranger TagSync是一个工具，用于同步Ranger Admin中的标记信息以及外部目录（如LDAP、Active Directory等）中的标记信息，以便管理员可以在Ranger Admin中定义基于标记的访问控制策略。

我们这次处理，主要是修改安全策略的元信息，经过定位，我们将目标放到了ranger-plugins-common库。ranger-plugins-common包含了一些与Hadoop生态系统组件无关的通用功能。它的作用是提供一些通用的功能和工具类，以便在Ranger插件中进行共享和重用。具体来说，ranger-plugins-common的jar包中包含以下几个方面的内容：
1. 安全相关的工具类：例如密码加密和解密工具类、SSL证书相关的工具类等。
2. 与策略相关的数据模型和类：例如Ranger策略的数据模型、策略解析器等。
3. 与审计相关的类和接口：例如审计记录的数据模型、审计记录器接口等。
4. 与插件开发相关的类和接口：例如插件注册器接口、插件工厂接口等。

除了上述类和接口，各plugins的serviceDef也打包放在了ranger-plugins-common库内。serviceDef是指定义Hadoop生态系统中组件的安全策略的元数据。每个Hadoop组件都有自己的serviceDef，它定义了该组件所支持的安全性策略和相应的属性。例如，HDFS组件的serviceDef定义了HDFS中文件和目录的访问控制策略，Hive组件的serviceDef定义了对Hive表的访问控制策略等等。

在ranger-plugins-common模块中，serviceDef的定义被打包在一个json文件中。这个文件包含了每个组件的serviceDef定义的元数据信息，例如组件名、组件类型、支持的访问控制类型、支持的资源类型、支持的属性等等。


# 撞墙确认不通的逻辑：
1. 尝试直接修改policy cache，但发现并不生效，且每次Ranger下发策略时，都会覆盖Cache内的policy文件。
2. 直接修改`/agents-common/src/main/resources/service-defs/ranger-servicedef-atlas.json`，完成编译后，替换关键组件的ranger-plugins-common-1.2.0.3.1.0.0-78.jar，发现并不生效。因为servicedef同时也会被记录到ranger元数据DB中，并进一步加载到内存。历史ServiceDef没有处理的情况下，前端显示也存在问题。

# 继续深挖
追踪git上`/agents-common/src/main/resources/service-defs/ranger-servicedef-atlas.json`的变更记录，admin-purge的变更其实涉及了多个issue，最终，我们锁定到了https://issues.apache.org/jira/browse/RANGER-2821 这个更新。在Review的记录（https://reviews.apache.org/r/72621/ ）中可以看到，Patch的Committer共变动了两个模块，一个更改了`ranger-servicedef-atlas.json`，另外一个则包含了Ranger的Java Patch逻辑，用来处理历史ServiceDef造成的问题。但是Java Patch的逻辑跟另外一个Patch存在逻辑上的重复，因此没有发到生产。我们直接将这个Java Patch从Diff文件中导出来，在源码中创建了新的class，完成了patch的编译。

# Java Patch应用
拿到了Patch的class文件并不能解决问题，要搞清楚，这个Java Patch是在何时打入Ranger Admin组件的。
首先我们确认了Java Patch放置的位置，`/ranger-admin/ews/webapp/WEB-INF/classes/org/apache/ranger/patch/` 可以看到，生产部署后，patch的class文件都被放到ranger-admin的webapp中。
在继续探索的过程中，我们看了社区内的一些issue，比如https://community.cloudera.com/t5/Community-Articles/RANGER-Policies-are-not-visible-disappeared-after-HDP/ta-p/245888 进一步确认了Ranger组件在安装过程中，以及启动过程中，都会由Ambari触发db_setup.py逻辑，在这个过程中利用patch function进行了打补丁操作。

``` 
Execute['ambari-python-wrap /usr/hdp/current/ranger-admin/db_setup.py'] {'logoutput': True, 'environment': {'PATH': '/usr/sbin:/sbin:/usr/lib/ambari-server/*:/usr/sbin:/sbin:/usr/lib/ambari-server/*:/usr/lib64/qt-3.3/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/var/lib/ambari-agent:/var/lib/ambari-agent', 'RANGER_ADMIN_HOME': u'/usr/hdp/current/ranger-admin', 'JAVA_HOME': u'/usr/local/jdk1.8.0_101'}, 'user': 'ranger'}
```

`db_setup.py`在main逻辑中，会检查JAVA_PATCHES为名的补丁记录，如果在ranger元数据库中的x_db_version_h表中已有激活记录，便不会触发新的打补丁操作，如果没有JAVA_PATCHES的记录，则会重新打一遍补丁。检查Ambari启动Ranger的日志，印证了脚本逻辑。因为x_db_version_h表中已有安装过程中引入的补丁记录，因此不会再进行打补丁操作。

``` 
if str(argv[i]) == "-javapatch":
    applyJavaPatches=xa_sqlObj.hasPendingPatches(db_name, db_user, db_password, "JAVA_PATCHES")
    if applyJavaPatches == True:
        log("[I] ----------------- Applying java patches ------------", "info")
        xa_sqlObj.execute_java_patches(xa_db_host, db_user, db_password, db_name)
        xa_sqlObj.update_applied_patches_status(db_name,db_user, db_password,"JAVA_PATCHES")
    else:
       log("[I] JAVA_PATCHES have already been applied","info")       
```

有了以上信息，整个Java Patch的应用流程就比较清晰了，我们只需改造`db_setup.py`脚本，重新触发`RANGER-2821`提供的`J10039`Java Patch的补丁逻辑即可。

最终处理流程：
1. 先变更`/agents-common/src/main/resources/service-defs/ranger-servicedef-atlas.json`，编译项目，获取更新的ranger-plugins-common-1.2.0.3.1.0.0-78.jar，并替换关联组件内的jar。
替换Ranger Admin，以及Atlas Meta Server所在节点上的/usr/hdp/3.1.0.0-78路径下的这些文件：
`./atlas/libext/ranger-atlas-plugin-impl/ranger-plugins-common-1.2.0.3.1.0.0-78.jar`
`./ranger-tagsync/lib/ranger-plugins-common-1.2.0.3.1.0.0-78.jar`
`./ranger-atlas-plugin/lib/ranger-atlas-plugin-impl/ranger-plugins-common-1.2.0.3.1.0.0-78.jar`
`./ranger-kms/ews/webapp/WEB-INF/classes/lib/ranger-plugins-common-1.2.0.3.1.0.0-78.jar`
`./ranger-kms/ews/webapp/WEB-INF/classes/lib/ranger-kms-plugin-impl/ranger-plugins-common-1.2.0.3.1.0.0-78.jar`
`./ranger-admin/ews/lib/ranger-plugins-common-1.2.0.3.1.0.0-78.jar`
`./ranger-admin/ews/webapp/WEB-INF/lib/ranger-plugins-common-1.2.0.3.1.0.0-78.jar`
`./ranger-usersync/lib/ranger-plugins-common-1.2.0.3.1.0.0-78.jar`

2. 将编译完成的`PatchForAtlasToAddAdminPurge_J10039.class`放入ranger admin所在节点的`/usr/hdp/3.1.0.0-78/ranger-admin/ews/webapp/WEB-INF/classes/org/apache/ranger/patch/` 路径下。

3. 改造`db_setup.py`脚本：
``` 
if str(argv[i]) == "-javapatch":
    applyJavaPatches=xa_sqlObj.hasPendingPatches(db_name, db_user, db_password, "J10039")
    if applyJavaPatches == True:
        log("[I] ----------------- Applying java patches ------------", "info")
        xa_sqlObj.execute_java_patches(xa_db_host, db_user, db_password, db_name)
        xa_sqlObj.update_applied_patches_status(db_name,db_user, db_password,"J10039")
    else:
       log("[I] J10039 have already been applied","info")       
```

4. 重启Ranger-Admin组件，触发打Java Patch逻辑，观察输出

``` 
2023-04-11 19:19:00,798  [I] java patch PatchForAtlasToAddAdminPurge_J10039 is being applied..
2023-04-11 19:19:11,881  [JISQL] /usr/local/jdk1.8.0_101/bin/java  -cp /usr/hdp/current/ranger-admin/ews/lib/mysql-connector-java.jar:/usr/hdp/current/ranger-admin/jisql/lib/* org.apache.util.sql.Jisql -driver mysqlconj -cstring jdbc:mysql://hdtest01.dev.com/ranger -u 'rangeradmin' -p '********' -noheader -trim -c \; -query "update x_db_version_h set active='Y' where version='J10039' and active='N' and updated_by='hdtest02.dev.com';"
2023-04-11 19:19:12,841  [I] java patch PatchForAtlasToAddAdminPurge_J10039 is applied..
`2023-04-11 19:19:12,842  [JISQL] /usr/local/jdk1.8.0_101/bin/java  -cp /usr/hdp/current/ranger-admin/ews/lib/mysql-connector-java.jar:/usr/hdp/current/ranger-admin/jisql/lib/* org.apache.util.sql.Jisql -driver mysqlconj -cstring jdbc:mysql://hdtest01.dev.com/ranger -u 'rangeradmin' -p '********' -noheader -trim -c \; -query "select version from x_db_version_h where version = 'J10039' and inst_by = 'Ranger 1.2.0.3.1.0.0-78' and active='Y';"`
```

5. 进入Ranger服务，检查Atlas Service Policy是否正常显示Admin Purge选项。
![截屏2023-04-12 14.04.16.png](/assets/media/RangerAtlas-Plugin的升级操作记录，加入Admin-Purge权限/tapd_48728548_1681279472_673.png)
6. 检查Atlas的service cache，检查是否存在新增Admin Purge操作
![截屏2023-04-12 14.05.45.png](/assets/media/RangerAtlas-Plugin的升级操作记录，加入Admin-Purge权限/tapd_48728548_1681279558_495.png)
7. 尝试Atlas的Purge操作，确认脚本可执行。

``` 
// 从Atlas的ES索引中获取待清理entity的GUID
curl -XGET -H'Content-Type: application/json' -u 'atlas:123456' 'mpes.sit.com:9200/janusgraph_vertex_index/_search?pretty' -d \
'{
"from": 0,
"size" : 5,
"query": {
  "bool": {
       "must": [
                  { "match": { "__typeName": "hive_table" } },
                  { "match": { "__state": "DELETED" } },
                  { "regexp": { "Referenceable.qualifiedName": ".*bi_analysis_platform_.*" } }
               ]
          }
         }
}'

// 调用purge操作清理test_clean.json内对应GUID的entity
curl -u admin:123456 -i -H "Content-Type: application/json" -X PUT  'http://hdtestcom01.dev.com:21000/api/atlas/admin/purge/' -d @test_clean.json

// 检查返回结果正常
{"mutatedEntities":{"PURGE":[{"typeName":"hive_column","attributes":{"owner":"service_data_studio","qualifiedName":"tmp_db.bi_analysis_platform_event_portrait_1f887578568c11ed_e04fe4c8d7ae4cda94061372ba39d732_0_0._c3@hdtest","name":"_c3"},"guid":"16c7bd92-d00c-4020-9cb4-d63ee69397ad","status":"DELETED","displayText":"_c3","classificationNames":[],"classifications":[],"meaningNames":[],"meanings":[],"isIncomplete":false,"labels":[]},{"typeName":"hive_column","attributes":{"owner":"hdtest","qualifiedName":"tmp_db.bi_analysis_platform_c15854712esa011ed_step1.of_base_area_main_province__gprofile_t_adm_user_feature_wide_complet_dd_user_id_user_id@hdtest"...
```