---
title: 'Java序列化有什么作用？序列化与不序列化有什么区别？'
description: '深入理解Java序列化机制，包括Serializable接口、serialVersionUID、transient关键字等核心概念'
pubDate: '2021-03-19'
updatedDate: '2021-03-23'
tags: ['序列化']
category: 'Java'
---

## 序列化是干啥用的？

序列化的原本意图是希望对一个Java对象作一下"变换"，变成字节序列，这样一来方便持久化存储到磁盘，避免程序运行结束后对象就从内存里消失，另外变换成字节序列也更便于网络运输和传播[^1]，所以概念上很好理解：

- **序列化**：把Java对象转换为字节序列。
- **反序列化**：把字节序列恢复为原先的Java对象。

[^1]: Java序列化机制自JDK 1.1版本引入，是Java远程方法调用（RMI）和Java Bean持久化的基础。

而且序列化机制从某种意义上来说也弥补了平台化的一些差异，毕竟转换后的字节流可以在其他平台上进行反序列化来恢复对象。

---

## 对象如何序列化？

然而Java目前并没有一个关键字可以直接去定义一个所谓的"可持久化"对象。

对象的持久化和反持久化需要靠程序员在代码里手动**显式地**进行序列化和反序列化还原的动作。

举个例子，假如我们要对`Student`类对象序列化到一个名为`student.txt`的文本文件中，然后再通过文本文件反序列化成`Student`类对象：

### 1、Student类定义

```java
public class Student implements Serializable {

    private String name;
    private Integer age;
    private Integer score;
    
    @Override
    public String toString() {
        return "Student:" + '\n' +
        "name = " + this.name + '\n' +
        "age = " + this.age + '\n' +
        "score = " + this.score + '\n'
        ;
    }
    
    // ... 其他省略 ...
}
```

### 2、序列化

```java
public static void serialize() throws IOException {

    Student student = new Student();
    student.setName("CodeSheep");
    student.setAge( 18 );
    student.setScore( 1000 );

    ObjectOutputStream objectOutputStream = 
        new ObjectOutputStream( new FileOutputStream( new File("student.txt") ) );
    objectOutputStream.writeObject( student );
    objectOutputStream.close();
    
    System.out.println("序列化成功！已经生成student.txt文件");
    System.out.println("==============================================");
}
```

### 3、反序列化

```java
public static void deserialize() throws IOException, ClassNotFoundException {
    ObjectInputStream objectInputStream = 
        new ObjectInputStream( new FileInputStream( new File("student.txt") ) );
    Student student = (Student) objectInputStream.readObject();
    objectInputStream.close();
    
    System.out.println("反序列化结果为：");
    System.out.println( student );
}
```

### 4、运行结果

```
序列化成功！已经生成student.txt文件
==============================================
反序列化结果为：
Student:
name = CodeSheep
age = 18
score = 1000
```

---

## Serializable接口有何用？

上面在定义`Student`类时，实现了一个`Serializable`接口，然而当我们点进`Serializable`接口内部查看，发现它**竟然是一个空接口**，并没有包含任何方法！

试想，如果上面在定义`Student`类时忘了加`implements Serializable`时会发生什么呢？

实验结果是：此时的程序运行**会报错**，并抛出`NotSerializableException`异常。

如果一个对象既不是**字符串**、**数组**、**枚举**，而且也没有实现`Serializable`接口的话，在序列化时就会抛出`NotSerializableException`异常！

原来`Serializable`接口也仅仅只是做一个标记用！它告诉代码只要是实现了`Serializable`接口的类都是可以被序列化的！然而真正的序列化动作不需要靠它完成。

---

## serialVersionUID号有何用？

相信你一定经常看到有些类中定义了如下代码行：

```java
private static final long serialVersionUID = -4392658638228508589L;
```

继续做一个简单实验，还拿上面的`Student`类为例，我们并没有人为在里面显式地声明一个`serialVersionUID`字段。

我们首先调用`serialize()`方法，将对象序列化到本地。接下来我们在`Student`类里面再增加一个名为`studentID`的字段，然后进行反序列化，结果**报错了**，并且抛出了`InvalidClassException`异常：序列化前后的`serialVersionUID`号码不兼容！

从这里可以得出**两个**重要信息：

1. **serialVersionUID是序列化前后的唯一标识符**
2. **默认如果没有人为显式定义过`serialVersionUID`，那编译器会为它自动声明一个！**

所以，为了`serialVersionUID`的确定性，写代码时还是建议，凡是`implements Serializable`的类，都最好人为显式地为它声明一个`serialVersionUID`明确值！

---

## 两种特殊情况

- 1、凡是被`static`修饰的字段是不会被序列化的
- 2、凡是被`transient`修饰符修饰的字段也是不会被序列化的

**对于第一点**，因为序列化保存的是**对象的状态**而非类的状态，所以会忽略`static`静态域也是理所应当的。

**对于第二点**，如果在序列化某个类的对象时，就是不希望某个字段被序列化（比如这个字段存放的是隐私值，如：`密码`等），那这时就可以用`transient`修饰符来修饰该字段。

---

## 序列化的受控和加强

### 约束性加持

序列化和反序列化的过程其实是**有漏洞的**，因为从序列化到反序列化是有中间过程的，如果被别人拿到了中间字节流，然后加以伪造或者篡改，那反序列化出来的对象就会有一定风险了。

**答案就是：** 自行编写`readObject()`函数，用于对象的反序列化构造，从而提供约束性。

```java
private void readObject( ObjectInputStream objectInputStream ) throws IOException, ClassNotFoundException {
    // 调用默认的反序列化函数
    objectInputStream.defaultReadObject();

    // 手工检查反序列化后学生成绩的有效性，若发现有问题，即终止操作！
    if( 0 > score || 100 < score ) {
        throw new IllegalArgumentException("学生分数只能在0到100之间！");
    }
}
```

### 单例模式增强

一个容易被忽略的问题是：**可序列化的单例类有可能并不单例**！

**解决办法是**：在单例类中手写`readResolve()`函数，直接返回单例对象：

```java
private Object readResolve() {
    return SingletonHolder.singleton;
}
```

这样一来，当反序列化从流中读取对象时，`readResolve()`会被调用，用其中返回的对象替代反序列化新建的对象。
