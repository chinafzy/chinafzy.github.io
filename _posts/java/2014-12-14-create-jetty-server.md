---
title: "创建简便的jetty server"
categories: java
tags: Java
---

## 创建一个maven项目

## 添加maven dependencies
```xml
<dependency>
  <groupId>org.eclipse.jetty</groupId>
  <artifactId>jetty-server</artifactId>
  <version>9.2.0.M1</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.eclipse.jetty</groupId>
  <artifactId>jetty-servlet</artifactId>
  <version>9.2.0.M1</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.eclipse.jetty</groupId>
  <artifactId>jetty-webapp</artifactId>
  <version>9.2.0.M1</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.glassfish.web</groupId>
  <artifactId>javax.servlet.jsp</artifactId>
  <version>${glassfish.javax.version}</version>
  <scope>test</scope>
</dependency>

<dependency>
  <groupId>org.glassfish.web</groupId>
  <artifactId>jsp-impl</artifactId>
  <version>${glassfish.javax-impl.version}</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>javax.servlet.jsp.jstl</groupId>
  <artifactId>jstl-api</artifactId>
  <version>1.2</version>
  <exclusions>
    <exclusion>
      <groupId>javax.servlet</groupId>
      <artifactId>servlet-api</artifactId>
    </exclusion>
  </exclusions>
</dependency>
<dependency>
  <groupId>org.glassfish.web</groupId>
  <artifactId>jstl-impl</artifactId>
  <version>1.2</version>
  <exclusions>
    <exclusion>
      <groupId>javax.servlet</groupId>
      <artifactId>servlet-api</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

## 编写测试代码
```java
package org.wtb.demo.integration;

import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.servlet.ServletHolder;
import org.eclipse.jetty.webapp.WebAppContext;
import org.springframework.web.servlet.DispatcherServlet;

public class TestJettyProgrammatic {

    public static void main(String[] args) throws Exception {
        Server server = createServer();

        try {
            server.start();

            Thread.currentThread().join();
        } finally {
            server.stop();
        }
    }

    private static Server createServer() {
        Server server = new Server(8888);

        WebAppContext jsp = new WebAppContext("src/main/webapp/", "/");
        server.setHandler(jsp);
        DispatcherServlet dispatcherServlet = new DispatcherServlet();
        jsp.addServlet(new ServletHolder(dispatcherServlet), "*.html");
        dispatcherServlet.setContextConfigLocation("classpath*:mvc.xml");
        // dispatcherServlet.set

        return server;
    }
}
```
