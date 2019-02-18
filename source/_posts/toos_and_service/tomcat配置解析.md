---
title: tomcat配置解析
date: 2018-11-04 12:12:28
tags:
- tomcat
---
## 1.  tomcat配置解析

### 1.1 二进制安装

- jdk8
- tomcat8.5
- 注意卸载系统的自带`java`环境`rpm -qa | grep java`

<!--more-->
```
wget http://git.centos8.com/lirui/nor_service/raw/master/tomcat/tomcat85.sh && bash tomcat85.sh
```

### 1.2 rpm安装

下载rpm包直接安装,**该安装没有配置环境变量**

配置环境变量

```bash
#安装后，做了java二进制软连系欸
# ll /usr/bin/java
lrwxrwxrwx 1 root root 22 May 24 18:03 /usr/bin/java -> /etc/alternatives/java
# ll /etc/alternatives/java
lrwxrwxrwx 1 root root 41 May 24 18:03 /etc/alternatives/java -> /usr/java/jdk1.8.0_171-amd64/jre/bin/java

#配置环境变量
vi /etc/profile
export JAVA_HOME=/usr/java/jdk1.7.0_80
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

source /etc/profile

#删除软连接
rm -rf /etc/alternatives/java
rm -rf /usr/bin/java

```


### 2. 配置详解

- 配置文件夹内概览/usr/local/tomcat-8.5.27/conf/server.conf

```bash
├── Catalina
│   └── localhost
├── catalina.policy #java安全策略配置
├── catalina.properties #内部包定义访问
├── context.xml #所有host的默认配置信息
├── jaspic-providers.xml
├── jaspic-providers.xsd
├── logging.properties #配置日志记录等信息
├── server.xml #主配置文件
├── tomcat-users.xml #Realm认证管理角色
├── tomcat-users.xsd
└── web.xml #配置servlet

```

#### 2.1 server.xml主配置文件

- 架构图
![enter image description here](https://qiniu.li-rui.top/20130826174226703.jpg)

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!--代表整个容器，最顶层元素在端口8005接受命令telnet localhost 8005 -->
<Server port="8005" shutdown="SHUTDOWN">
  <!--监听器，可以在特定的事件发生时执行特定的动作，比如tomcat的启动和停止-->

  <!--记录tomcat和Java以及操作系统信息，必须配置-->
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <!--Apache Portable Runtime APR apache可移植运行库检测，如果存在则进行加载实现高性能-->
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <!--检测类加载器的内存泄露-->
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <!--初始化<GlobalNamingResources>中的全局资源-->
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <!--ThreadLocal仅仅支持 他自己线程创建的变量，其他线程不可以访问，在GC运行时会导致内存泄露，此监听器
  可以刷新线程池，当Context元素的renewThreadsWhenStoppingContext属性设置为true才可与启用该监听器-->
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <!--  定义全局资源   -->
  <GlobalNamingResources>
    <!--这里定义了一个用户数据库UserDatabase，数据存储在conf/tomcat-users.xml文件内-->
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <!--容器下的服务，包含一个engine和connector-->
  <Service name="Catalina">

    <!--连接客户端和内部容器的连接器，用来收发数据包-->
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />

    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />


    <!--请求处理引擎，处理connector传递过来的请求
	Engine包含Host，Host包含Context。一个Engine组件可以处理Service中的所有请求，
	一个Host组件可以处理发向一个特定虚拟主机的所有请求，
	一个Context组件可以处理一个特定Web应用的所有请求-->
    <Engine name="Catalina" defaultHost="localhost">




      <!--Realm配置-->
      <Realm className="org.apache.catalina.realm.LockOutRealm">

	    <!--Realm提供了一种用户密码与web应用的映射关系,这里面使用的数据源为UserDatabase
		这是在<GlobalNamingResources>中定义的-->
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>

	  <!--虚拟主机，包含多个web应用-->
      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">


        <!--这里定义了一个valve也就是 “阀门”，它时请求处理中一个组建，下面时针对上层host的日志记录配置的“阀门”-->
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
        <!--配置改虚拟主机中的一个web应用
<Context path="/kaka" docBase="kaka" debug="0" reloadbale="true">
-->

      </Host>
    </Engine>
  </Service>
</Server>

```

### 3. 配置HTTPS

#### 3.1 配置证书

- tomcat https实现两种方案JSSE和APR
- JSSE包含IO和NIO
- APR可以利用opensslcrt和key文件

<br>下面采用JSSE NIO实现，首先将证书转换为jks文件,放在conf目录，然后在<Connector port="80"/>下面添加

```xml
<Connector port="443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true">
        <SSLHostConfig>
            <Certificate certificateKeystoreFile="conf/tom.jks"
			 certificateKeystorePassword="123456789"
                         type="RSA" />
        </SSLHostConfig>
</Connector>
```

#### 3.2 自动跳转

在网站目录内文件**web.xml**后面添加

```xml
<login-config>
        <!-- Authorization setting for SSL -->
        <auth-method>CLIENT-CERT</auth-method>
        <realm-name>Client Cert Users-only Area</realm-name>
    </login-config>
    <security-constraint>
        <!-- Authorization setting for SSL -->
        <web-resource-collection >
            <web-resource-name >SSL</web-resource-name>
            <url-pattern>/*</url-pattern>
        </web-resource-collection>
        <user-data-constraint>
            <transport-guarantee>CONFIDENTIAL</transport-guarantee>
        </user-data-constraint>
    </security-constraint>
```

### 4 多虚拟主机配置

#### 4.1 单端口多虚拟主机

 - 注意默认目录就在`/var/local/test1/ROOT`内
 - 文件`server.xml`在`<Engine></Engine>`标签内，添加`Host`标签

```xml
<Host name="www.test1.com"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
      <Context path="" docBase="/var/local/test1/ROOT" debug="0"/>
       <!-- 日志记录格式 -->
       <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="wholesale-admin_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />

</Host>

<Host name="www.test2.com"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
      <Context path="" docBase="/var/local/test2/ROOT" debug="0"/>
       <!-- 日志记录格式 -->
       <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="wholesale-admin_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />

</Host>

```
#### 4.2 多端口多域名

- 文件`server.xml`在`<Server></Server>`标签内，添加`Service`标签

```xml
<!--
需要改动的地方有
Service name,
<Connector port="8081" protocol="HTTP/1.1"
<Connector port="8091" protocol="AJP/1.3"
defaultHost="www.test1.com">

--->
<Service name="Catalina1">

    <Connector port="8081" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />

    <Connector port="8091" protocol="AJP/1.3" redirectPort="8443" />



    <Engine name="Catalina" defaultHost="www.test1.com">


      <Realm className="org.apache.catalina.realm.LockOutRealm">

        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>


        <Host name="www.test1.com"  appBase="webapps"
                    unpackWARs="true" autoDeploy="true">
                <Context path="" docBase="/var/local/test1/ROOT" debug="0"/>
                <!-- 日志记录格式 -->
                <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                        prefix="wholesale-admin_access_log" suffix=".txt"
                        pattern="%h %l %u %t &quot;%r&quot; %s %b" />

        </Host>


    </Engine>
  </Service>

  <Service name="Catalina2">

    <Connector port="8082" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />

    <Connector port="8092" protocol="AJP/1.3" redirectPort="8443" />



    <Engine name="Catalina" defaultHost="www.test2.com">


      <Realm className="org.apache.catalina.realm.LockOutRealm">

        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>


        <Host name="www.test2.com"  appBase="webapps"
                    unpackWARs="true" autoDeploy="true">
                <Context path="" docBase="/var/local/test2/ROOT" debug="0"/>
                <!-- 日志记录格式 -->
                <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                        prefix="wholesale-admin_access_log" suffix=".txt"
                        pattern="%h %l %u %t &quot;%r&quot; %s %b" />

        </Host>


    </Engine>
  </Service>

```

#### 4.3 单域名多端口

- 在`<Service><Service/>`标签内添加`Connector`标签

```xml
<Connector port="8081" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
<Connector port="8082" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />


```

### 5. 多Tomcat进程启动

- 主要是改动下面三个端口

```bash
<Server port="8092" shutdown="SHUTDOWN">
<Connector port="8082" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
<Connector port="8029" protocol="AJP/1.3" redirectPort="8443" />
```
### 6. tomcat内存限制

- 在启动脚本`catalina.sh`最后加入
- 两个参数分别为：初始化内存，可以使用内存

```bash
JAVA_OPTS='-Xms2048m -Xmx2048m'
```

### 7 tomcat启动

- 启动脚本

- `centos 7/6`的启动脚本位置`/etc/rc.d/rc.local`
- 注意`chmod +x /etc/rc.d/rc.local`
- 主要是添加`java`的环境变量

```bash
#java path
JAVA_HOME=/var/local/jdk1.8.0_144
JRE_HOME=/var/local/jdk1.8.0_144/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH


/bin/sh /var/local/www/bin/startup.sh

```

- 启动服务systemctl

```bash
# Systemd unit file for tomcat
[Unit]
Description=Apache Tomcat Web Application Container
After=syslog.target network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/jre
Environment=CATALINA_PID=/opt/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/tomcat
Environment=CATALINE_BASE=/opt/tomcat
Environment='CATALINE_OPTS=-Xms128M -Xmx765M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.haedless=true -Djava.security.egd=file:/dev/./urandom'

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/bin/kill -15 $MAINPID

User=tomcat
Group=tomcat

[Install]
WantedBy=multi-user.target
```

- 启动init.d

```bash
#!/bin/bash
#
# chkconfig: 2345 95 20
# description: Tomcat 8 start/stop/status init.d script
# processname: tomcat
#
# Tomcat 8 start/stop/status init.d script
#
# Updates:
# @author: Tongliang Liu <cooniur@gmail.com>
# Added chkconfig header to make it work with chkconfig
#
# Initially forked from: https://gist.github.com/valotas/1000094
# @author: Miglen Evlogiev <bash@miglen.com>
#
# Original Release updates:
# Updated method for gathering pid of the current proccess
# Added usage of CATALINA_BASE
# Added coloring and additional status
# Added check for existence of the tomcat user
# Added termination proccess
#

#Location of JAVA_HOME (the directory contains bin folder)
export JAVA_HOME=/usr/lib/jvm/default

#Add Java binary files to PATH
export PATH=$JAVA_HOME/bin:$PATH

#CATALINA_HOME is the location of the bin files of Tomcat
export CATALINA_HOME=/var/local/tomcat

#CATALINA_BASE is the location of the configuration files of this instance of Tomcat
export CATALINA_BASE=/var/local/tomcat

#TOMCAT_USER is the default user of tomcat
export TOMCAT_USER=tomcat

#TOMCAT_USAGE is the message if this script is called without any options
TOMCAT_USAGE="Usage: $0 {\e[00;32mstart\e[00m|\e[00;31mstop\e[00m|\e[00;31mkill\e[00m|\e[00;32mstatus\e[00m|\e[00;31mrestart\e[00m}"

#SHUTDOWN_WAIT is wait time in seconds for java proccess to stop
SHUTDOWN_WAIT=20

tomcat_pid() {
  echo `ps -fe | grep $CATALINA_BASE | grep -v grep | tr -s " "|cut -d" " -f2`
}

start() {
  pid=$(tomcat_pid)
  if [ -n "$pid" ]; then
    echo -e "\e[00;31mTomcat is already running (pid: $pid)\e[00m"
  else
    # Start tomcat
    echo -e "\e[00;32mStarting tomcat\e[00m"
    ulimit -n 100000
    umask 007
    if [ `user_exists $TOMCAT_USER` = "1" ]
    then
      /bin/su $TOMCAT_USER -c $CATALINA_HOME/bin/startup.sh
    else
      sh $CATALINA_HOME/bin/startup.sh
    fi
    status
  fi
  return 0
}

status(){
  pid=$(tomcat_pid)
  if [ -n "$pid" ]; then
    echo -e "\e[00;32mTomcat is running with pid: $pid\e[00m"
  else
    echo -e "\e[00;31mTomcat is not running\e[00m"
  fi
}

terminate() {
  echo -e "\e[00;31mTerminating Tomcat\e[00m"
  kill -9 $(tomcat_pid)
}

stop() {
  pid=$(tomcat_pid)
  if [ -n "$pid" ]; then
    echo -e "\e[00;31mStoping Tomcat\e[00m"
    sh $CATALINA_HOME/bin/shutdown.sh

    let kwait=$SHUTDOWN_WAIT
    count=0;
    until [ `ps -p $pid | grep -c $pid` = '0' ] || [ $count -gt $kwait ]
    do
      echo -n -e "\n\e[00;31mwaiting for processes to exit\e[00m";
      sleep 1
      let count=$count+1;
    done

    if [ $count -gt $kwait ]; then
      echo -n -e "\n\e[00;31mkilling processes didn't stop after $SHUTDOWN_WAIT seconds\e[00m"
      terminate
    fi
  else
    echo -e "\e[00;31mTomcat is not running\e[00m"
  fi

  return 0
}

user_exists(){
  if id -u $1 >/dev/null 2>&1; then
    echo "1"
  else
    echo "0"
  fi
}

case $1 in
  start)
    start
  ;;
  stop)
    stop
  ;;
  restart)
    stop
    start
  ;;
  status)
    status
  ;;
  kill)
    terminate
  ;;
  *)
    echo -e $TOMCAT_USAGE
  ;;
esac

exit 0
```

### 8 tomcat日志输出

- 文件为`tailf /var/local/tomcat-8.5.27/logs/catalina.out`
