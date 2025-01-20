---
title: 解决Infra solr迁移与分片问题
date: 2020-12-02 00:00:00 +0800
categories: [大数据]
tags: [故障处理]
---

# 【故障处理】解决Infra solr迁移与分片问题

清理Infra Solr collection及分片问题
可以参考https://lucene.apache.org/solr/guide/6_6/collections-api.html
与https://gist.github.com/gzhelyaz/6e6ae31fe3c1b19688d61529323e056d
利用api来删除collection。

## Short Description:
Clean up the ranger atlas collections and recreate them from ambari infra solr

## Article
How to clean up &amp; recreate collections on Ambari Infra

Sometimes you need to clean up an installation that went with issues and there are missing collections on Ambari Infra solr.

In order to do this, go thry the next list of steps to clean up solr collections and recreate them from scratch.



__IMPORTANT: This steps will delete all collection data, so unless you are sure data can be removed dont perform these steps.__

>1. Switch adudit solr to OFF from Ranger -> Ranger Audit configuration. Restarted all affected.
>2. Stop Altas service
>3. From the ambari infra host remove all the collections using rest api

>`$ kinit -kt /etc/security/keytabs/ambari-infra-solr.service.keytab infra-solr/....` 
>`$ export SOLR_HOST=<solr host fqdn>` 
>## Ranger collection:
`$ curl -i -v --negotiate -u : "http://$SOLR_HOST:8886/solr/admin/collections?action=DELETE&amp;name=ranger_audits"` 
## Atlas collections:
`$ curl -i -v --negotiate -u : "http://$SOLR_HOST:8886/solr/admin/collections?action=DELETE&amp;name=vertex_index"`
`$ curl -i -v --negotiate -u : "http://$SOLR_HOST:8886/solr/admin/collections?action=DELETE&amp;name=edge_index"`
`$ curl -i -v --negotiate -u : "http://$SOLR_HOST:8886/solr/admin/collections?action=DELETE&amp;name=fulltext_index"`
4) Stop the Ambari Infra
5) Remove the /infra-solr znode
__*IMPORTANT: This next command will remove all the configuration of all the collections on zookeeper for infra-solr.__

`$ zookeeper-client`
`rmr /infra-solr`

6) Start Ambari Infra again (once started the znode /infra-solr should be created again)
7) Switch adudit solr back ON from Ranger -> Ranger Audit configuration
8) Restart all affected
9) Still need to restart all Ranger components one more time
10) Start Atlas
11) After all services come up on zookeeper you should see the following znodes have been created: `[vertex_index, edge_index, fulltext_index, ranger_audits]`