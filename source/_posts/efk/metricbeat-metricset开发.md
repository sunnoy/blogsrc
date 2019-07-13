---
title: metricbeat-metricset开发
date: 2018-11-05 12:12:28
tags:
- elk
- metricbeat
---

# metricbeat-metricset开发

## 开发方向

- 直接拓展metricbeat
- 直接开发beat让metricbeat作为库依赖使用

## Metricbeat构成

### Metricbeat module

Metricbeat包含一系列的module和metricsets，module的命名有他所采集的服务来决定比如mysqlmodule，每个module包含了许多metricsets，metricsets可以用一个指令返回多条数据，比如mysqlmodule的status metricsets就是来返回mysql状态信息。
<!-- more -->
#### module和metricsets开发要求

- 必须有fields.yml用来生成module文档和Elasticsearch模板
- 有文档文件
- 有集成测试
- 涵盖80%的测试

## 创建一个metricset

metricset是module的一部分，metricset用来采集并结构化服务的数据

### 执行create-metricset指令

首先安装Virtualenv，该工具用来在同一个系统中对不同的pyhton环境进行隔离，metricset脚本需要用到

```bash
yum -y install python-setuptools
easy_install virtualenv
```

或者

```bash
curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
python get-pip.py
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple  virtualenv
#永久配置清华大学镜像
#pip install pip -U
#pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

在 go/src/github.com/elastic/beats/metricbeat下面执行

```bash
make create-metricset
```

该命令会调用当前目录下的Makefile文件的指令`create-metricset`，由Makefile可知调用的python脚本

```Makefile
# Creates a new metricset. Requires the params MODULE and METRICSET
.PHONY: create-metricset
create-metricset: python-env
        @${PYTHON_ENV}/bin/python ${ES_BEATS}/metricbeat/scripts/create_metricset.py --path=$(PWD) --es_beats=$(ES_BEATS) --module=$(MODULE) --metricset=$(METRICSET)
```

该脚本使用交互模式要求输入module名称和metricset名称，如果输入的module没有就会自动创建并且准备好相应的基础文件。基础文件来自于go/src/github.com/elastic/beats/metricbeat/scripts下面

### 编译

编译需要执行两条命令

```bash
make collect
make
```

#### make collect
第一条命令也可见Makefile，用来收集并更新相关的文件，在每次更改配置文件后都要进行`make collect`，各个module和metricsets配置文件在go/src/github.com/elastic/beats/metricbeat/modules.d，metricsets的配置文件在其所属module里面

```Makefile
# Runs all collection steps and updates afterwards
.PHONY: collect
collect: fields collect-docs configs kibana imports
```

执行的命令

```bash
/usr/local/soft/go/src/github.com/elastic/beats/metricbeat/build/python-env/bin/python ../metricbeat/scripts/modules_collector.py --docs_branch=master
```

#### make

make用来编译二进制文件，

```bash
go build -i -ldflags "-X github.com/elastic/beats/libbeat/version.buildTime=2018-11-07T09:33:26Z -X github.com/elastic/beats/libbeat/version.commit=b9afa344e937b086d95cea725c84a7755b384ab0"
```

## 生成文件解析

创建后会在`module/{module}/{metricset}`生成下面文件

- \{metricset}.go
- _meta/docs.asciidoc
- _meta/data.json
- _meta/fields.yml

### {metricset}.go

该文件定义了如何采集数据和输出

下面是个模板文件，该文件可以在go/src/github.com/elastic/beats/metricbeat/scripts/module/metricset找到，可知

```go
package {metricset}

import (
        "github.com/elastic/beats/libbeat/common"
        "github.com/elastic/beats/libbeat/common/cfgwarn"
        "github.com/elastic/beats/metricbeat/mb"
)

// init registers the MetricSet with the central registry as soon as the program
// starts. The New function will be called later to instantiate an instance of
// the MetricSet for each host defined in the module's configuration. After the
// MetricSet has been created then Fetch will begin to be called periodically.
func init() {
        mb.Registry.MustAddMetricSet("{module}", "{metricset}", New)
}

// MetricSet holds any configuration or state information. It must implement
// the mb.MetricSet interface. And this is best achieved by embedding
// mb.BaseMetricSet because it implements all of the required mb.MetricSet
// interface methods except for Fetch.
type MetricSet struct {
        mb.BaseMetricSet
        counter int
}

// New creates a new instance of the MetricSet. New is responsible for unpacking
// any MetricSet specific configuration options if there are any.
func New(base mb.BaseMetricSet) (mb.MetricSet, error) {
        cfgwarn.Experimental("The {module} {metricset} metricset is experimental.")

        config := struct{}{}
        if err := base.Module().UnpackConfig(&config); err != nil {
                return nil, err
        }

        return &MetricSet{
                BaseMetricSet: base,
                counter:       1,
        }, nil
}

// Fetch methods implements the data gathering and data conversion to the right
// format. It publishes the event which is then forwarded to the output. In case
// of an error set the Error field of mb.Event or simply call report.Error().
func (m *MetricSet) Fetch(report mb.ReporterV2) {
        report.Event(mb.Event{
                MetricSetFields: common.MapStr{
                        "counter": m.counter,
                },
        })
        m.counter++
}
```

模板代码执行了下面几个流程来采集数据并持久化然后返回到beat平台

#### 初始化

init()方法调用了函数mb.Registry.AddMetricSet，该函数首先把module和metricset进行中心注册，然后传入New函数变量，该New函数在导入模块之后和采集数据之前执行

```go
func init() {
        if err := mb.Registry.AddMetricSet("{module}", "{metricset}", New); err != nil {
                panic(err)
        }
}
```

#### 定义返回字段

该struct来定义变量，这些变量用来持久化采集数据和在多个采集之间配置，此struct必须要继承mb.BaseMetricSet字段，然后再添加其他的字段

```go
type MetricSet struct {
        mb.BaseMetricSet
        counter int
}
```

#### 创建实例

该New会创建一个MetricSet实例，在进行采集之前执行

```go
func New(base mb.BaseMetricSet) (mb.MetricSet, error) {

        config := struct{}{}

        if err := base.Module().UnpackConfig(&config); err != nil {
                return nil, err
        }

        return &MetricSet{
                BaseMetricSet: base,
        }, nil
}
```

#### 数据采集

Fetch()在`period`的事件间隔内去采集数据，该采集的返回是common.MapStr对象，这个对象要返回到Elasticsearch，当发生错误时也要返回，无论成功与否都要返回事件，由代码可知event就是common.MapStr对象，最终这个对象会解析成json格式数据

```go
func (m *MetricSet) Fetch() (common.MapStr, error) {

        event := common.MapStr{
                "counter": m.counter,
        }
        m.counter++

        return event, nil
}
```

#### 多次采集

相对于单次采集，多次采集Fetch()函数将会返回common.MapStr数组，对于多个事件Metricbeat会自动加上时间戳

```go
(m *MetricSet) Fetch() ([]common.MapStr, error)
```

## MetricSet相关配置

### 添加MetricSet配置选项

比如增加密码配置选项，在配置文件加入过后需要拓展New方法

首先增加配置字段

```yml
metricbeat.modules:
- module: {module}
  metricsets: ["{metricset}"]
  password: "test1234"
```

拓展New方法

首先需要在struct定义出该变量然后把struct传入方法UnpackConfig

```go
type MetricSet struct {
        mb.BaseMetricSet
        password string
}

func New(base mb.BaseMetricSet) (mb.MetricSet, error) {

        //解析出配置
        config := struct {
                Password string `config:"password"`
        }{
                Password: "",
        }
        err := base.Module().UnpackConfig(&config)
        if err != nil {
                return nil, err
        }

        return &MetricSet{
                BaseMetricSet: base,
                password:      config.Password,
        }, nil
}
```

### data.go

采集后的数据需要做数据转换创建data.go文件，该文件内主要包含eventMapping(...)方法来进行数据转换

### fields.yml

该文件这些作用

- 创建Elasticsearch模板
- 创建Kibana索引配置
- 创建field文档为metricset

### docs.asciidoc

metricset需要存在文档，文档位置module/{module}/{metricset}/_meta/docs.asciidoc

## 开发项目

根据所给目录来得出所在磁盘的分区情况，[项目源码](https://github.com/sunnoy/metricbeatk)





















