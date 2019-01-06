---
title:  "Maven Tips"
date:   2018-12-19 07:08:11 +0800
categories: maven tips
tags: Maven Tips
---

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

```
add node of action
```xml
<action>
    <execute />
</action>
```

## os-maven-plugin
https://github.com/trustin/os-maven-plugin#issues-with-eclipse-m2e-or-other-ides

copy jar to `eclipse_home/dropins/`

os-maven-plugin is a Maven extension, a Maven plugin, and an Eclipse plugin

## local archetype-catalog
```
curl http://47.99.116.48:8081/repository/maven-public/archetype-catalog.xml >  ~/.m2/repository/archetype-catalog.xml
```

## run maven prjs

+ copy dependency
```bash
mvn dependency:copy-depedencies
```

+ run
```bash
java -cp classes/$(ls dependency/*.jar | awk '{printf ":"$0}') io.netty.example.proxy.HexDumpProxy
```

## maven test configuration in pom.xml
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

## architype
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
