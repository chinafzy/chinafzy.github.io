---
title: "logback xml sample"
categories: java
tags: Java
---

## logback.xml推荐配置
这里是个人使用的logback配置。
+ logpath：设置日志文件的存储路径 
+ patt_*：日志的格式
  + patt_debug：开发调试环境下使用，它最大的优势是在eclipse的控制台上打印出来的日志，有明显的代码行号跳转链接，鼠标点击后可以直接跳到代码位置。
  + patt_pro：生产环境配置，也带有【代码运行位置】。  
  注意，【代码运行位置】特性会带来性能上的损耗，在高并发下这个影响比较明显。但是一般人都不会遇到很多高并发，一般建议加上，因为对于排查问题很有用。
  + patt_fast：生产环境配置的高速版，去除了【代码运行位置】
  + patt_simple：只打印消息本身，适用于做记录业务信息。
+ append建议开启sync模式，一般是有益无害。
+ 在刚接手项目时候，一定要把Spring的`RequestMappingHandlerMapping`和`AutoConfigurationReportLoggingInitializer`的日志打开，这里坑巨多。  
如果用了数据库，也最好把Spring的数据库事务日志打开，因为你不知道别人会怎么配置的。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="logback.xsd">

  <property name="logpath" value="logs" />

  <property name="patt_pro" value="%d{yyyMMdd HH:mm:ss.SSS} [%-1thread] [%-5level] %logger{1}.%M %L - %msg%n" />
  <property name="patt_debug" value="%d{yyyMMdd HH:mm:ss.SSS} [%-1thread] [%-5level] %logger.%M\\(%F:%L\\) - %msg%n" />
  <property name="patt_fast" value="%d{yyyMMdd HH:mm:ss.SSS} [%-1thread] [%-5level] %logger{1} - %msg%n" />
  <property name="patt_simple" value="%msg%n" />
  <property name="patt" value="${patt_debug}" />

  <!-- <logger name="org.springframework" additivity="true" level="DEBUG" /> -->
  <!-- <logger name="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping" additivity="true" level="INFO" /> -->
  <!-- <logger name="org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer" additivity="true" level="INFO" /> -->

  <appender name="std" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>${patt}</pattern>
    </encoder>
  </appender>

  <appender name="keepalive_file" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${logpath}/keepalive.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>${logpath}/keepalive.log.%d{yyyyMMdd.HH}</fileNamePattern>
      <maxHistory>10000</maxHistory>
    </rollingPolicy>
    <encoder>
      <pattern>${patt}</pattern>
    </encoder>
  </appender>
  <appender name="keepalive_async" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>5000</queueSize>
    <appender-ref ref="keepalive_file" />
  </appender>

  <appender name="root_file" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${logpath}/root.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>${logpath}/root.log.%d{yyyyMMdd.HH}</fileNamePattern>
      <maxHistory>10000</maxHistory>
    </rollingPolicy>
    <encoder>
      <pattern>${patt}</pattern>
    </encoder>
  </appender>
  <appender name="root_async" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>5000</queueSize>
    <appender-ref ref="root_file" />
  </appender>

  <root level="INFO">
    <appender-ref ref="std" />
    <!-- <appender-ref ref="root_async" /> -->
  </root>

</configuration>
```

## logback.xsd
这个不是官方推荐的，但是使用之后对于编辑logback.xml非常方便。  

使用方法：放在logback.xml一起。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema" elementFormDefault="qualified">
  <xsd:element name="configuration" type="Configuration" />
  <xsd:element name="included" type="Configuration" />

  <xsd:complexType name="Configuration">
    <xsd:choice maxOccurs="unbounded">
      <xsd:element name="statusListener" minOccurs="0" maxOccurs="unbounded" type="StatusListener" />
      <xsd:element name="contextListener" minOccurs="0" maxOccurs="unbounded" type="ContextListener" />
      <xsd:element name="include" minOccurs="0" maxOccurs="unbounded" type="Include" />
      <xsd:element name="contextName" minOccurs="0" maxOccurs="1" type="xsd:string" />
      <xsd:element name="define" minOccurs="0" maxOccurs="unbounded" type="Define" />
      <xsd:element name="timestamp" minOccurs="0" maxOccurs="1" type="Timestamp" />
      <xsd:element name="if" minOccurs="0" maxOccurs="unbounded" type="If" />
      <xsd:element name="property" minOccurs="0" maxOccurs="unbounded" type="Property" />
      <xsd:element name="appender" minOccurs="0" maxOccurs="unbounded" type="Appender" />
      <xsd:element name="logger" minOccurs="0" maxOccurs="unbounded" type="Logger" />
      <xsd:element name="root" minOccurs="0" maxOccurs="1" type="Root" />
      <xsd:any namespace="##other" processContents="lax" minOccurs="0" maxOccurs="unbounded" />
    </xsd:choice>
    <xsd:attribute name="debug" type="xsd:boolean" use="optional" />
    <xsd:attribute name="scan" type="xsd:string" use="optional" />
    <xsd:attribute name="scanPeriod" type="xsd:string" use="optional" />
    <xsd:anyAttribute />
  </xsd:complexType>

  <xsd:complexType name="StatusListener">
    <xsd:attribute name="class" type="xsd:string" use="optional" />
    <xsd:anyAttribute />
  </xsd:complexType>

  <xsd:complexType name="ContextListener">
    <xsd:choice maxOccurs="unbounded">
      <xsd:element name="resetJUL" type="xsd:boolean" />
      <xsd:any namespace="##other" processContents="lax" minOccurs="0" maxOccurs="unbounded" />
    </xsd:choice>
    <xsd:attribute name="class" type="xsd:string" />
    <xsd:anyAttribute />
  </xsd:complexType>

  <xsd:complexType name="Include">
    <xsd:attribute name="optional" use="optional" type="xsd:boolean" />
    <xsd:attribute name="file" use="optional" type="xsd:string" />
    <xsd:attribute name="resource" use="optional" type="xsd:string" />
    <xsd:attribute name="url" use="optional" type="xsd:string" />
  </xsd:complexType>

  <xsd:complexType name="Define">
    <xsd:sequence>
      <xsd:any minOccurs="0" maxOccurs="unbounded" />
    </xsd:sequence>
    <xsd:anyAttribute />
  </xsd:complexType>

  <xsd:complexType name="Timestamp">
    <xsd:sequence>
      <xsd:any minOccurs="0" maxOccurs="unbounded" />
    </xsd:sequence>
    <xsd:attribute name="key" type="xsd:string" use="optional" />
    <xsd:attribute name="datePattern" type="xsd:string" use="optional" />
    <xsd:attribute name="timeReference" type="xsd:string" use="optional" />
    <xsd:anyAttribute />
  </xsd:complexType>

  <xsd:complexType name="RollingPolicy">
    <xsd:choice maxOccurs="unbounded">
      <xsd:element name="fileNamePattern" minOccurs="1" maxOccurs="1" type="xsd:string" />
      <xsd:element name="maxHistory" minOccurs="0" maxOccurs="1" type="xsd:int" />
      <xsd:element name="minIndex" minOccurs="0" maxOccurs="1" type="xsd:int" />
      <xsd:element name="maxIndex" minOccurs="0" maxOccurs="1" type="xsd:int" />
      <xsd:element name="timeBasedFileNamingAndTriggeringPolicy" minOccurs="0" maxOccurs="1" type="TriggeringPolicy" />
      <xsd:element name="cleanHistoryOnStart" minOccurs="0" maxOccurs="1" type="xsd:boolean" />
    </xsd:choice>
    <xsd:attribute name="class" type="xsd:string" use="required" />
  </xsd:complexType>
  <xsd:complexType name="TriggeringPolicy">
    <xsd:sequence>
      <xsd:element name="maxFileSize" minOccurs="0" maxOccurs="1" type="xsd:string" />
    </xsd:sequence>
    <xsd:attribute name="class" type="xsd:string" use="required" />
  </xsd:complexType>

  <xsd:complexType name="If">
    <xsd:choice maxOccurs="unbounded">
      <xsd:element type="Configuration" name="then" />
      <xsd:element type="Configuration" name="else" />
      <xsd:any namespace="##other" processContents="lax" minOccurs="0" maxOccurs="unbounded" />
    </xsd:choice>
    <xsd:attribute name="condition" type="xsd:string" />
    <xsd:anyAttribute />
  </xsd:complexType>

  <xsd:complexType name="Property">
    <xsd:sequence>
      <xsd:any minOccurs="0" maxOccurs="unbounded" />
    </xsd:sequence>
    <xsd:attribute name="name" type="xsd:string" use="optional" />
    <xsd:attribute name="value" type="xsd:string" use="optional" />
    <xsd:attribute name="file" type="xsd:string" use="optional" />
    <xsd:attribute name="resource" type="xsd:string" use="optional" />
    <xsd:anyAttribute />
  </xsd:complexType>

  <xsd:complexType name="Appender">
    <xsd:choice maxOccurs="unbounded">
      <xsd:element name="file" minOccurs="0" maxOccurs="1" type="xsd:string" />
      <xsd:element name="encoder" minOccurs="0" maxOccurs="1" type="Encoder" />
      <xsd:element name="filter" minOccurs="0" maxOccurs="1" type="Filter" />
      <xsd:element name="append" minOccurs="0" maxOccurs="1" type="xsd:boolean" />
      <xsd:element name="prudent" minOccurs="0" maxOccurs="1" type="xsd:boolean" />
      <xsd:element name="layout" minOccurs="0" maxOccurs="1" type="Layout" />
      <xsd:element name="rollingPolicy" minOccurs="0" maxOccurs="1" type="RollingPolicy" />
      <xsd:any namespace="##other" processContents="skip" minOccurs="0" maxOccurs="unbounded" />
    </xsd:choice>
    <xsd:attribute name="name" type="xsd:string" use="required" />
    <xsd:attribute name="class" type="xsd:string" use="required" />
    <xsd:anyAttribute />
  </xsd:complexType>

  <xsd:complexType name="Filter">
    <xsd:choice maxOccurs="unbounded">
      <xsd:element name="level" minOccurs="0" maxOccurs="unbounded" type="xsd:string" />
      <xsd:any namespace="##other" processContents="lax" minOccurs="0" maxOccurs="unbounded" />
    </xsd:choice>
    <xsd:attribute name="class" type="xsd:string" use="optional" />
    <xsd:anyAttribute />
  </xsd:complexType>

  <xsd:complexType name="Layout">
    <xsd:choice maxOccurs="unbounded">
      <xsd:element name="pattern" minOccurs="0" maxOccurs="1" type="xsd:string" />
      <xsd:any namespace="##other" processContents="lax" minOccurs="0" maxOccurs="unbounded" />
    </xsd:choice>
    <xsd:attribute name="class" type="xsd:string" use="optional" />
    <xsd:anyAttribute />
  </xsd:complexType>

  <xsd:complexType name="Logger">
    <xsd:choice maxOccurs="unbounded">
      <xsd:element name="appender-ref" minOccurs="0" maxOccurs="unbounded" type="AppenderRef" />
      <xsd:any namespace="##other" processContents="lax" minOccurs="0" maxOccurs="unbounded" />
    </xsd:choice>
    <xsd:attribute name="name" type="xsd:string" use="required" />
    <xsd:attribute name="level" type="LoggerLevels" use="optional" />
    <xsd:attribute name="additivity" type="xsd:boolean" use="optional" />
    <xsd:anyAttribute />
  </xsd:complexType>

  <xsd:complexType name="Encoder">
    <xsd:choice maxOccurs="unbounded">
      <xsd:element name="pattern" type="xsd:string" minOccurs="0" maxOccurs="1" />
      <xsd:element name="layout" minOccurs="0" maxOccurs="1" type="Layout" />
      <xsd:element name="immediateFlush" type="xsd:boolean" minOccurs="0" maxOccurs="1" />
      <xsd:element name="outputPatternAsHeader" type="xsd:boolean" minOccurs="0" maxOccurs="1" />
      <xsd:element name="charset" type="xsd:string" minOccurs="0" maxOccurs="1" />
    </xsd:choice>
    <xsd:attribute name="class" type="xsd:string" use="optional" />
  </xsd:complexType>

  <xsd:complexType name="Root">
    <xsd:sequence>
      <xsd:element name="appender-ref" minOccurs="0" maxOccurs="unbounded" type="AppenderRef" />
    </xsd:sequence>
    <xsd:attribute name="level" use="required" type="LoggerLevels" />
    <xsd:anyAttribute />
  </xsd:complexType>

  <xsd:complexType name="AppenderRef">
    <xsd:attribute name="ref" type="xsd:string" />
  </xsd:complexType>

  <xsd:simpleType name="LoggerLevels">
    <xsd:union>
      <xsd:simpleType>
        <xsd:restriction base="xsd:string">
          <xsd:enumeration value="OFF" />
          <xsd:enumeration value="ALL" />
          <xsd:enumeration value="INHERITED" />
          <xsd:enumeration value="NULL" />
          <xsd:enumeration value="ERROR" />
          <xsd:enumeration value="WARN" />
          <xsd:enumeration value="INFO" />
          <xsd:enumeration value="DEBUG" />
          <xsd:enumeration value="TRACE" />
        </xsd:restriction>
      </xsd:simpleType>
      <xsd:simpleType>
        <xsd:restriction base="xsd:string">
          <xsd:pattern value="($\{.+:-)?[Oo][Ff]{2}\}?" />
          <xsd:pattern value="($\{.+:-)?[Aa][Ll]{2}\}?" />
          <xsd:pattern value="($\{.+:-)?[Ii][Nn][Hh][Ee][Rr][Ii][Tt][Ee][Dd]\}?" />
          <xsd:pattern value="($\{.+:-)?[Nn][Uu][Ll]{2}\}?" />
          <xsd:pattern value="($\{.+:-)?[Ee][Rr]{2}[Oo][Rr]\}?" />
          <xsd:pattern value="($\{.+:-)?[Ww][Aa][Rr][Nn]\}?" />
          <xsd:pattern value="($\{.+:-)?[Ii][Nn][Ff][Oo]\}?" />
          <xsd:pattern value="($\{.+:-)?[Dd][Ee][Bb][Uu][Gg]\}?" />
          <xsd:pattern value="($\{.+:-)?[Tt][Rr][Aa][Cc][Ee]\}?" />
        </xsd:restriction>
      </xsd:simpleType>
    </xsd:union>
  </xsd:simpleType>

  <xsd:complexType name="JmxConfigurator" />
</xsd:schema>

```
