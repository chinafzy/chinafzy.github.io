---
title: "AOP Examples"
categories: spring
tags: Spring
---

[源链接](https://docs.spring.io/spring/docs/4.3.22.RELEASE/spring-framework-reference/html/aop.html){:target="_blank"}

## expression

```java
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?name-pattern(param-pattern) throws-pattern?)
```

## examples
### the execution of any public method
```java
execution(public * *(..))
```

### the execution of any method with a name beginning with "set":
```java
execution(* set*(..))
```

### the execution of any method defined by the AccountService interface:
```java
execution(* com.xyz.service.AccountService.*(..))
```

### the execution of any method defined in the service package:
```java
execution(* com.xyz.service.*.*(..))
```

### the execution of any method defined in the service package or a sub-package:
```java
execution(* com.xyz.service..*.*(..))
```

### any join point (method execution only in Spring AOP) within the service package:
```java
within(com.xyz.service.*)
```

### any join point (method execution only in Spring AOP) within the service package or a sub-package:
```java
within(com.xyz.service..*)
```

### any join point (method execution only in Spring AOP) where the proxy implements the AccountService interface:
```java
this(com.xyz.service.AccountService)
```

### any join point (method execution only in Spring AOP) where the target object implements the AccountService interface:
```java
target(com.xyz.service.AccountService)
```

### any join point (method execution only in Spring AOP) which takes a single parameter, and where the argument passed at runtime is Serializable:

```java
args(java.io.Serializable)
```

### any join point (method execution only in Spring AOP) where the target object has an @Transactional annotation:
```java
@target(org.springframework.transaction.annotation.Transactional)
```

### any join point (method execution only in Spring AOP) where the declared type of the target object has an @Transactional annotation:
```java
@within(org.springframework.transaction.annotation.Transactional)
```

### any join point (method execution only in Spring AOP) where the executing method has an @Transactional annotation:
```java
@annotation(org.springframework.transaction.annotation.Transactional)
```

### any join point (method execution only in Spring AOP) which takes a single parameter, and where the runtime type of the argument passed has the @Classified annotation:

```java
@args(com.xyz.security.Classified)
```

### any join point (method execution only in Spring AOP) on a Spring bean named tradeService:
```java
bean(tradeService)
```

### any join point (method execution only in Spring AOP) on Spring beans having names that match the wildcard expression *Service:
```java
bean(*Service)
```

