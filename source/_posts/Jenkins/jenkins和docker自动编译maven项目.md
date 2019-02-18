---
title: jenkins和docker自动编译maven项目
tags:
- jenkins
---

# jenkins+maven镜像

dockerfile

<!--more-->

```bash
#################################################################
# DO NOT UPGRADE alpine until https://github.com/jenkinsci/docker/issues/508 is fixed
FROM openjdk:8u121-jdk-alpine

MAINTAINER pcp-team@lirui

RUN apk add --no-cache git openssh-client curl unzip bash ttf-dejavu coreutils \
    && apk add docker --update-cache --repository http://mirrors.ustc.edu.cn/alpine/v3.4/main/ --allow-untrusted

ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000
ARG http_port=8080
ARG agent_port=50000

ENV JENKINS_HOME /var/jenkins_home
ENV JENKINS_SLAVE_AGENT_PORT ${agent_port}

# Jenkins is run with user `jenkins`, uid = 1000
# If you bind mount a volume from the host or a data container,
# ensure you use the same uid
RUN addgroup -g ${gid} ${group} \
    && adduser -h "$JENKINS_HOME" -u ${uid} -G ${group} -s /bin/bash -D ${user} \
        && mkdir -p /usr/share/jenkins/ref/init.groovy.d

# Jenkins home directory is a volume, so configuration and build history
# can be persisted and survive image upgrades
VOLUME /var/jenkins_home

# `/usr/share/jenkins/ref/` contains all reference configuration we want
# to set on a fresh new installation. Use it to bundle additional plugins
# or config file with your custom jenkins Docker image.
#RUN mkdir -p /usr/share/jenkins/ref/init.groovy.d


# Use tini as subreaper in Docker container to adopt zombie processes
ADD tini-static-amd64 /bin/tini
RUN chmod +x /bin/tini


COPY init.groovy /usr/share/jenkins/ref/init.groovy.d/tcp-slave-agent-port.groovy

ADD jenkins-war-2.149.war /usr/share/jenkins/jenkins.war


ENV JENKINS_UC https://updates.jenkins.io
RUN chown -R ${user} "$JENKINS_HOME" /usr/share/jenkins/ref

# install maven
ADD apache-maven-3.2.5-bin.tar.gz /opt
RUN ln -s /opt/apache-maven-3.2.5 /opt/maven
RUN ln -s /opt/maven/bin/mvn /usr/local/bin
ENV MAVEN_HOME /opt/maven

# for main web interface:
EXPOSE ${http_port}

# will be used by attached slave agents:
EXPOSE ${agent_port}

ENV COPY_REFERENCE_FILE_LOG $JENKINS_HOME/copy_reference_file.log

USER ${user}

COPY jenkins-support /usr/local/bin/jenkins-support
COPY jenkins.sh /usr/local/bin/jenkins.sh
ENTRYPOINT ["/bin/tini", "--", "/usr/local/bin/jenkins.sh"]

# from a derived Dockerfile, can use `RUN plugins.sh active.txt` to setup /usr/share/jenkins/ref/plugins from a support bundle
COPY plugins.sh /usr/local/bin/plugins.sh
COPY install-plugins.sh /usr/local/bin/install-plugins.sh

```

## 安装Jenkins

```bash
docker pull jenkins/jenkins

docker run -p 8002:8080 -p 50000:50000 -d --restart=always --name jenkins -u root -v /opt/jkins:/var/jenkins_home jenkins/jenkins
```

## compose启动

```yml
version: '3.1'
services:
  jenkins:
    image: jenkinsm:1.0
    container_name: jenkins
    environment:
      JAVA_OPTS: "-Djava.awt.headless=true"
    user: root
    ports:
      - 50000:50000
      - 8080:8080
    volumes:
      - /data/jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /data/mvn_files:/root/.m2
      - /data/maven/setting.xml:/opt/apache-maven-3.2.5/conf/settings.xml

```

## 安装相关插件

blueocen

# 项目中准备

项目根目录中

```bash
[root@localhost pcp_test_master]# ll -h
total 36K
-rw-r--r-- 1 root root  176 Jan 16 17:57 Dockerfile
-rw-r--r-- 1 root root  659 Jan 16 17:57 Jenkinsfile
-rw-r--r-- 1 root root  18K Jan 16 18:17 pom.xml
drwxr-xr-x 3 root root 4.0K Jan 16 17:57 src
drwxr-xr-x 4 root root 4.0K Jan 16 18:21 target
```

## Jenkinsfile

```bash
pipeline {

	agent any
    stages {

        stage('build war package') {
            
            steps {
            echo 'build war package by maven'
            sh 'mvn clean package'
            }
        }



        stage('build docker image') {
            
            steps {
            echo 'tag image then build docker image'
            sh 'docker build -t docker.li-rui.top/pcp/ppp:0.1 .'
            }
        }

        stage('push docker image') {
            steps {
            sh 'docker push docker.li-rui.top/pcp/ppp:0.1'
            }
        }

    }

}

```

## dockerfile

```bash
FROM docker.li-rui.top/pcp/tomcat:0.5
MAINTAINER pcp-team@lirui
LABEL author="pcp-team" \
      version="0.3"

COPY target/ppp.war server/apache-tomcat-7.0.69/webapps

```

# 添加git库

在blueocen中添加pipeline

http格式，用户名密码

