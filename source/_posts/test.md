---
title: test
date: 2022-12-25 10:24:38
tags:
---

# 深入操作符
操作符的实现原理?他是如何拦截事件，然后变换处理之后，最后传递到观察者手中的呢？

这里依然以 map()为例,看看map背后到底做了什么:


这个例子更好理解:
```java
//原始被观察者观察的是字符串
//原始观察者观察到的是整数;

//原始被观察者,真正负责发数据的;
Observable<String> observable = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("1");
        subscriber.onNext("2");
        subscriber.onNext("3");
        subscriber.onCompleted();
    }
});


//代理被观察者,是由原始的被观察者衍生出来的;
Observable<Integer> observableProxy = observable.map(new Func1<String, Integer>() {
    @Override
    public Integer call(String s) {
        System.out.println("转换工厂...");
        return Integer.valueOf(s);
    }
});


//原始的观察者;
Subscriber subscriber = new Subscriber<Integer>() {
    @Override
    public void onCompleted() {

    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(Integer integer) {
        System.out.println("我是真正的观察者-> " + integer);
    }
};
observableProxy.subscribe(subscriber);
```

首先看一下这个`map`方法内部是如何实现的:首先明确一点:map方法返回的是一个代理被观察者,所以其实是原始观察者订阅了代理被观察者,这是整个前提!!!


```java
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
        //创建了全新代理的的Observable，构造函数传入的参数是OnSubscribe
        //OnSubscribeMap显然是OnSubscribe的一个实现类，
        //也就是说，OnSubscribeMap需要实现call()方法
        //构造函数传入了原始的被观察者Observable对象,也就是说代理被观察者持有一个
  		//原始 被观察者的一个引用;
        //和一个开发者自己实现的Func1的实例,就是map的转化方法
    return create(new OnSubscribeMap<T, R>(this, func)); //this是原始被观察者;
}
```

因为map 实现的功能是 将 a 转换为 b,a/b均是被观察者,所以map中又创建了一个 Observable 的对象;

map背后究竟做了什么?

```java
//牢牢记住,这个OnSubscribeMap本质上还是一个OnSubscribe对象,用来创建被观察者时传入的,这是整个前提!
//上面也看到了 map()方法中又创建了一个新的被观察者(代理被观察者)
public final class OnSubscribeMap<T, R> implements OnSubscribe<R> {
    //用于保存原始的Observable对象
    final Observable<T> source;
    //还有我们传入的那个Func1的实例,func1中提供了转换  a->b 方法; 是通过 Func1 中的 T call()方法实现的
    final Func1<? super T, ? extends R> transformer;

    public OnSubscribeMap(Observable<T> source, Func1<? super T, ? extends R> transformer) {
        this.source = source;
        this.transformer = transformer;
    }

    //实现了call方法，我们知道call方法传入的Subscriber
  	//mad,这个call 方法一定要和 Func1 中的 转换方法 call 区分开,这个 call 方法是用来发送事件的;
    // 这个代理的被观察者的call方法有什么用呢?还用问,被观察者没有call方法还玩个毛.
  
  
 //和原来不同的是,这个call中并不是调用观察者的next()方法.而是产生一个新的观察者=>代理观察者;
  
  
    //然后产生一个新的的订阅关系=>原始被观察者和代理观察者的订阅
    //就是订阅之后，外部传入真实的的观察者,观察者什么时候传进来的?答:最终就是这个代理被观察者订阅的原始的观察者啊,所以这个o就是原始观察者;
    @Override
    public void call(final Subscriber<? super R> o) {
        // o 是原始观察者.
        //把外部传入的真实观察者传入到MapSubscriber，构造一个 代理的观察者
        //parent是代理观察者;
        MapSubscriber<T, R> parent = new MapSubscriber<T, R>(o, transformer);
      
       //给原始的观察者添加一个订阅;改了啊,改了,原始的观察者现在订阅代理观察者了,mad;
       //其实下面这句话可以忽略,没啥实际意义;
        o.add(parent);
        //让外部的Observable去订阅这个代理的观察者
      //其实是: 代理观察者去订阅原始的观察者
        source.unsafeSubscribe(parent);  //!!!!核心在这一步,正是由于这一步,原始被观察者就发送数据了;
    }

    //Subscriber的子类，用于构造一个代理的观察者
    static final class MapSubscriber<T, R> extends Subscriber<T> {
        //这个Subscriber保存了原始的观察者
        final Subscriber<? super R> actual;
        
      //我们自己在外部自己定义的Func1,转换方法 a->b
        final Func1<? super T, ? extends R> mapper;

        boolean done;

        public MapSubscriber(Subscriber<? super R> actual, Func1<? super T, ? extends R> mapper) {
            this.actual = actual; //原始观察者;
            this.mapper = mapper;  //转换方法;
        }
      
      
        //原始的Observable发送的onNext()等事件
        //都会首先传递到代理观察者这里?为什么,在哪里实现的?
      //其实并没有什么所谓的传递,事件只是一个变量,直接用就可以了;
      //看上面代理被观察者中的call方法,最后不是将 原始被观察者 与 代理观察者绑定了嘛...
      //有道理 √
      //本来啊,是被观察者
        @Override
        public void onNext(T t) {
            R result;

            try {
                //mapper其实就是开发者自己创建的Func1，
                //call()开始变换数据
              //t是事件,事件只是一个变量,直接用就可以,并不存在事件的传递;
              //这里只是将一个变量转换为另一种类型的变量;
                result = mapper.call(t);
            } catch (Throwable ex) {
                Exceptions.throwIfFatal(ex);
                unsubscribe();
                onError(OnErrorThrowable.addValueAsLastCause(ex, t));
                return;
            }
            //调用真实的观察者的onNext()
            //从而在变换数据之后，把数据送到真实的观察者手中
            actual.onNext(result);
        }
        //onError()方法也是一样
        @Override
        public void onError(Throwable e) {
            if (done) {
                RxJavaHooks.onError(e);
                return;
            }
            done = true;

            actual.onError(e);
        }


        @Override
        public void onCompleted() {
            if (done) {
                return;
            }
            actual.onCompleted();
        }

        @Override
        public void setProducer(Producer p) {
            actual.setProducer(p);
        }
    }
}
```

代理被观察者存在的意义:之前我在想,如果经过map转换的话,只是需要一个代理观察者就好了,为什么还需要一个代理被观察者,其实代理被观察者在这里有两个使命:

- 并不是所有的代码都是按照流式一下写下来的,可能是分别创建观察者,被观察者,然后订阅,这个时候map的转换需要产生一个代理被观察者
- 代理被观察者与原始观察者进行订阅,代理被观察者中的call方法是先执行的,那么它的call()方法内部实现了什么呢?就是原始被观察者和代理被观察者的订阅,因为代理被观察者是没有数据源的,只有在这里call()中进行订阅,原始被观察者才会发出数据,给代理被观察者用func1转换方法处理,处理后再发送原始观察者;

1.  问题1:这个代理观察者的观察者参数是什么时候传进来的?

   答:不要心急,记住总的原则是:观察者是在订阅的时候传入进来的.我们可以这样理解,本来我们创建被观察者和观察者,然后订阅.现在多了一个 map 操作,经过map操作后,被观察者就变了,变成了这个代理被观察者,这时候我们忘掉原始的观察者吧.原始的观察者只负责产生原始的事件源,何况这个代理被观察者中还持有原始被观察者的引用.那么在这个代理被观察者的call方法中其实是使用原始被观察者的call发原始数据,只不过发之前先不给观察者,而是自己调用func1来把事件转换一下;



map(Func1 func1);

首先我们看在使用map时,需要传入一个 Func1 实例,其实 Func1 是一个接口,需要实现的方法是一个 call()方法,正是靠这个call 方法实现的转换;

接下来看 map 内部是怎么实现的,其创建了一个 Observable 实例并返回;这也很好理解,进行了 map 转换后还是一个被观察的对象;,

return create(new OnSubscribeMap<T, R>(this, func));

跟进去这个创建的流程中,前面也分析了创建一个 Observable 对象不能 new,而是使用一个 create()方法,并传入一个 OnSubscribe 的实例;同理 map 中创建的 Observable 对象也一样,只是这个传入的是 OnSubscribeMap 对象,这是 OnSubscribe 接口的一个实现类, OnSubscribeMap 需要两个参数,第一个参数是原始的被观察着,第二个参数是用来转换的 Func1 对象,之前也讲过,每个 被观察者中都持有一个 OnSubscribe,这个接口要求实现一个 call 方法,这个call方法是用来触发事件开始执行的,我们用了map新键的一个被观察者,对象被转换后,也需要一个新的触发机制,就在这个call中实现.



接下来看一下 OnSubscribeMap 中时如何实现 call() 方法的,在这个call方法中传入的是观察者,嗯,没错,传入的是观察者,这里新键了一个观察者,我们称之为代理观察者,



## 总结

这里面一共涉及到了四个对象

- 被观察者
- 代理被观察者
- 观察者
- 代理观察者



1. 被观察者通过 map 转换后产生了新的 代理被观察者,并持有原始被观察者的引用与转换的方法

2. (map方法产生的)代理被观察者 与 原始观察者产生订阅(明面上的,你看得到)

3. 通过上一步的订阅,代理被观察者就持有了原始观察者的引用

4. 代理被观察者的call方法中创建一个代理观察者

5. 代理观察者与原始被观察者进行订阅

6. 通过上一步的订阅,原始观察者发送消息给 代理观察者,发来的是 String

7. 代理观察者是代理被观察者的内部类,自然也可以访问代理被观察者的属性---转换方法

8. 代理观察者在其`onNext()`方法中获取到`String`,然后通过转换方法将 String 装换为 int

9. 代理观察者通过原始观察者的引用,将 int传给观察者(其实这时的代理观察者又有点被观察者的意思,它调用了原始观察者的 onNext(int) 方法)

10. 整个流程结束;

