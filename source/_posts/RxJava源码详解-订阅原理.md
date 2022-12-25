---
title: RxJava源码详解-订阅原理
date: 2017-04-03 09:19:10
tags: 框架设计之美
---

## 深入观察者与被观察者模式

下面以一个经典的例子进行讲解:

```java
//创建一个被观察者（开关）
 Observable switcher=Observable.create(new Observable.OnSubscribe<String>(){

        @Override
        public void call(Subscriber<? super String> subscriber) {
            subscriber.onNext("On");
            subscriber.onNext("Off");
            subscriber.onNext("On");
            subscriber.onNext("On");
            subscriber.onCompleted();
        }
    });
//创建一个观察者（台灯）
 Subscriber light=new Subscriber<String>() {
        @Override
        public void onCompleted() {
            //被观察者的onCompleted()事件会走到这里;
            Log.d("DDDDDD","结束观察...\n");
        }

        @Override
        public void onError(Throwable e) {
                //出现错误会调用这个方法
        }
        @Override
        public void onNext(String s) {
            //处理传过来的onNext事件
            Log.d("DDDDD","handle this---"+s)
        }
//订阅
switcher.subscribe(light);
```



针对上面的代码提出两个问题:
1. 被观察者中的Observable.OnSubscribe是什么，有什么用?
2. call(subscriber)方法中，subscriber哪里来的?
3. 为什么只有在订阅之后，被观察者才会开始发送消息?

## Observable 类深入
首先看一下OnSubscribe是什么?

```java
//前面也提到Acton1这个接口，内部只有一个待实现call()方法
//没啥特别，人畜无害
public interface Action1<T> extends Action {
    void call(T t);
}
//OnSubscribe继承了这个Action1接口,自己并没有增加新的方法;
//但是新增了 Subscriber 类的泛型要求,也就是说只能用在订阅者中
public interface OnSubscribe<T> extends Action1<Subscriber<? super T>> {
      // OnSubscribe仍然是个接口
}
```

前面也提到`Acton1`这个接口，内部只有一个待实现`call()`方法.
`OnSubscribe `接口继承了这个` Action1` 接口,但是自己并没有增加新的方法;只是把泛型限定在了 `Subscriber`,也就是说必须传入观察者对象;
那么也就是说，`OnSubscribe`本质上也是和` Action1`一样的接口，只不过它专门用于`Observable`内部。

那么将` OnSubscribe` 接口传入` Observable` 后,该接口干什么?

答:用来触发事件的发生,在`call()`内部调用观察者的 `onNext()`方法!!!



要注意的是在 `Observable` 被观察者的类中，`OnSubscribe`是它唯一的属性,
同时也是`Observable`构造函数中唯一必须传入的参数，也就是说，只要创建了`Observable`，那么内部也一定有一个 `OnSubscribe` 实体对象。

当然,`Observable`是没有办法直接new的,我们只能通过create(),just()等等方法创建,这些方法背后去调用了`new Observable(onSubscribe)`

再来大体看一下 `Observable `这个类的骨架:

```java
public class Observable<T> {
    //唯一的属性
    final OnSubscribe<T> onSubscribe;

    //构造函数，因为protected，我们只能使用create函数
    protected Observable(OnSubscribe<T> f) {
        this.onSubscribe = f;
    }

    //create(onSubscribe) 内部调用构造函数。
    public static <T> Observable<T> create(OnSubscribe<T> f) {
        return new Observable<T>(RxJavaHooks.onCreate(f));
    }
}
```


## Observer 类深入

Observer是一个接口,其内部有三个方法:

- void onCompleted():处理被观察者发送完事件后的逻辑
- void onError(Throwable e):处理出错逻辑
- void onNext(T t):处理事件,有一个事件就执行一次



顺便说一下 Subscriber,其实 Observer的实现类,但也是抽象的,主要是也实现了Subscription接口,所以相对于 Observer接口添加了一些新的方法,也有一些除了这两个接口之外自定义的方法,都自己实现了

```java
abstract class Subscriber<T> implements Observer<T>, Subscription
```

看一下其中的主要方法:

Observer中的方法:(Observer中的方法在Subscriber中均未实现,需要开发者自己实现,只有这个三个Subscriber没有实现)

- void onCompleted():处理被观察者发送完事件后的逻辑
- void onError(Throwable e):处理出错逻辑
- void onNext(T t):处理事件,有一个事件就执行一次



Subscription中的方法:

- void unsubscribe():取消订阅,停止接受被观察者发来的消息;
- boolean isUnsubscribed():是否取消了订阅被观察者发的消息;



Subscriber自身的方法:

-  void add(Subscription s):添加一个订阅对象

-  void onStart():是一个空实现,开发者可以自己实现该方法,该方法被调用的时机是在被观察者与观察者产生订阅,但是此时被观察者还为发出任何消息


## subscribe(subscriber)方法深入
当创建了 Observable 和 Observer 之后，调用subscribe(subscriber)方法产生订阅关系时，发生了什么呢？

```java
//给被观察者传入了观察者对象
//这个方法是有返回值的,其类型是Subscription,熟悉吗?这是上面Subscriber对象实现的某一个接口;我们经常拿到这个引用,然后调用其unsubscribe()方法防止内存泄露啊!!!
public final Subscription subscribe(final Observer<? super T> observer) {
   ....
    //调用了下面的方法,将observer类型包装成subscribe类型
    return subscribe(new ObserverSubscriber<T>(observer));
}

public final Subscription subscribe(Subscriber<? super T> subscriber) {
    //往下调用
    return Observable.subscribe(subscriber, this);
}

//其实最终是调用了这个方法
static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
   //注:这个方法是在被观察者中的方法,执行订阅函数,那么此时被观察者就持有了观察者的引用了
  //也就是说现在被观察者可以调用观察者的任何方法了;
  //被观察者持有观察者的引用,这是整个观察者模型最关键的地方;
  
  
    //刚产生订阅关系的时候,这时被观察者还未发送消息,先执行观察者的onStart()方法
    subscriber.onStart();

    // add a significant depth to already huge call stacks.
    try {
        // 在这里简单讲，对onSubscribe进行封装，不必紧张。
      //该方法的源码单纯的将 onSubscribe 对象返回了,什么也没做,所以你可以忽略这一步;
        OnSubscribe onSubscribe=RxJavaHooks.onObservableStart(observable, observable.onSubscribe);

        //这个才是重点！！！
        //这个调用的具体实现方法就是我们创建观察者时
        //写在Observable.create()中的call()呀
        //而调用了那个call(),就意味着事件开始发送了
        onSubscribe.call(subscriber);
        
		//源码中该方法的实现是:单纯的把subscriber返回了;
      //其实在真正使用时我们并不关心订阅函数的返回值,反正该方法中被观察者发送消息,观察者处理消息的逻辑已经执行了;
        return RxJavaHooks.onObservableReturn(subscriber);
        } catch (Throwable e) {

        }
        return Subscriptions.unsubscribed();
    }
}
```


代码看到这里，我们就可以对上面三个问题做统一的回答了：

1. onSubscribe是Observable内部唯一属性，是连接Observable和subscriber的关键，相当于连接台灯和开关的那根电线,主要是其内部的call()方法,在该方法中发送了消息;
2. call(Subscriber<? super String> subscriber)中的subscriber，就是我们自己创建的那个观察者
3. 只有在订阅的时候，才会发生onSubscribe.call(subscriber)，进而才会开始调用onNext(),onComplete()等。