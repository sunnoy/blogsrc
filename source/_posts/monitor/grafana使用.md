---
title: grafana使用
date: 2018-12-12 12:14:28
tags:
- grafana
- monitor
---

# 介绍

![grafana-logo](https://qiniu.li-rui.top/grafana-logo.png)

## 数据源

grafana支持的数据源有很多
- Graphite
- InfluxDB
- OpenTSDB
- Prometheus
- Elasticsearch
- CloudWatch

<!--more-->

## Organization

grafana支持组织单位

## User

grafana可以对某一个面板进行权限控制

## Dashboard相关

Dashboard可以有Row也可以没有，Dashboard主要由Panel组成，Panel一般放在Row里面

## Query Editor

Query Editor提供了指标的查询功能，而且可以自动补全，可以对某一个指标进行多样化展示

# 安装

grafana的安装方式很多，这里使用docker安装

```bash
docker run -d --name grafana -p 3000:3000 --net="host" grafana/grafana
```

安装过后一些目录的作用

```bash
#主配置文件
/etc/grafana/grafana.ini
#数据文件
/var/lib/grafana
#home目录
/usr/share/grafana
#日志目录
/var/log/grafana
#插件目录
/var/lib/grafana/plugins
#自定义一些精细化配置的文件夹
/etc/grafana/provisioning
```

# 配置

grafana的配置有两种方式
- 界面操作
- 配置文件

界面操作较为简单，这里主要介绍使用配置文件

# 配置文件

在grafana中从程序主体到数据源再到Dashboard中Panel配置都可以从配置文件里面搞定

grafana的配置文件在/etc/grafana，该目录下文件结构如下

```bash
├── config.monitoring
├── grafana.ini
├── ldap.toml
└── provisioning
    ├── dashboards
    │   ├── Ceph\ -\ Cluster-1544005047598.json
    │   ├── dashboard.yml
    │   └── Node\ Exporter\ Full-1544005083113.json
    └── datasources
        └── datasource.yml
```

## 程序主体配置grafana.ini

程序主题的配置文件为`grafana.ini`

详细字段含义，[来源](https://www.jianshu.com/p/a21bf4462051)

```bash
app_mode：应用名称，默认是production

[path]
data：一个grafana用来存储sqlite3、临时文件、回话的地址路径
logs：grafana存储logs的路径

[server]
http_addr：监听的ip地址，，默认是0.0.0.0
http_port：监听的端口，默认是3000
protocol：http或者https，，默认是http
domain：这个设置是root_url的一部分，当你通过浏览器访问grafana时的公开的domian名称，默认是localhost
enforce_domain：如果主机的header不匹配domian，则跳转到一个正确的domain上，默认是false
root_url：这是一个web上访问grafana的全路径url，默认是%(protocol)s://%(domain)s:%(http_port)s/
router_logging：是否记录web请求日志，默认是false
cert_file：如果使用https则需要设置
cert_key：如果使用https则需要设置

[database]
grafana默认需要使用数据库存储用户和dashboard信息，默认使用sqlite3来存储，你也可以换成其他数据库
type：可以是mysql、postgres、sqlite3，默认是sqlite3
path：只是sqlite3需要，定义sqlite3的存储路径
host：只是mysql、postgres需要，默认是127.0.0.1:3306
name：grafana的数据库名称，默认是grafana
user：连接数据库的用户
password：数据库用户的密码
ssl_mode：只是postgres使用


[security]
admin_user：grafana默认的admin用户，默认是admin
admin_password：grafana admin的默认密码，默认是admin
login_remember_days：多少天内保持登录状态
secret_key：保持登录状态的签名
disable_gravatar：


[users]
allow_sign_up：是否允许普通用户登录，如果设置为false，则禁止用户登录，默认是true，则admin可以创建用户，并登录grafana
allow_org_create：如果设置为false，则禁止用户创建新组织，默认是true
auto_assign_org：当设置为true的时候，会自动的把新增用户增加到id为1的组织中，当设置为false的时候，新建用户的时候会新增一个组织
auto_assign_org_role：新建用户附加的规则，默认是Viewer，还可以是Admin、Editor


[auth.anonymous]
enabled：设置为true，则开启允许匿名访问，默认是false
org_name：为匿名用户设置组织名称
org_role：为匿名用户设置的访问规则，默认是Viewer


[auth.github]
针对github项目的，很明显
enabled = false
allow_sign_up = false
client_id = some_id
client_secret = some_secret
scopes = user:email
auth_url = https://github.com/login/oauth/authorize
token_url = https://github.com/login/oauth/access_token
api_url = https://api.github.com/user
team_ids =
allowed_domains =
allowed_organizations =


[auth.google]
针对google app的
enabled = false
allow_sign_up = false
client_id = some_client_id
client_secret = some_client_secret
scopes = https://www.googleapis.com/auth/userinfo.profile https://www.googleapis.com/auth/userinfo.email
auth_url = https://accounts.google.com/o/oauth2/auth
token_url = https://accounts.google.com/o/oauth2/token
api_url = https://www.googleapis.com/oauth2/v1/userinfo
allowed_domains =


[auth.basic]
enabled：当设置为true，则http api开启基本认证


[auth.ldap]
enabled：设置为true则开启LDAP认证，默认是false
config_file：如果开启LDAP，指定LDAP的配置文件/etc/grafana/ldap.toml


[auth.proxy]
允许你在一个HTTP反向代理上进行认证设置
enabled：默认是false
header_name：默认是X-WEBAUTH-USER
header_property：默认是个名称username
auto_sign_up：默认是true。开启自动注册，如果用户在grafana DB中不存在

[analytics]
reporting_enabled：如果设置为true，则会发送匿名使用分析到stats.grafana.org，主要用于跟踪允许实例、版本、dashboard、错误统计。默认是true
google_analytics_ua_id：使用GA进行分析，填写你的GA ID即可


[dashboards.json]
如果你有一个系统自动产生json格式的dashboard，则可以开启这个特性试试
enabled：默认是false
path：一个全路径用来包含你的json dashboard，默认是/var/lib/grafana/dashboards


[session]
provider：默认是file，值还可以是memory、mysql、postgres
provider_config：这个值的配置由provider的设置来确定，如果provider是file，则是data/xxxx路径类型，如果provider是mysql，则是user:password@tcp(127.0.0.1:3306)/database_name，如果provider是postgres，则是user=a password=b host=localhost port=5432 dbname=c sslmode=disable
cookie_name：grafana的cookie名称
cookie_secure：如果设置为true，则grafana依赖https，默认是false
session_life_time：session过期时间，默认是86400秒，24小时


以下是官方文档没有，配置文件中有的
[smtp]
enabled = false
host = localhost:25
user =
password =
cert_file =
key_file =
skip_verify = false
from_address = admin@grafana.localhost

[emails]
welcome_email_on_sign_up = false
templates_pattern = emails/*.html


[log]
mode：可以是console、file，默认是console、file，也可以设置多个，用逗号隔开
buffer_len：channel的buffer长度，默认是10000
level：可以是"Trace", "Debug", "Info", "Warn", "Error", "Critical"，默认是info

[log.console]
level：设置级别

[log.file]
level：设置级别
log_rotate：是否开启自动轮转
max_lines：单个日志文件的最大行数，默认是1000000
max_lines_shift：单个日志文件的最大大小，默认是28，表示256MB
daily_rotate：每天是否进行日志轮转，默认是true
max_days：日志过期时间，默认是7,7天后删除

```

## ldap认证配置ldap.toml

grafana支持多种认证方式来登陆

- Google OAuth
- GitHub OAuth
- Gitlab OAuth
- Generic OAuth (Okta2, BitBucket, Azure, OneLogin, Auth0)
- LDAP Authentication (OpenLDAP, ActiveDirectory, etc)

而对于ldap需要改配文件来配置

## provisioning配置

provisioning中主要用来存放dashboard和数据源配置，需要在主体配置文件中开

```bash
[paths]
#默认位置conf/provisioning，非默认需要指定
provisioning = conf/provisioning
```

### Datasources配置

这里给出配置文件介绍

```yml
apiVersion: 1

#该列表中的数据源会在grafana启动时删除
deleteDatasources:
  - name: Graphite
    orgId: 1

#数据源列表
datasources:
  
- name: Graphite
  #数据源的类型，有Prometheus，graphite等可选
  type: graphite
  #根据不同的数据源类型来选取
  access: proxy
  # <int> org id. will default to orgId 1 if not specified
  orgId: 1
  # <string> url
  url: http://localhost:8080
  # <string> database password, if used
  password:
  # <string> database user, if used
  user:
  # <string> database name, if used
  database:
  # <bool> enable/disable basic auth
  basicAuth:
  # <string> basic auth username
  basicAuthUser:
  # <string> basic auth password
  basicAuthPassword:
  # <bool> enable/disable with credentials headers
  withCredentials:
  # <bool> mark as default datasource. Max one per org
  isDefault:
  # <map> fields that will be converted to json and stored in jsonData
  jsonData:
     graphiteVersion: "1.1"
     tlsAuth: true
     tlsAuthWithCACert: true
  # <string> json object of data that will be encrypted.
  secureJsonData:
    tlsCACert: "..."
    tlsClientCert: "..."
    tlsClientKey: "..."
  version: 1
  # <bool> allow users to edit datasources from the UI.
  editable: false
```

Prometheus数据源配置

```yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
```

### dashboard配置

文件dashboard.yml,dashboards是支持动态加载的，该配置文件会将Prometheus作为数据源来根据/etc/grafana/provisioning/dashboards内的json文件来创建dashboard

disableDeletion字段来控制当删除一个json文件时grafana是否去数据库中删除该dashboard

```yml
apiVersion: 1
providers:
- name: 'Prometheus'
  orgId: 1
  folder: ''
  type: file
  disableDeletion: false
  editable: true
  options:
    path: /etc/grafana/provisioning/dashboards
```

## docker中的环境变量

conf/grafana.ini中的配置都可以使用docker的环境变量设置，不过需要相应的语法`GF_<SectionName>_<KeyName>`

比如conf/grafana.ini中的配置

```bash
# default section
instance_name = ${HOSTNAME}

[security]
admin_user = admin

[auth.google]
client_secret = 0ldS3cretKey
```

环境变量可写

```bash
GF_DEFAULT_INSTANCE_NAME=my-instance
GF_SECURITY_ADMIN_USER=true
GF_AUTH_GOOGLE_CLIENT_SECRET=newS3cretKey
```

# docke-compose

```yml
  grafana:
    image: grafana/grafana:5.4.0
    container_name: grafana
    ports:
      - 3000:3000
    volumes:
      - /data/monitor/graf-conf:/etc/grafana
      - /data/monitor/graf-data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: "1qazxsw2#"
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_NAME: "Main Org."
      GF_AUTH_ANONYMOUS_ORG_ROLE: "Viewer"
      GF_PATHS_PROVISIONING: "/etc/grafana/provisioning"
    user: root
    restart: always
```

# 与Prometheus结合

除了前面提到的数据源配置，我们再来看一下如何将Prometheus中的标签等作为grafana的变量使用

## 支持的变量

grafana支持的变量主要有：

```bash
label_values(label)	
label_values(metric, label)	specified metric.
metrics(metric)	
query_result(query)	
```

#添加变量

在dashboard中点击右上角的齿轮设置

![变量](https://qiniu.li-rui.top/变量.png)

点击新加变量后，可如下填写，最下方会显示该变量的取值列表

![var2](https://qiniu.li-rui.top/var2.png)


