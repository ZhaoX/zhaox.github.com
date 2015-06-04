---
layout: post
title: "Java序列化中的serialVersionUID有什么用？"
description: ""
category: Java 
tags: [Java, Serialization]
---
{% include JB/setup %}

如果一个实现了Serializable的类没有serialVersionUID属性，IDE(比如Eclipse)通常会报这样一个warning:

> The serializable class Foo does not declare a static final
> serialVersionUID field of type long

那这个serialVersionUID是做什么用的呢？可以看看JDK中Serializable接口的注释。

> The serialization runtime associates with each serializable class a
> version  number, called a serialVersionUID, which is used during
> deserialization to  verify that the sender and receiver of a
> serialized object have loaded  classes for that object that are
> compatible with respect to serialization.  If the receiver has loaded
> a class for the object that has a different  serialVersionUID than
> that of the corresponding sender's class, then  deserialization will
> result in an {@link InvalidClassException}.  A  serializable class can
> declare its own serialVersionUID explicitly by  declaring a field
> named <code>"serialVersionUID"</code> that must be static,  final, and
> of type <code>long</code>:<p>
> 
>  serialVersionUID 表示可序列化类的版本，在反序列化对象时，用来确认序列化与反序列化该对象所使用的类的版本是否兼容。 
> 如果类的版本不一致，那么反序列化将不能正常进行，抛出InvalidClassException。
> 
>  If a serializable class does not explicitly declare a
> serialVersionUID, then  the serialization runtime will calculate a
> default serialVersionUID value  for that class based on various
> aspects of the class, as described in the  Java(TM) Object
> Serialization Specification.  However, it is <em>strongly 
> recommended</em> that all serializable classes explicitly declare 
> serialVersionUID values, since the default serialVersionUID
> computation is  highly sensitive to class details that may vary
> depending on compiler  implementations, and can thus result in
> unexpected  <code>InvalidClassException</code>s during
> deserialization.  Therefore, to  guarantee a consistent
> serialVersionUID value across different java compiler 
> implementations, a serializable class must declare an explicit 
> serialVersionUID value.  It is also strongly advised that explicit 
> serialVersionUID declarations use the <code>private</code> modifier
> where  possible, since such declarations apply only to the immediately
> declaring  class--serialVersionUID fields are not useful as inherited
> members. Array  classes cannot declare an explicit serialVersionUID,
> so they always have  the default computed value, but the requirement
> for matching  serialVersionUID values is waived for array classes.
> 
>  如果一个可序列化的类没有包含serialVersionUID，运行时会根据这个类的特征自动计算出一个serialVersionUID。 
> 那么，为什么不能用默认的这个实现呢，似乎更省事?
> 因为不同的编译器实现会导致同一个类的源代码文件，被计算出不同的serialVersionUID.

StackOverflow上有一个类似的问题：
http://stackoverflow.com/questions/285793/what-is-a-serialversionuid-and-why-should-i-use-it
点击下面这个链接，可以查看更多关于Serializable接口的信息：
https://github.com/ZhaoX/jdk-1.7-annotated/blob/master/src/java/io/Serializable.java

