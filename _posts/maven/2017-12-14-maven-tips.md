---
title:  "Maven Tips"
date:   2018-12-19 07:08:11 +0800
categories: maven
tags: Maven Tips
is_tip: true
---

## os-maven-plugin
这个插件是根据操作系统信息来做一些环节变量处理的，非常有用。

eclipse项目在引入时候，可能会报错，解决办法是将其作为eclipse插件导入就可以了。具体位置是`$eclipse_home/dropins/`

[官方文档](https://github.com/trustin/os-maven-plugin#issues-with-eclipse-m2e-or-other-ides){:target="_blank"}中有这句话：  
os-maven-plugin is a Maven extension, a Maven plugin, and an Eclipse plugin  
os-maven-plugin 是一个Maven扩展，一个Maven的插件，还是一个Eclipse插件

## archetype 
```bash
mvn archetype:generate \
  -DinteractiveMode=false \
  -DarchetypeCatalog=local \
  -DarchetypeGroupId=org.wtb \
  -DarchetypeArtifactId=arch-java8 \
  -DgroupId=org.wtb \
  -Dversion=1.0-SNAPSHOT \
  -DartifactId=learn-java8
```


## 拷贝本地 archetype-catalog
```
curl http://47.99.116.48:8081/repository/maven-public/archetype-catalog.xml >  ~/.m2/repository/archetype-catalog.xml
```

## 直接在命令行下跑Maven的项目

1. 拷贝依赖jar包到`target/dependency`目录下
```bash
mvn dependency:copy-dependencies
```

2. 运行命令
```bash
java -cp target/classes/$(ls target/dependency/*.jar | awk '{printf ":"$0}') io.netty.example.proxy.HexDumpProxy
```

## pom.xml中设置test默认参数 
```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <configuration>
        <argLine>
-Xms3G -Xmx3G -XX:+UseG1GC
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps
-Xloggc:/data/log/test.log
        </argLine>
      </configuration>
    </plugin>
  </plugins>
</build>
```

## 运行Maven项目，可以被eclipse远程调试
1. 首先运行这个命令
```bash
mvn -Dmaven.surefire.debug="-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=8000 -Xnoagent -Djava.compiler=NONE" test -Dtest=TestCase
```
如下面显示，Maven会开启debug模式，监听在8000端口
```bash
Listening for transport dt_socket at address: 8000
```

2. 在eclipse中，
  + 选择菜单 `Run` -> `Debug Configurations`
  + 弹出菜单里，选择`Remote Java Application`
  + 添加一个调试条目。注意填写端口号和ip地址，以及选择正确的项目（跟远程程序一致）
  + 点击`Debug`按钮，开始debug。Eclipse将连接远程地址，成功后远程debug程序继续执行。


## Plugin execution not covered by lifecycle configuration: org.codehaus.gmaven:groovy-maven-plugin:2.0:execute in eclipse
```xml
<plugin>
    <groupId>org.eclipse.m2e</groupId>
    <artifactId>lifecycle-mapping</artifactId>
    <version>1.0.0</version>
    <configuration>
        <lifecycleMappingMetadata>
            <pluginExecutions>
                <pluginExecution>
                    <pluginExecutionFilter>
                        <groupId>org.codehaus.gmaven</groupId>
                        <artifactId>groovy-maven-plugin</artifactId>
                        <versionRange>[2.0,)</versionRange>
                        <goals>
                            <goal>execute</goal>
                        </goals>
                    </pluginExecutionFilter>
                    <action>
                        <execute />
                    </action>
                </pluginExecution>
            </pluginExecutions>
        </lifecycleMappingMetadata>
    </configuration>
</plugin>
```

add node of action
```xml
<action>
    <execute />
</action>
```
