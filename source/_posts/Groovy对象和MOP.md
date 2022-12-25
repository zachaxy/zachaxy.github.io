---
title: Groovy对象和MOP
date: 2017-12-08 21:32:11
tags: Groovy
---

# Groovy 对象
Groovy 中的对象其实本质也是 Java 对象，只不过比 Java 对象附加了一些其它的功能。在 Groovy 中的对象，其顶级父类也是 `java.lang.Object`，同时其也实现了 `groovy.lang.GroovyObject` 接口。

```groovy
public interface GroovyObject {
    Object invokeMethod(String name, Object args);

    Object getProperty(String propertyName);

    void setProperty(String propertyName, Object newValue);

    MetaClass getMetaClass();

    void setMetaClass(MetaClass metaClass);
}
```

我们的 Groovy 中的对象是可以直接调用 GroovyObject 接口中的方法，然而我们定义的 Groovy 类中没有自己实现过该接口中的任何方法，因此其必定有一个类帮我们实现好了这些方法。现在还未得到证实，但是我猜测是 GroovyObjectSupport 这个类提供的实现，并且在运行时注入到了我们的 Groovy 类中，所以我们才能调用这些方法。
```groovy
public abstract class GroovyObjectSupport implements GroovyObject {

    // never persist the MetaClass
    private transient MetaClass metaClass;

    public GroovyObjectSupport() {
        this.metaClass = InvokerHelper.getMetaClass(this.getClass());
    }

    public Object getProperty(String property) {
        return getMetaClass().getProperty(this, property);
    }

    public void setProperty(String property, Object newValue) {
        getMetaClass().setProperty(this, property, newValue);
    }

    public Object invokeMethod(String name, Object args) {
        return getMetaClass().invokeMethod(this, name, args);
    }

    public MetaClass getMetaClass() {
        if (metaClass == null) {
            metaClass = InvokerHelper.getMetaClass(getClass());
        }
        return metaClass;
    }

    public void setMetaClass(MetaClass metaClass) {
        this.metaClass = metaClass;
    }
}
```


这里重点关注一个对象 MetaClass。在 Groovy 的世界中，每个对象(不管是 Groovy 对象还是 Java 对象)都包含一个 MetaClass 对象，该 MetaClass 对象持有其所依附的对象的所有信息(包括属性和方法)，每当我们调用一个对象的方法时，都是由该 MetaClass 对象负责路由对方法的调用。我们知道一旦一个类被加载进 JVM，那么这个类就无法修改了，但是我们可以修改这个类的 MetaClass 对象，从而实现对类动态的添加方法和行为。

Groovy 中还有一种特殊的对象——实现了 GroovyInterceptable 接口的类，GroovyInterceptable 接口是一个标记接口，其扩展了 GroovyObject，对于实现了该接口的对象而言，只要调用该对象上的任何方法，都会被 invokeMethod 方法拦截。(要实现拦截，不仅要在类的定义中声明实现该接口，同时还要重写其 invokeMethod 方法，才会有拦截的效果)
```groovy
public interface GroovyInterceptable extends GroovyObject {
}
```

## Groovy 中的方法调用机制
![Groovy interception mechanism](http://docs.groovy-lang.org/next/html/documentation/assets/img/xGroovyInterceptions.png.pagespeed.ic.lQZRl0FSbC.webp)

该图描述了 Groovy 中方法调用的路由机制。这里做以下补充：
1. invokeMethod 方法是 GroovyObject 接口中的方法，所有的 Groovy 类都默认实现了该方法。而 GroovyInterceptable 只是一个标记接口，该接口的作用是将 invokeMethod 方法的调用时机提前到了最前面，也就是所有的方法调用都会先统一路由到 invokeMethod 方法中，若为实现 GroovyInterceptable 接口，那么 invokeMethod 方法只有最后才有机会执行。
2. 若在类的定义中声明了 GroovyInterceptable 接口，但是在类中没有覆盖 invokeMethod 方法，则等同于没有实现 GroovyInterceptable 接口，路由转向左侧。
3. 若未实现 GroovyInterceptable 接口，而一个类的外部直接调用了 invokeMethod 方法，那么就是方法的直接调用了，不存在拦不拦截的问题，但是如果该类中又没有覆盖 invokeMethod 方法，那么会调用 methodMissing 方法(如果有的话)
4. 若向一个类的 metaClass 中添加了 invokeMethod 方法或者 methodMissing 方法，在外部调用一个不存在的方法时，会路由到该 invokeMethod 方法上，如果没有实现 invokeMethod 方法，那么会路由到 metaClass 上的 methodMissing 方法上(如果有的话)


提供一个例子直观的感受下 groovy 的方法路由机制
```groovy
class TestMethodInvocation extends GroovyTestCase {
    void testInterceptedMethodCallonPOJO() {
        def val = new Integer(3)
        Integer.metaClass.toString = { -> 'intercepted' }

        assertEquals "intercepted", val.toString()
    }

    //实现了 GroovyInterceptable 接口,复写 invokeMethod,那么所有的方法调用,都会被路由到 invokeMethod 中
    void testInterceptableCalled() {
        def obj = new AnInterceptable()
        assertEquals 'intercepted', obj.existingMethod()
        assertEquals 'intercepted', obj.nonExistingMethod()
        assertEquals 'intercepted', obj.invokeMethod("existingMethod", null)
        assertEquals 'intercepted', obj.invokeMethod("nonExistingMethod", null)
    }

    void testInterceptedExistingMethodCalled() {
        //将原有的 ex2 方法覆盖了
        AGroovyObject.metaClass.existingMethod2 = { -> 'intercepted' }
        def obj = new AGroovyObject()
        assertEquals 'intercepted', obj.existingMethod2()
    }

    void testUnInterceptedExistingMethodCalled() {
        def obj = new AGroovyObject()
        assertEquals 'existingMethod', obj.existingMethod()
    }

    void testPropertyThatIsClosureCalled() {
        def obj = new AGroovyObject()
        assertEquals 'closure called', obj.closureProp()
    }

    void testMethodMissingCalledOnlyForNonExistent() {
        def obj = new ClassWithInvokeAndMissingMethod()
        assertEquals 'existingMethod', obj.existingMethod()
        assertEquals 'missing called', obj.nonExistingMethod()
        assertEquals 'invoke called', obj.invokeMethod("haha", null)

    }

    void testInvokeMethodCalledForOnlyNonExistent() {
        def obj = new ClassWithInvokeOnly()
        assertEquals 'existingMethod', obj.existingMethod()
        assertEquals 'invoke called', obj.nonExistingMethod()
    }

    void testClassWithMethodMissingOnly(){
        def obj = new ClassWithMethodMissingOnly()
        assertEquals 'existingMethod', obj.existingMethod()
        assertEquals 'missing called', obj.nonExistingMethod()
        assertEquals 'missing called', obj.invokeMethod("haha",null)
    }
    void testMethodFailsOnNonExistent() {
        def obj = new TestMethodInvocation()
        shouldFail(MissingMethodException) { obj.nonExistingMethod() }
    }
}

// 实现了 GroovyInterceptable 接口,复写 invokeMethod,那么所有的方法调用,都会被路由到 invokeMethod 中
class AnInterceptable implements GroovyInterceptable {
    def existingMethod() {}

    def invokeMethod(String name, args) { 'intercepted' }
}

// 普通的 Groovy 类
class AGroovyObject {
    def existingMethod() { 'existingMethod' }

    def existingMethod2() { 'existingMethod2' }
    def closureProp = { 'closure called' }
}

// 普通的 Groovy 类,并实现 invokeMethod 和 methodMissing 方法
class ClassWithInvokeAndMissingMethod {
    def existingMethod() { 'existingMethod' }

    def invokeMethod(String name, args) { 'invoke called' }

    def methodMissing(String name, args) { 'missing called' }
}

// 普通的 Groovy 类,并实现 invokeMethod
class ClassWithInvokeOnly {
    def existingMethod() { 'existingMethod' }

    def invokeMethod(String name, args) { 'invoke called' }
}

class ClassWithMethodMissingOnly {
    def existingMethod() { 'existingMethod' }

    def methodMissing(String name, args) { 'missing called' }
}
```

# MOP(MetaObject Protocol)
MOP：元对象协议。由 Groovy 语言中的一种协议。该协议的出现为元编程提供了优雅的解决方案。而 MOP 机制的核心就是 MetaClass。
元编程：编写能够操作程序的程序，也包括操作程序自身。


正是 Groovy 提供了 MOP 的机制，才使得 Groovy 对象更加灵活，我们可以根据对象的 metaClass，动态的查询对象的方法和属性。这里所属的动态指的是在运行时，根据所提供的方法或者属性的字符串，即可得到
有点类似于 Java 中的反射，但是在使用上却比 Java 中的反射简单的多。

## 动态访问方法
常用的方法有:
- getMetaMethod()
- resondsTo()
- hasProperty()
- ......  




在使用了 getMetaMethod 方法得到一个 MetaMethod 后，可以调用其 invoke 方法执行。

```groovy
str = 'hello'
targetMethod = str.metaClass.getMetaMethod('toUpperCase')
targetMethod.invoke(str)
```

## 动态访问方法或属性——Groovy 的语法糖
```groovy
class Foo{
	int bar
	def func1(){}
}

foo = new Foo()

String propertyName = 'bar'
String methodName = 'func1'

//访问属性
foo[propertyName]
foo."$propertyName"

//访问方法
foo."$methodName"()
foo.invokeMethod(methodName,null)
```

