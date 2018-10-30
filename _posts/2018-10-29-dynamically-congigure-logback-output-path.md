---
layout: post
title: "logback动态定义日志输出路径"
date: 2018-10-29 18:05:06
categories: log
---
生产环境错综复杂，往往开发环境下一切正常，发布到生产环境却出现始料未及的问题，使用logback记录系统运行日志这一基础功能也不例外。这里就记录一次由于日志输出路径配置不当而引起的问题。 <!-- excerpt -->

首先描述一下问题：生产环境使用一个IP（一台主机）下运行3个weblogic容器分别监听不同端口，应用部署在weblogic中。日志输出使用绝对路径，配置如下：

```xml
<property name="LOG_HOME" value="/websoftware/filesys/logs"/>
```

至此，你可能已经想到即将出现的问题了，多JVM同时操作一个文件……我们发现这样做的现象是日志里永远只有一个端口下的系统日志信息。于是开始解决问题……

#### 1、logback是支持多JVM同时操作一个日志文件的

```xml
    <appender name="sysLogAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 支持多JVM同时操作同一个日志文件 -->
        <prudent>true</prudent> 
        <file>${LOG_HOME}/sys.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            ...
        </rollingPolicy>
        <encoder>
            ...
        </encoder>
    </appender>
``` 

如上面配置，使用`<prudent>true</prudent>`即可支持多JVM同时操作一个日志文件，官方给的说明如下：

    如果使用prudent模式，FileAppender将安全的写入到指定文件，即使存在运行在不同机器上的、其他JVM中运行的其他FileAppender实例。Prudent模式更依赖于排他文件锁，经验表明加了文件锁后，写日志的开始是正常的3倍以上。当prudent模式关闭时，每秒logging event的吞吐量为100,000，当prudent模式开启时，大约为每秒33,000。

除了性能问题，使用`prudent`模式还可能出现日志不回滚、不同端口下日志掺杂在一起降低可读性等问题。

#### 2、使用相对路径输出日志

既然使用绝对路径造成多JVM操作同一文件，那使用相对路径各写各的不就可以了！修改`LOG_HOME`为：`<property name="LOG_HOME" value="logs"/>`搞定。

但是，这样日志文件就保存在程序运行目录下了，如果你对系统日志放在服务器bin目录下没感到不爽（当然你可以使用`../`将它放到想要的位置），或者愿意忍受不同运行环境下日志输出路径的差异，最重要的你们生产环境下没有对日志输出位置的统一要求，那么使用相对路径完全可以解决。

#### 3、动态定义日志的输出路径

本文重点！该生产环境就是有统一的日志存放目录，那怎么办？！如果每个端口的应用将日志输出到日志存放目录下的不同子目录，或者生成不同名称的日志文件问题自然解决。但是怎么方便地实现呢？

所幸，logback提供`PropertyDefiner`接口，在配置文件中使用`<define>`标签，指定接口`PropertyDfiner`对应的实现类，即可在运行时定义变量。

在logback中定义一个变量，使用该变量表示当前服务的端口号：

```xml
<define name="PORT_DIR" class="com.xxx.log.PortDir" />
```

实现类`PortDir`代码如下（不直接实现`PropertyDefiner`，而是继承logback提供的已经实现`PropertyDefiner`的抽象类`PropertyDefinerBase`）:

```java
package com.xxx.log;

import ch.qos.logback.core.PropertyDefinerBase;
import com.xxx.util.SysUtil;

public class PortDir extends PropertyDefinerBase {
    @Override
    public String getPropertyValue() {
        String port = SysUtil.getPort();
        return port == null ? "0" : port;
    }
}
```

获取本地服务端口的方法：

```java
    /**
     * 获取本地Weblogic的端口
     */
    public static String getPort() throws NamingException, MalformedObjectNameException,
            AttributeNotFoundException, MBeanException, ReflectionException, InstanceNotFoundException {
        Context ctx = new InitialContext();
        MBeanServer tMBeanServer = (MBeanServer) ctx.lookup("java:comp/env/jmx/runtime");
        ObjectName tObjectName = new ObjectName("com.bea:Name=RuntimeService,Type=weblogic.management.mbeanservers.runtime.RuntimeServiceMBean");
        ObjectName serverrt = (ObjectName) tMBeanServer.getAttribute(tObjectName, "ServerRuntime");
        String port = String.valueOf(tMBeanServer.getAttribute(serverrt, "ListenPort"));
        return port;
    }
```

这样，logback配置文件中即可使用`${PORT_DIR}`来获取当前服务的端口号了，`LOG_HOME`修改如下：

```xml
<property name="LOG_HOME" value="/websoftware/filesys/logs/p_${PORT_DIR}"/>
```

问题解决,生产环境日志输出如下：

```
[xxx@xxx /websoftware/filesys/logs]$ls
p_8013  p_8015  p_8017
```

#### 附、自定义转换器

logback中提供多种转换器，比如：%d表示日期，%thread表示线程名，%msg：日志消息，%n是换行符等等。同时，logback允许自定义转换器：

自定义一个获取当前端口号的转换器，继承`ClassicConverter`:

```java
package com.xxx.log;

import ch.qos.logback.core.PropertyDefinerBase;
import com.xxx.util.SysUtil;
public class PortConverter extends ClassicConverter {

    @Override
    public String convert(ILoggingEvent iLoggingEvent) {
        String port = SysUtil.getPort();
        return port == null ? "0" : port;
    }
}
```

在`logback.xml`中注册：

```xml
<conversionRule conversionWord="port" converterClass="com.xxx.log.PortConverter" />
```

这样就可以在Pattern中使用`%port`打印出本地端口号：

```xml
<appender name="sysLogAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
    ...
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        ...
    </rollingPolicy>
    <encoder>
        ...
        <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %port [%thread] %-5level %logger{50} - %msg%n</Pattern>
    </encoder>
</appender>
```