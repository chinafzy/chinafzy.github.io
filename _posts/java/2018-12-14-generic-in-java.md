---
title:  "Java中的泛型"
date:   2018-12-14 17:08:11 +0800
tags: Java Program
categories: java
---

## 前言

看了很多解释泛型的，很少有从“作用”的角度来解释它。

**注意：这里的讨论，都是从实用的角度出发，讨论语言应该的表现。**

## 怎么选择extends或者super
JDK自带范例`java.util.Collections`，就完全可以说明，选择extends或者super的时机

```java
public static <T> void copy (List<? super T> dest, List<? extends T> src) {
    int srcSize = src.size();
    if (srcSize > dest.size())
        throw new IndexOutOfBoundsException("Source does not fit in dest");

    if (srcSize < COPY_THRESHOLD ||
        (src instanceof RandomAccess && dest instanceof RandomAccess)) {
        for (int i=0; i<srcSize; i++)
            dest.set(i, src.get(i));
    } else {
        ListIterator<? super T> di=dest.listIterator();
        ListIterator<? extends T> si=src.listIterator();
        for (int i=0; i<srcSize; i++) {
            di.next();
            di.set(si.next());
        }
    }
}
```
+ src列表，在函数体内，只用来提供元素给别人使用，它是生产者。它需要关注的是，自己提供的元素，不能超过一个类型范围。也就是说，需要限定自己的类型的上限（upper bound）。<? extends T>就约定了，每个元素是T或它的子孙类型，不超出生产的需求；
+ 而dest列表，在函数体内，只用来接受元素，它是消费者。这个列表，必须要“至少”允许T可以加入到列表中，也就是说，这个列表的元素类型的下限（lower bound）是T。<? super T>就约定了，这个列表“至少”允许加入T。

Effective In Java总结为PECS（ProducerExtends ConsumerSuper）。
- 生产者：要求upper bound，即extends
- 消费者：要求lower bound，即super

如果代码同时对这个列表有读和写的行为，也就是说这个列表同时作为生产者和消费者，那自然就是super和extends都不选择，直接List\<T>。

**小结：从生产消费者的角度来看，PECS。**

如果不能理解，再尝试俩次，如果还不能理解，继续看下面的sample。

## 错误的数组协变（arrays covariant）
举另外一个例子，java里面的数组协变问题。这个协变是支持向上的，也就是说Number[]可以变成Object[]，但是不能变成Integer[]。
```java
Integer[] nums = new Integer[]{1};
Object[] objs = nums;
Object obj1 = objs[0];
objs[0] = "string";
```
这个代码是可以编译运行的，但是运行到第四行的时候，会出`ClassCastException`。

我们来分析下：
+ 第三行是生产行为，向上的协变，没有问题。
+ 第四行是消费行为，向上的协变，导致了消费行为里面的“类型发散”问题不能被预先检查出来。

参考上面章节的说法：消费行为应该有个约束：“至少是某个类型”，也就是lower bound。

**小结：数组协变是个错误而且脑残的设定，这个特性应该被放弃，而且作为一个典型错误永远保留在教科书里面。**


## 泛型和协变
用generic来重写上面的case吧。标记#A#B的地方，我们绕过了泛型检测：
```java
List<Number> nums = new ArrayList<Number>();
nums.add(new Float(1));

 //
// #A, transform to Lower
@SuppressWarnings({ "rawtypes" , "unchecked" })
List<Integer> ints = (List) nums;
// ERROR: As producer, it might produce an unexpected type here.
Integer i1 = ints.get(0);
// As consumer, it can only accept the right data type.
ints.add(1);
// OK.
Number n1 = nums.get(nums.size() - 1);

//
// #B, transform to upper
@SuppressWarnings({ "rawtypes" , "unchecked" })
List<Object> objs = (List) nums;
// As producer, it can produce the right data type.
Object obj1 = objs.get(0);
// ERROR: As consumer, it accepts a wrong data type.
objs.add( "string");
// Validation fails.
Number n2 = nums.get(nums.size() - 1);
```

一眼就可以看出：
+ #A（向下转换）：接受数据（消费）没问题；提供数据（生产）出错了。实际生产的是一个Float，但是预期是Integer。
+ #B（向上转换）：提供数据（生产）没问题；接受数据（消费）出错了。添加了一个String进去，但是我们对这个列表的最初期望是Number，破坏了约束，导致了后面的错误。
222

**小结：泛型不支持协变。因为向下会有生产问题，向上会有消费问题。**

**BTW：** 从这个sample里面可以看出来，泛型的作用，就是约束类型。也可以看出来，@SuppressWarnings是一个有害的技巧，特别是它支持如此多的范围（TYPE , FIELD, METHOD, PARAMETER, CONSTRUCTOR, LOCAL_VARIABLE）。简单来说，如果你在一个Method上申明了这个，那Method里面一切同类型warning都会被ignore，如果在类的级别上申明，那全部都可以忽略了。@SuppressWarnings的作用范围的扩散，是技术逻辑的让步，是sb们的胜利；而@SuppressWarnings的滥用，是sb们的狂欢。也许，让它扩大到PACKAGE范围，sb们会更高兴。

## bound约束
再看另外一个sample：
```java
List<Number> nums = new ArrayList<Number>();

List<? extends Number> unknownNums = nums;
// #a
unknownNums.add( null);
Number num3 = unknownNums.get(0);

List<? super Number> unknowns = nums;
unknowns.add( new Float(1));
// #b
Object o = unknowns.get(0);
```

这个范例里面比较好玩的：
+ unknownNums只能接受null，因为列表的元素的类型是“Number的某一个不确定的子类型”，但是不是任何一个“确定的”类型，所以什么都不满足这个条件，反而是null，什么类型都不是，可以满足。<? extends>在消费行为里只能接受null，这样就没意义了。
+ unknowns读取元素，是Number的“某一个不确定的祖先类型”，但是不是任何一个“确定的”类型，所以只能用java里面的根类型Object来表述。<? super> 在生产行为下只能提供Object类型，这样的泛型也就没了意义。

**小结：PECS倒过来，就没有任何意义了。**


## 总结
其实前面的四个小节都是在重复一个道理：PECS的原因。
