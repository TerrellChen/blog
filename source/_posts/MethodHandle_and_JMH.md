---
title: 反射(MethodHandle)及JMH测试
date: 2020-08-02 00:40:00
tags:
---
<!-- toc -->

## 1 背景

在写Java agent的过程中遇到不可避免需要走反射调用的场景。之前也知道反射的性能会有很大的损失，搜了一把这方面的文章，希望能找到正确的姿势来尽可能避免性能的损耗。

按照搜索结果进行性能排序，推荐的依次是：直接调用->ReflectAsm->Method，以及偶然间发现的有小部分文章提到JDK7引入的MethodHandle。

在使用过程中，发现MethodHandle能找到的资料比较匮乏且并不好用，本文主要简单介绍MethodHandle的使用方式，并对上面提到的几项技术进行基础的性能测试验证。

## 2 使用MethodHandle的姿势

讲真这里坑了我挺久，照着搜到的资料一阵操作各种异常，最终还是免不了自己看代码。

MethodHandle(java.long.invoke)的使用与Method(java.long.reflect)的使用比较类似，主要障碍是在如何生成一个MethodHandle。

首先需要通过java.long.invoke中有提供MethodHandles工具类的静态方法创建Lookup，之后通过Lookup的几个接口进行MethodHandle的生成。通常会有如下四种场景，下面分别介绍。

### 2.1 获取MethodHandle

**运行环境**

```java
public interface FooInterface {
    Object bar(int a, char b, String c);
}

public class Foo implements FooInterface {

    @Override
    public Object bar(int a, char b, String c) {
        return "Foo#bar";
    }

    protected int bar(int a, char b) {
        return 0;
    }

    private void bar(int a) {
    }
}

public class Bar {
    protected static FooInterface fooInterface = new FooInterface() {
        @Override
        public Object bar(int a, char b, String c) {
            return "Bar#Foo";
        }
    };
}
```

**获取Public方法**

```java
Class fooClazz = Class.forName("terrell.common.entity.Foo");
MethodHandle publicFooBarMethodHandle = MethodHandles.lookup().findVirtual(fooClazz, "bar", MethodType.methodType(Object.class, int.class, char.class, String.class));
```

**获取Private方法**

```java
Class fooClazz = Class.forName("terrell.common.entity.Foo");
Method privateFooBarMethod = fooClazz.getDeclaredMethod("bar", int.class);
privateFooBarMethod.setAccessible(true);
MethodHandle privateFooBarMethodHandle = MethodHandles.lookup().unreflect(privateFooBarMethod);
```

**获取匿名内部类方法**

```java
// 获取想要的匿名内部类
Class fooInterfaceClazz = Class.forName("terrell.common.entity.FooInterface");
Reflections reflections = new Reflections("terrell.common.entity");
Set<Class<? extends FooInterface>> classesSubTypesOfFooInterface = reflections.getSubTypesOf(FooInterface.class);
Iterator<Class<? extends FooInterface>> iterator  = classesSubTypesOfFooInterface.iterator();
Class fooInterfaceAnonymousImplementation = null;
while (iterator.hasNext()) {
  fooInterfaceAnonymousImplementation = iterator.next();
  if (fooInterfaceAnonymousImplementation.getName().contains("Bar")) {
    System.out.println(fooInterfaceAnonymousImplementation.getName());
    break;
  }

}
// 之后类似于获取Private方法的Methodhandle
Method  anonymousPrivateBarFooMethod = fooInterfaceAnonymousImplementation.getMethod("bar", int.class, char.class, String.class);
anonymousPrivateBarFooMethod.setAccessible(true);
MethodHandle anonymousPrivateBarFooMethodHandle = MethodHandles.lookup().unreflect(anonymousPrivateBarFooMethod);
```

获取MethodHandle的方式如上，Lookup是创建MethodHandle的工厂，通过Lookup的方法来创建MethodHandle。搜到的资料推荐的是以下的方法及存在的问题：

- 获取Public：findVirtual（无问题）
- 获取Private: findSpecial（需要有access权限。。）
- 获取匿名内部类：没找到

后来找到通过先获取Method，再通过Method生成MethodHandle的方式，对于Private权限或匿名内部类都通用。

### 2.2 MethodHandle的调用

MethodHandle的调用提供了下面几种方法

```java
Object foo = publicFooBarMethodHandle.invoke(fooInstance, 0, 'a', "a"));
Object bar = publicFooBarMethodHandle.invokeExact((Foo)fooInstance, 0, 'a', "a"));
Object foobar = publicFooBarMethodHandle.invokeWithArguments(fooInstance, 0, 'a', "a"));
```

- invoke: 参数可以转换类型的调用
- invokeExact: 参数类型要求一致的调用
- invokeWithArguments: 这个的用途目前我还没理解特别清晰，可以自己再去看下文档，这里摘录一部分（Unlike the signature polymorphic methods `invokeExact` and `invoke`, `invokeWithArguments` can be accessed normally via the Core Reflection API and JNI. It can therefore be used as a bridge between native or reflective code and method handles.）

## 3 JMH的介绍及使用

**引入**

```java
compile 'org.openjdk.jmh:jmh-core:1.23'
compile 'org.openjdk.jmh:jmh-generator-annprocess:1.23'
```

**标记**

```java
// 标记在需要测试的方法上
@Benchmark
```

**配置**

```java
// 测试模式，Mode下四选一，需要用的时候可以看看，一目了然
@BenchmarkMode(Mode.Throughput)
// 预热环节，time单位是second
@Warmup(batchSize = 1, iterations = 1, time = 5)
// 测死进程数
@Fork(value = 1, warmups = 1)
// 测试环节
@Measurement(iterations = 2, time = 5)
```

**执行**

```
idea中安装JMH plugin
应用生效后，直接执行标记过的方法或类即可
```

## 4 几种反射方式的性能测试及结论

**测试环境**

```
# JMH version: 1.23
# VM version: JDK 1.8.0_191, Java HotSpot(TM) 64-Bit Server VM, 25.191-b12
# VM invoker: /Library/Java/JavaVirtualMachines/jdk1.8.0_191.jdk/Contents/Home/jre/bin/java
# VM options: -Dfile.encoding=UTF-8
# Warmup: 1 iterations, 5 s each
# Measurement: 2 iterations, 5 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time

硬件部分
MacMini 2018
处理器 i7 6C 3.2GHz
内存 32GB
```



**测试代码**

```java
import com.esotericsoftware.reflectasm.MethodAccess;
import org.openjdk.jmh.annotations.*;

import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.reflect.Method;

@BenchmarkMode(Mode.Throughput)
@Warmup(batchSize = 1, iterations = 1, time = 5)
@Fork(value = 1, warmups = 1)
@Measurement(iterations = 2, time = 5)
public class TestReflection {
    private static MethodHandle methodHandle;
    private static Method method;
    private static MethodAccess methodAccess;

    static {
        try {
            method = TestReflection.class.getMethod("doSomething", int.class, int.class);
            methodHandle = MethodHandles.lookup().unreflect(method);
            methodAccess = MethodAccess.get(TestReflection.class);
        } catch (Throwable t) {
            t.printStackTrace();
        }
    }

    private static int COUNT = 100000;

    public int doSomething(int a, int b) {
        if (a > b) {
            return a * 100 - b << 2;
        } else {
            return a + b * 15;
        }
    }

    @Benchmark
    public int testDirectInvoke() {
        int start = 0;
        for (int i = 0; i < COUNT; i++) {
            start = (int) doSomething(start, i);
        }
        return start;
    }

    @Benchmark
    public int testCachedMethod() throws Exception {
        int start = 0;
        for (int i = 0; i < COUNT; i++) {
            start = (int) method.invoke(this, start, i);
        }
        return start;
    }

    @Benchmark
    public int testCachedMethodHandleAndInvokeWithArguments() throws Throwable {
        int start = 0;
        for (int i = 0; i < COUNT; i++) {
            start = (int) methodHandle.invokeWithArguments(this, start, i);
        }
        return start;
    }

    @Benchmark
    public int testCachedMethodHandleAndInvoke() throws Throwable {
        int start = 0;
        for (int i = 0; i < COUNT; i++) {
            start = (int) methodHandle.invoke(this, start, i);
        }
        return start;
    }

    @Benchmark
    public int testCachedMethodHandleAndInvokeExact() throws Throwable {
        int start = 0;
        for (int i = 0; i < COUNT; i++) {
            start = (int) methodHandle.invokeExact(this, start, i);
        }
        return start;
    }

    @Benchmark
    public int testCachedReflectAsm() {
        int start = 0;
        for (int i = 0; i < COUNT; i++) {
            start = (int) methodAccess.invoke(this, "doSomething", start, i);
        }
        return start;
    }

}
```

**主要测试类型及结果**

```
Benchmark                                                     Mode  Cnt      Score   Error  Units
TestReflection.testCachedMethod                              thrpt    2   2051.502          ops/s
TestReflection.testCachedMethodHandleAndInvoke               thrpt    2   3866.245          ops/s
TestReflection.testCachedMethodHandleAndInvokeExact          thrpt    2   4061.695          ops/s
TestReflection.testCachedMethodHandleAndInvokeWithArguments  thrpt    2     88.988          ops/s
TestReflection.testCachedReflectAsm                          thrpt    2    505.247          ops/s
TestReflection.testDirectInvoke                              thrpt    2  19160.435          ops/s
```

这里使用的吞吐量测试的模式，可以看到同等条件下，ReflectAsm的性能并不如很多资料介绍的那么好（难道我姿势不对？也没找到别的姿势了）；而Method（缓存过Method本体）调用也没有想象中那么差；同时可以确认的是MethodHandle使用准确参数方式调用的性能是几种测试的反射类型中最好的。

## 5 留下的坑

关于MethodHandle与Method的区别未进行深入。

关于MethodHandle几种调用方式的性能差异原因未进行深入。

ReflectAsm性能问题未进行深入。

以上随缘挖掘。。
