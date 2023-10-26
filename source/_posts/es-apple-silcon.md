---
title: 安装 ElasticSearch 7.17.4 与 Logstash / Kibana 到 Apple Silcon (M1 Arm) macOS
tags: macOS, ElasticSearch, Logstash, Kibana
date: 2023-10-11 00:44:33
---


由于软件协议问题， brew 官方删除了 es 的源，改为 es 官方自行维护的 tap 。另外，这些 tap 默认都使用包自带的 jdk ，然后会由于 macOS 的权限问题无法运行，因此需要通过 brew 安装 openjdk 后手动指定环境变量。

### 安装 ElasticSearch

```bash
brew tap elastic/tap
brew install elastic/tap/elasticsearch-full
```

这个时候应该会报错：

```bash
Error: elastic/tap/elasticsearch-full: Calling plist_options is disabled! Use service.require_root instead.
Please report this issue to the elastic/tap tap (not Homebrew/brew or Homebrew/homebrew-core), or even better, submit a PR to fix it:
  /opt/homebrew/Library/Taps/elastic/homebrew-tap/Formula/elasticsearch-full.rb:68
```

不用担心，根据 [这个 issue](https://github.com/elastic/homebrew-tap/issues/146) ，只需要找到提示的这个 .rb 文件的对应行，修改并替换：

<!-- more -->

```bash
plist_options :manual => "elasticsearch"
# 修改为↓
@plist_manual = "elasticsearch"` 
```

记得重新运行 `brew install elastic/tap/elasticsearch-full` 。

同时安装 openjdk ：

```bash
brew install openjdk
sudo ln -sfn /opt/homebrew/opt/openjdk/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk.jdk
```

这个时候如果直接运行 `elasticsearch` 是会报错的。

先运行 `/usr/libexec/java_home` 找到最新的 openjdk 路径，然后：

```bash
code /opt/homebrew/Cellar/elasticsearch-full/7.17.4/homebrew.mxcl.elasticsearch-full.plist
```

在最后一个 `</dict>` 前插入：

```xml
    <key>EnvironmentVariables</key>
    <dict>
      <key>ES_JAVA_HOME</key>
      <string>/opt/homebrew/Cellar/openjdk/21/libexec/openjdk.jdk/Contents/Home</string>
    </dict>
```

记得把 `<string>` 中的路径改为你的实际 openjdk 路径。

然后执行：

```bash
echo "\nxpack.ml.enabled: false\n" >> /opt/homebrew/etc/elasticsearch/elasticsearch.yml
```

再设置一个全局的环境变量：

```bash
echo "\nexport ES_JAVA_HOME=$(/usr/libexec/java_home)\n" >> ~/.zshrc
```

这时候就可以通过 `elasticsearch` 或者 `brew services start elasticsearch-full` 来启动了。

测试一下连接：

```bash
(base) ➜  ~ curl -s http://localhost:9200
{
  "name" : "Dims-MBP-Max.lan",
  "cluster_name" : "elasticsearch_dimpurr",
  "cluster_uuid" : "1efKwx_RQFa-BqsDfbUV0Q",
  "version" : {
    "number" : "7.17.4",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "79878662c54c886ae89206c685d9f1051a9d6411",
    "build_date" : "2022-05-18T18:04:20.964345128Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

### 安装 Logstash

如法炮制：

```bash
brew install elastic/tap/logstash-full
```

然后报错：

```bash
Warning: logstash-full: logstash-oss: Calling plist_options is disabled! Use service.require_root instead.
Please report this issue to the elastic/tap tap (not Homebrew/brew or Homebrew/homebrew-core), or even better, submit a PR to fix it:
  /opt/homebrew/Library/Taps/elastic/homebrew-tap/Formula/logstash-oss.rb:43
```

同样，找到提示的 .rb 文件，修改并替换：

```bash
plist_options :manual => "logstash"
# 修改为↓
@plist_manual = "logstash"
```

记得重新运行 `brew install elastic/tap/logstash-full` 。

然后 `code /opt/homebrew/Cellar/logstash-full/7.17.4/homebrew.mxcl.logstash-full.plist` 还是增加环境变量，不过这次是：

```xml
    <key>EnvironmentVariables</key>
    <dict>
      <key>LS_JAVA_HOME</key>
        <string>/opt/homebrew/Cellar/openjdk/21/libexec/openjdk.jdk/Contents/Home</string>
    </dict>
```

然后给 zsh 也加上：

```bash
echo "\nexport LS_JAVA_HOME=$(/usr/libexec/java_home)\n" >> ~/.zshrc
```

这时候你就可以通过 `logstash` 或者 `brew services start logstash-full` 来启动了。

你也可以安装插件：`logstash-plugin install logstash-input-jdbc` 。

### 安装 Kibana

一毛一样：

```bash
brew install elastic/tap/kibana-full
```

报错省略，总之编辑对应文件：

```bash
plist_options :manual => "kibana"
# 修改为↓
@plist_manual = "kibana"
```

神奇的是 kibana 好像不需要修改 Java 环境变量，直接 `brew services start kibana-full` 或者 `kibana` 就可以了。访问 http://localhost:5601 就能看到漂亮的界面了。

### 安装中文分词

为了改善中文文本的搜索效果，您可能需要使用专门为中文设计的分词器，例如ik分词器（ik_max_word或ik_smart）。但要使用这些分词器，您需要先安装相应的Elasticsearch插件。

在 https://github.com/medcl/elasticsearch-analysis-ik 寻找对应版本的插件，然后：

```bash
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.3.0/elasticsearch-analysis-ik-6.3.0.zip
# 或者
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.17.4/elasticsearch-analysis-ik-7.17.4.zip
```

重启 es 让插件生效。重设映射：

在使用ik分词器之前，您需要重新设置body字段的映射。但请注意，更改现有字段的映射会导致该字段之前的数据丢失。您需要重新索引数据。

创建一个新的索引，例如OldIndexName_new，并为body字段设置ik_max_word分词器。

```bash
curl -X PUT "localhost:9200/OldIndexName_new" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "body": {
        "type": "text",
        "analyzer": "ik_max_word"
      },
      // ... 其他字段的映射（确保它们与旧索引相同或包含所需的更改）
    }
  }
}'
```

使用Reindex API复制数据：

从旧索引OldIndexName复制数据到新索引OldIndexName_new。

```bash
curl -X POST "localhost:9200/_reindex" -H 'Content-Type: application/json' -d'
{
  "source": {
    "index": "OldIndexName"
  },
  "dest": {
    "index": "OldIndexName_new"
  }
}
'
```

验证新索引中的数据：

确保新索引OldIndexName_new中的数据与旧索引OldIndexName中的数据一致。

```bash
curl -X GET "localhost:9200/OldIndexName_new/_search"
```

删除旧索引并将新索引重命名：

首先，删除旧的OldIndexName索引。

```bash
curl -X DELETE "localhost:9200/OldIndexName"
```

接下来，您可以将新索引的名称OldIndexName_new更改为OldIndexName，使其与旧索引的名称一致。

```bash
curl -X POST "localhost:9200/OldIndexName_new/_rename/OldIndexName"
```

执行这些操作后，body字段将使用ik_max_word分词器，同时您的数据也会保存在新的索引中。再次强调，在执行这些操作之前，建议您先备份数据，并在非生产环境中进行测试。


### 配置安全

最小安全： https://www.elastic.co/guide/en/elasticsearch/reference/7.17/security-minimal-setup.html

先停止 es 和 kibana 。编辑 `$ES_PATH_CONF/elasticsearch.yml` 文件并添加以下内容：

```yaml
xpack.security.enabled: true
```

在我们的情况下，可以使用

```bash
echo "\nxpack.security.enabled: true\n" >> /opt/homebrew/etc/elasticsearch/elasticsearch.yml
```

如果群集具有单个节点，请在 `$ES_PATH_CONF/elasticsearch.yml` 文件中添加该 discovery.type 设置并将值设置为 single-node 。此设置可确保您的节点不会无意中连接到可能正在您的网络上运行的其他集群。

```yaml
discovery.type: single-node
```

要与群集通信，必须为内置用户配置用户名。除非启用匿名访问，否则将拒绝所有不包含用户名和密码的请求。通过运行 `elasticsearch-setup-passwords` 实用程序为内置用户设置密码。

您可以针对群集中的任何节点运行该 elasticsearch-setup-passwords 实用程序。但是，您只应为整个群集运行一次此实用程序。

使用该 auto 参数将随机生成的密码输出到控制台，您可以稍后在必要时更改这些密码：

```bash
./bin/elasticsearch-setup-passwords auto
```

如果要使用自己的密码，请使用 interactive 而不是 auto 参数运行命令。使用此模式将引导您完成所有内置用户的密码配置。

```bash
./bin/elasticsearch-setup-passwords interactive
```

启用 Elasticsearch 安全功能后，用户必须使用有效的用户名和密码登录 Kibana。您需要将 Kibana 配置为使用之前创建的内置 kibana_system 用户和密码。Kibana 执行一些需要使用 kibana_system 用户的后台任务。此帐户不适用于个人用户，并且无权从浏览器登录 Kibana。相反，您将以 elastic 超级用户身份登录 Kibana。

将 elasticsearch.username 设置添加到 KIB_PATH_CONF/kibana.yml 文件并将值设置为 kibana_system 用户： (该 KIB_PATH_CONF 变量是 Kibana 配置文件的路径。如果使用归档分发版（ zip 或 tar.gz ）安装 Kibana，则变量默认为 KIB_HOME/config 。如果您使用的是软件包发行版（Debian 或 RPM），则变量默认为 /etc/kibana .)

```yaml
elasticsearch.username: "kibana_system"
```

我们的情况下：

```bash
echo "\nelasticsearch.username: \"kibana_system\"\n" >> /opt/homebrew/etc/kibana/kibana.yml
```

在安装 Kibana 的目录中(我们应该是 `/opt/homebrew/Cellar/kibana-full/7.17.4/`)，运行以下命令以创建 Kibana 密钥库并添加安全设置。创建 Kibana 密钥库：

```bash
(base) ➜  7.17.4 git:(stable) kibana-keystore create
Created Kibana keystore in /opt/homebrew/etc/kibana/kibana.keystore
```

emm, 看起来和运行目录也没啥关系。

将 kibana_system 用户的密码添加到 Kibana 密钥库：

```bash
./bin/kibana-keystore add elasticsearch.password
```

重新启动 Kibana。以 elastic 用户身份登录 Kibana。使用此超级用户帐户可以管理空间、创建新用户和分配角色。如果您在本地运行 Kibana，请转到查看 http://localhost:5601 登录页面。

运行后：

```bash
 FATAL  Error: [config validation of [elasticsearch].password]: expected value of type [string] but got [number]
```

杯具了，记得密码不能设置为纯数字。 7.x 重置密码的过程比较复杂，我使用了很暴力的直接删除 `rm -rf /opt/homebrew/etc/elasticsearch/elasticsearch.keystore` 然后重启 es 再运行设置密码，不知道不会不会有后遗症。

在Logstash的Elasticsearch输出插件中，您可以指定用户名和密码，例如：

```ruby
output {
  elasticsearch {
    hosts => ["https://your-elasticsearch-host:port"]
    index => "your-index-name"
    user => "your-username"
    password => "your-password"
    ssl => true
    cacert => "/path/to/ca.crt"  # 如果需要的话
  }
}
```

### 连接 PostgreSQL

打开 http://localhost:5601/app/fleet/integrations/postgresql-1.10.0/add-integration ，填写连接信息，然后点击 `Check Connection` ，如果成功，点击 `Save` 。

---

参考： https://gist.github.com/todgru/0ba097d63318313f12a52594217f8e2b