---
title: Apache使用
date: 2018-11-04 12:12:28
tags:
- Apache
---
# Apache使用

### 1. 源码安装脚本

```
wget http://git.centos8.com/lirui/nor_service/raw/master/apache/httpd2.4.sh && bash httpd2.4.sh
```
 <!--more-->

### 2. NGINX目录及配置
脚本执行完成后，自动创建应用以及配置目录
#### 2.1 应用目录
- **apache依赖安装位置/usr/local/apached**
- **apache安装位置/usr/local/apached/apache24**
- **启动二进制文件/usr/local/apached/apache24/bin/httpd**



#### 2.2 配置

- **主配置文件/etc/httpd/httpd.conf**
- **/usr/local/apached/目录结构**

``` bash
.
├── bin
├── build
├── cgi-bin
├── error
├── htdocs
├── icons
├── include
├── logs
├── man
├── manual
└── modules


```
#### 2.3 配置详解
##### 2.3.1 主配置文件模块注释
```bash


#站点主目录
ServerRoot "/usr/local/apached/apache24"
#监听端口
Listen 80

#####################################apache认证系列##########################################
#使用text文件进行认证
#LoadModule authn_file_module modules/mod_authn_file.so
#使用DBM 文件进行认证
#LoadModule authn_dbm_module modules/mod_authn_dbm.so
#匿名用户访问认证区域
#LoadModule authn_anon_module modules/mod_authn_anon.so
#使用sql数据库认证
#LoadModule authn_dbd_module modules/mod_authn_dbd.so
#缓存认证凭证，减轻服务器负载
#LoadModule authn_socache_module modules/mod_authn_socache.so
#
#LoadModule authn_core_module modules/mod_authn_core.so
#LoadModule authz_host_module modules/mod_authz_host.so
#LoadModule authz_groupfile_module modules/mod_authz_groupfile.so
#LoadModule authz_user_module modules/mod_authz_user.so
#LoadModule authz_dbm_module modules/mod_authz_dbm.so
#LoadModule authz_owner_module modules/mod_authz_owner.so
#LoadModule authz_dbd_module modules/mod_authz_dbd.so
#LoadModule authz_core_module modules/mod_authz_core.so
#LoadModule access_compat_module modules/mod_access_compat.so
#LoadModule auth_basic_module modules/mod_auth_basic.so
#LoadModule auth_form_module modules/mod_auth_form.so
#LoadModule auth_digest_module modules/mod_auth_digest.so




############################################缓存系列##########################################
##################################################缓存之文件缓存
#缓存类型之文件缓存，主要缓存静态文件
#LoadModule file_cache_module modules/mod_file_cache.so
#需要添加两个CacheFile(用于打开指定静态文件到内存中)与 
#MMapFile用于构建指定静态文件的列表到内存中

#################################################缓存之对动态文件的缓存
#存在两个存储管理模块，mod_cache_disk和mod_cache_socache
#启用基于URL键的内容动态缓冲到内存或者磁盘中
#详细配置https://httpd.apache.org/docs/2.4/mod/mod_cache.html
LoadModule cache_module modules/mod_cache.so

#启用基于磁盘的缓冲管理器，响应头和相应体在磁盘中的不同位置利用md5
LoadModule cache_disk_module modules/mod_cache_disk.so

#共享对象缓存(也就是键值缓存)将响应头和响应体合并在一起缓存
LoadModule cache_socache_module modules/mod_cache_socache.so
#推荐使用shmcb
#LoadModule socache_shmcb_module modules/mod_socache_shmcb.so
#使用DBM进行缓存
#LoadModule socache_dbm_module modules/mod_socache_dbm.so
#LoadModule socache_memcache_module modules/mod_socache_memcache.so





#做一个hook来进行定时任务
#LoadModule watchdog_module modules/mod_watchdog.so
#构建宏命令，批量创建vhost
#LoadModule macro_module modules/mod_macro.so
#处理sql数据库连接
#LoadModule dbd_module modules/mod_dbd.so
#记录apacheIO日志，调试时打开
#LoadModule dumpio_module modules/mod_dumpio.so




#########################过滤器系列###############################################

#过滤HTTP体请求
#LoadModule request_module modules/mod_request.so
#上下文过滤配置
LoadModule filter_module modules/mod_filter.so
#对客户端进行带宽限速
#LoadModule ratelimit_module modules/mod_ratelimit.so
LoadModule deflate_module modules/mod_deflate.so
#响应到达客户端之前通过内在程序处理
#LoadModule ext_filter_module modules/mod_ext_filter.so

#服务端解析HTML语法
#LoadModule include_module modules/mod_include.so

#LoadModule substitute_module modules/mod_substitute.so
#LoadModule sed_module modules/mod_sed.so

#对HTTP请求进行缓冲
#LoadModule buffer_module modules/mod_buffer.so
#指定接受请求的限时和速率
LoadModule reqtimeout_module modules/mod_reqtimeout.so

# Configure mod_proxy_html to understand HTML4/XHTML1
<IfModule proxy_html_module>
Include /etc/httpd/extra/proxy-html.conf
</IfModule>

#限制HTTP请求方式
#LoadModule allowmethods_module modules/mod_allowmethods.so


###############################################################

#开启服务器针对不同文件拓展名进行不同的操作和以及编码
LoadModule mime_module modules/mod_mime.so
#针对客户端请求进行记录
LoadModule log_config_module modules/mod_log_config.so
#额外的调试日志记录
#LoadModule log_debug_module modules/mod_log_debug.so
#对请求和响应的字节数进行记录包括SSL
#LoadModule logio_module modules/mod_logio.so

#更改发送到CGI和SSI页面的环境变量
LoadModule env_module modules/mod_env.so
#控制缓存过期时间
#LoadModule expires_module modules/mod_expires.so
#该指令可以控制和更改HTTP请求和响应头
LoadModule headers_module modules/mod_headers.so
#提供一个环境变量去定义每一个HTTP请求
#LoadModule unique_id_module modules/mod_unique_id.so
#可以使用正则来设置环境变量
LoadModule setenvif_module modules/mod_setenvif.so
#设定不同版本apache所响应的动作
LoadModule version_module modules/mod_version.so
#负载均衡中设置用户IP
#LoadModule remoteip_module modules/mod_remoteip.so

##################################反向代理模块#########################################
#apache代理模块
#开启apache代理，提供一个基本的代理功能
#LoadModule proxy_module modules/mod_proxy.so

#负载均衡使用 必须要开启mod_proxy+mod_proxy_balancer+lbmethod/4
#LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
#基于请求的负载均衡策略
#LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so
#基于流量的负载均衡策略
#LoadModule lbmethod_bytraffic_module modules/mod_lbmethod_bytraffic.so
#这个主要请求给不忙的服务器
#LoadModule lbmethod_bybusyness_module modules/mod_lbmethod_bybusyness.so
#基于心跳的负载均衡策略
#LoadModule lbmethod_heartbeat_module modules/mod_lbmethod_heartbeat.so
#该配置动态配置反向代理
#LoadModule proxy_express_module modules/mod_proxy_express.so
#对负载均衡器后端动态健康检查
#LoadModule proxy_hcheck_module modules/mod_proxy_hcheck.so


#CONNECT (for SSL)
#LoadModule proxy_connect_module modules/mod_proxy_connect.so
#LoadModule proxy_ftp_module modules/mod_proxy_ftp.so
#LoadModule proxy_http_module modules/mod_proxy_http.so
#LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
#LoadModule proxy_scgi_module modules/mod_proxy_scgi.so
#他将socket包传到其他程序
#LoadModule proxy_fdpass_module modules/mod_proxy_fdpass.so

#Web-sockets
#LoadModule proxy_wstunnel_module modules/mod_proxy_wstunnel.so

#Apache JServe Protocol version
#LoadModule proxy_ajp_module modules/mod_proxy_ajp.so


###########session模块#####################################################
#加入session支持
#LoadModule session_module modules/mod_session.so
#mod_session子模块，添加HTTPcookie支持
#LoadModule session_cookie_module modules/mod_session_cookie.so
#基于dbd/sql的session支持
#LoadModule session_dbd_module modules/mod_session_dbd.so

#该模块开启一个共享内存区域，数据以插槽的形式存储
#LoadModule slotmem_shm_module modules/mod_slotmem_shm.so
#该模块提供加密SSL和TLS其中SSL v3 and TLS v1.x 
#LoadModule ssl_module modules/mod_ssl.so


#基本安全策略对Linux系统，必须打开
LoadModule unixd_module modules/mod_unixd.so
#发布 WebDAV ('Web-based Distributed Authoring and Versioning')
#LoadModule dav_module modules/mod_dav.so
#对mod_dav支持文件系统
#LoadModule dav_fs_module modules/mod_dav_fs.so

#提供服务链接状态和性能表现
LoadModule status_module modules/mod_status.so
#提供站点目录文件列表
LoadModule autoindex_module modules/mod_autoindex.so
#提供apache综合的配置概览
#LoadModule info_module modules/mod_info.so
#使用外部的程序执行cgi脚本
#LoadModule cgid_module modules/mod_cgid.so

#提供对虚拟主机的批量配置
LoadModule vhost_alias_module modules/mod_vhost_alias.so
#提供对不同浏览器的编码文件类型等的支持
#LoadModule negotiation_module modules/mod_negotiation.so
#提供结尾“/”补充
LoadModule dir_module modules/mod_dir.so

#基于媒体类型或请求方法，为执行CGI脚本而提供
#LoadModule actions_module modules/mod_actions.so

#尝试去忽略url中拼写去调整url
#LoadModule speling_module modules/mod_speling.so
#该模块提供用户自定义url支持
#LoadModule userdir_module modules/mod_userdir.so
#提供不同文件系统的文档重定向
LoadModule alias_module modules/mod_alias.so

#启用重写
LoadModule rewrite_module modules/mod_rewrite.so

<IfModule unixd_module>
#
# If you wish httpd to run as a different user or group, you must run
# httpd as root initially and it will switch.  
#
# User/Group: The name (or #number) of the user/group to run httpd as.
# It is usually good practice to create a dedicated user and group for
# running httpd, as with most system services.
#
User daemon
Group daemon

</IfModule>

# 'Main' server configuration
#
# The directives in this section set up the values used by the 'main'
# server, which responds to any requests that aren't handled by a
# <VirtualHost> definition.  These values also provide defaults for
# any <VirtualHost> containers you may define later in the file.
#
# All of these directives may appear inside <VirtualHost> containers,
# in which case these default settings will be overridden for the
# virtual host being defined.
#

#
# ServerAdmin: Your address, where problems with the server should be
# e-mailed.  This address appears on some server-generated pages, such
# as error documents.  e.g. admin@your-domain.com
#
ServerAdmin you@example.com

#
# ServerName gives the name and port that the server uses to identify itself.
# This can often be determined automatically, but we recommend you specify
# it explicitly to prevent problems during startup.
#
# If your host doesn't have a registered DNS name, enter its IP address here.
#
ServerName localhost:80

#
# Deny access to the entirety of your server's filesystem. You must
# explicitly permit access to web content directories in other 
# <Directory> blocks below.
#
<Directory />
    AllowOverride none
    Require all denied
</Directory>

#
# Note that from this point forward you must specifically allow
# particular features to be enabled - so if something's not working as
# you might expect, make sure that you have specifically enabled it
# below.
#

#
# DocumentRoot: The directory out of which you will serve your
# documents. By default, all requests are taken from this directory, but
# symbolic links and aliases may be used to point to other locations.
#
DocumentRoot "/usr/local/apached/apache24/htdocs"
<Directory "/usr/local/apached/apache24/htdocs">
    #
    # Possible values for the Options directive are "None", "All",
    # or any combination of:
    #   Indexes Includes FollowSymLinks SymLinksifOwnerMatch ExecCGI MultiViews
    #
    # Note that "MultiViews" must be named *explicitly* --- "Options All"
    # doesn't give it to you.
    #
    # The Options directive is both complicated and important.  Please see
    # http://httpd.apache.org/docs/2.4/mod/core.html#options
    # for more information.
    #
    Options Indexes FollowSymLinks

    #
    # AllowOverride controls what directives may be placed in .htaccess files.
    # It can be "All", "None", or any combination of the keywords:
    #   AllowOverride FileInfo AuthConfig Limit
    #
    AllowOverride None

    #
    # Controls who can get stuff from this server.
    #
    Require all granted
</Directory>

#
# DirectoryIndex: sets the file that Apache will serve if a directory
# is requested.
#
<IfModule dir_module>
    DirectoryIndex index.html
</IfModule>

#
# The following lines prevent .htaccess and .htpasswd files from being 
# viewed by Web clients. 
#
<Files ".ht*">
    Require all denied
</Files>

#
# ErrorLog: The location of the error log file.
# If you do not specify an ErrorLog directive within a <VirtualHost>
# container, error messages relating to that virtual host will be
# logged here.  If you *do* define an error logfile for a <VirtualHost>
# container, that host's errors will be logged there and not here.
#
ErrorLog "logs/error_log"

#
# LogLevel: Control the number of messages logged to the error_log.
# Possible values include: debug, info, notice, warn, error, crit,
# alert, emerg.
#
LogLevel warn

<IfModule log_config_module>
    #
    # The following directives define some format nicknames for use with
    # a CustomLog directive (see below).
    #
    LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
    LogFormat "%h %l %u %t \"%r\" %>s %b" common

    <IfModule logio_module>
      # You need to enable mod_logio.c to use %I and %O
      LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\" %I %O" combinedio
    </IfModule>

    #
    # The location and format of the access logfile (Common Logfile Format).
    # If you do not define any access logfiles within a <VirtualHost>
    # container, they will be logged here.  Contrariwise, if you *do*
    # define per-<VirtualHost> access logfiles, transactions will be
    # logged therein and *not* in this file.
    #
    CustomLog "logs/access_log" common

    #
    # If you prefer a logfile with access, agent, and referer information
    # (Combined Logfile Format) you can use the following directive.
    #
    #CustomLog "logs/access_log" combined
</IfModule>

<IfModule alias_module>
    #
    # Redirect: Allows you to tell clients about documents that used to 
    # exist in your server's namespace, but do not anymore. The client 
    # will make a new request for the document at its new location.
    # Example:
    # Redirect permanent /foo http://www.example.com/bar

    #
    # Alias: Maps web paths into filesystem paths and is used to
    # access content that does not live under the DocumentRoot.
    # Example:
    # Alias /webpath /full/filesystem/path
    #
    # If you include a trailing / on /webpath then the server will
    # require it to be present in the URL.  You will also likely
    # need to provide a <Directory> section to allow access to
    # the filesystem path.

    #
    # ScriptAlias: This controls which directories contain server scripts. 
    # ScriptAliases are essentially the same as Aliases, except that
    # documents in the target directory are treated as applications and
    # run by the server when requested rather than as documents sent to the
    # client.  The same rules about trailing "/" apply to ScriptAlias
    # directives as to Alias.
    #
    ScriptAlias /cgi-bin/ "/usr/local/apached/apache24/cgi-bin/"

</IfModule>

<IfModule cgid_module>
    #
    # ScriptSock: On threaded servers, designate the path to the UNIX
    # socket used to communicate with the CGI daemon of mod_cgid.
    #
    #Scriptsock cgisock
</IfModule>

#
# "/usr/local/apached/apache24/cgi-bin" should be changed to whatever your ScriptAliased
# CGI directory exists, if you have that configured.
#
<Directory "/usr/local/apached/apache24/cgi-bin">
    AllowOverride None
    Options None
    Require all granted
</Directory>

<IfModule headers_module>
    #
    # Avoid passing HTTP_PROXY environment to CGI's on this or any proxied
    # backend servers which have lingering "httpoxy" defects.
    # 'Proxy' request header is undefined by the IETF, not listed by IANA
    #
    RequestHeader unset Proxy early
</IfModule>

<IfModule mime_module>
    #
    # TypesConfig points to the file containing the list of mappings from
    # filename extension to MIME-type.
    #
    TypesConfig /etc/httpd/mime.types

    #
    # AddType allows you to add to or override the MIME configuration
    # file specified in TypesConfig for specific file types.
    #
    #AddType application/x-gzip .tgz
    #
    # AddEncoding allows you to have certain browsers uncompress
    # information on the fly. Note: Not all browsers support this.
    #
    #AddEncoding x-compress .Z
    #AddEncoding x-gzip .gz .tgz
    #
    # If the AddEncoding directives above are commented-out, then you
    # probably should define those extensions to indicate media types:
    #
    AddType application/x-compress .Z
    AddType application/x-gzip .gz .tgz

    #
    # AddHandler allows you to map certain file extensions to "handlers":
    # actions unrelated to filetype. These can be either built into the server
    # or added with the Action directive (see below)
    #
    # To use CGI scripts outside of ScriptAliased directories:
    # (You will also need to add "ExecCGI" to the "Options" directive.)
    #
    #AddHandler cgi-script .cgi

    # For type maps (negotiated resources):
    #AddHandler type-map var

    #
    # Filters allow you to process content before it is sent to the client.
    #
    # To parse .shtml files for server-side includes (SSI):
    # (You will also need to add "Includes" to the "Options" directive.)
    #
    #AddType text/html .shtml
    #AddOutputFilter INCLUDES .shtml
</IfModule>

#
# The mod_mime_magic module allows the server to use various hints from the
# contents of the file itself to determine its type.  The MIMEMagicFile
# directive tells the module where the hint definitions are located.
#
#MIMEMagicFile /etc/httpd/magic

#
# Customizable error responses come in three flavors:
# 1) plain text 2) local redirects 3) external redirects
#
# Some examples:
#ErrorDocument 500 "The server made a boo boo."
#ErrorDocument 404 /missing.html
#ErrorDocument 404 "/cgi-bin/missing_handler.pl"
#ErrorDocument 402 http://www.example.com/subscription_info.html
#

#
# MaxRanges: Maximum number of Ranges in a request before
# returning the entire resource, or one of the special
# values 'default', 'none' or 'unlimited'.
# Default setting is to accept 200 Ranges.
#MaxRanges unlimited

#
# EnableMMAP and EnableSendfile: On systems that support it, 
# memory-mapping or the sendfile syscall may be used to deliver
# files.  This usually improves server performance, but must
# be turned off when serving from networked-mounted 
# filesystems or if support for these functions is otherwise
# broken on your system.
# Defaults: EnableMMAP On, EnableSendfile Off
#
#EnableMMAP off
#EnableSendfile on

# Supplemental configuration
#
# The configuration files in the /etc/httpd/extra/ directory can be 
# included to add extra features or to modify the default configuration of 
# the server, or you may simply copy their contents here and change as 
# necessary.

# Server-pool management (MPM specific)Multi-Processing Modules
#启用多进程模块
Include /etc/httpd/extra/httpd-mpm.conf

# Multi-language error messages
#Include /etc/httpd/extra/httpd-multilang-errordoc.conf

# Fancy directory listings
#Include /etc/httpd/extra/httpd-autoindex.conf

# Language settings
#Include /etc/httpd/extra/httpd-languages.conf

# User home directories
#Include /etc/httpd/extra/httpd-userdir.conf

# Real-time info on requests and configuration
#Include /etc/httpd/extra/httpd-info.conf

# Virtual hosts
Include /etc/httpd/extra/httpd-vhosts.conf

# Local access to the Apache HTTP Server Manual
#Include /etc/httpd/extra/httpd-manual.conf

# Distributed authoring and versioning (WebDAV)
#Include /etc/httpd/extra/httpd-dav.conf

# Various default settings启用默认拓展，修改httpd主要配置
Include /etc/httpd/extra/httpd-default.conf



# Secure (SSL/TLS) connections
#Include /etc/httpd/extra/httpd-ssl.conf
#
# Note: The following must must be present to support
#       starting without SSL on platforms with no /dev/random equivalent
#       but a statically compiled-in mod_ssl.
#
<IfModule ssl_module>
SSLRandomSeed startup builtin
SSLRandomSeed connect builtin
</IfModule>

Include /etc/httpd/extra/httpd-vhosts.conf


```

# 3 配置ssl

### 3.1 配置

- 开启ssl模块，在/etc/httpd.conf中去掉下面行注释
- 开启shmcb缓存模块，ssl需要用到
- 将httpd-ssl.conf包含到主配置文件

去掉/etc/httpd.conf下面三行注释

```
#LoadModule ssl_module modules/mod_ssl.so
#LoadModule socache_shmcb_module modules/mod_socache_shmcb.so
#Include /etc/httpd/extra/httpd-ssl.conf

```

- 添加证书和key文件，把文件移动到/etc/httpd/分别命名为.key,.crt,修改<VirtualHost _default_:443>中域名

```
ServerName localhost:443
SSLCertificateFile "/etc/httpd/server.crt"
SSLCertificateKeyFile "/etc/httpd/server.key"

```

### 3.2 自动跳转

主配置文件首先开启跳转模块

```
LoadModule rewrite_module modules/mod_rewrite.so
```

- 在网站根目录创建文件**.htaccess**

```
RewriteEngine On 
RewriteCond %{HTTPS}  !=on 
RewriteRule ^/?(.*) https://%{SERVER_NAME}/$1 [R,L] 

```

- vhost中配置

```xml
<VirtualHost *:80>
    ServerAdmin webmaster@kws.com
    DocumentRoot /kuaiyun/www_local/vip.ddgo.cc
    ServerName erp.ddgo.vip
    CustomLog logs/vip.ddgo.cc-access_log combined
    ErrorLog logs/vip.ddgo.cc-error_log
    DirectoryIndex index.html index.php
    <IfModule mpm_itk_module>
       AssignUserId www www
    </IfModule>
	<Directory "/kuaiyun/www_local/vip.ddgo.cc">
                Options Indexes FollowSymLinks MultiViews
                AllowOverride All
                Order allow,deny
                Allow from all
	</Directory>
</VirtualHost>
```








