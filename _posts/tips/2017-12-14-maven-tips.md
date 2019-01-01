---
title:  "Maven Tips"
date:   2018-12-19 07:08:11 +0800
categories: tips maven
tags: maven tips
---

## Plugin execution not covered by lifecycle configuration: org.codehaus.gmaven:groovy-maven-plugin:2.0:execute in eclipse
{{ page.path }}
{% highlight xml linenos %}
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
{% endhighlight %}

``` xml
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
{% highlight xml %}
<action>
    <execute />
</action>
{% endhighlight %}

## os-maven-plugin
https://github.com/trustin/os-maven-plugin#issues-with-eclipse-m2e-or-other-ides

copy jar to `eclipse_home/dropins/`

os-maven-plugin is a Maven extension, a Maven plugin, and an Eclipse plugin

## local archetype-catalog
```
curl http://47.99.116.48:8081/repository/maven-public/archetype-catalog.xml >  ~/.m2/repository/archetype-catalog.xml
```
