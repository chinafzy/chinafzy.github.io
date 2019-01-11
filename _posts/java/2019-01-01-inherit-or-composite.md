---
title: 继承还是聚合，设计模式的取舍？
categories: java
tags: Java Pattern
---

## 从JDK的一个经典bug说起继承(inherit)的不足

JDK实现了观察者设计模式，
+ `java.util.Observable`：可被观察的对象，主要函数
  + #addObserver()/deleteObserver()：增删观察者
  + #notifyObservers()：通知观察者数据变动
+ `java.util.Observer`：观察调用的callback，函数
  + #update(Observable observable, Object obj)

假设我要做一个UI类，在内部资源内容发生变动时候，需要`通知`别的模块，观察者模式是最好的选择。  
我们期望的代码是这样的：
```java
class MgrView extends View {
    private Observable monitor = new Observable();
    
    public Mgr(Observer callback){
        monitor.addObserver(callback);
    }

    public void do1(Object res){
        // ... 处理业务 ...
        
        // 处理完了后发送通知 
        monitor.update(res);
    }
}
```

但是我们在查看JDK自带的包，发现了一个问题：  
Observable中的关键代码：控制Observable是否启用的`#setChanged()`/`#clearChanged()`，可见性是`protected`。  
因此要使用这个类，只能`继承`。

从概念上说：我的类的基础特性是UI View；观察者模式只是它的一个分支特性，不是根本特性。  
因此，继承Observable就是一个错误的方式。

但是，JDK的观察者模式默认了是继承模式。这个给我们的使用带来了很大的麻烦。是一个完全错误的考虑。

从技术层面来看，它后续还不能把那个`protected`升级为`public`，因为这样的话，现有的子类如果有protected @Override这个方法的，就会出错了。

## 简单结论：何时适合做继承或者组合
从上面的例子延伸考虑：
+ UIView是我们的组件的主要特性，组件“是一个(IsA)”UIView  
+ 组件还有可观察的次要属性，组件“有一个(HasA)”Observable   

把一个HasA的特性变成了IsA，就会让组件多出来很多莫名其妙的功能。

这个是最简单也是最基础的分辨。但是有时候可以模糊执行：  
**如果没有IsA，而且这个需求没有后续发展，HasA可以冒充IsA来使用**。

例如：要做一个临时的小工具，这个工具不是基于别的工具的基础上来做，而有“可观察”特性。这个时候，就可以直接继承Observable，也告诉使用的人，里面其他的函数别管了。

## 从SpringBoot的一个例子
SpringBoot里面可以很方便的配置对redis的集成，但是美中不足的是：  
**如果我们要同时访问多个redis，那就麻烦了**。

扩展的方式在别的地方有写，这里只是讨论这个设计的得失：

一个应用，应该是有一个多个redis数据源，而不是只有一个。在spring-boot的设计中，很明显的感觉到，他把数据源的集成做成了IsA，而不是HasA。

