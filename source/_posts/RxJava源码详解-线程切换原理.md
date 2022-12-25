---
title: RxJava源码详解-线程切换原理
date: 2017-04-03 09:43:49
tags: 框架设计之美
---



# 线程调度深入
一个基本线程调度的例子:事件在IO线程产生,然后再UI线程被消费;
```java
Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Hello RxJava!");
        subscriber.onCompleted();
    }
})
.subscribeOn(Schedulers.io())
.observeOn(AndroidSchedulers.mainThread())
.subscribe(new Subscriber<String>() {
    @Override
    public void onCompleted() {
        System.out.println("completed!");
    }
    @Override
    public void onError(Throwable e) {
    }
    @Override
    public void onNext(String s) {
        System.out.println(s);
    }
});
```
## subscribeOn()原理
`subscribeOn()`用来指定`Observable`在哪个线程中执行事件流，也就是指定`Observable`中`OnSubscribe`(计划表)的`call()`方法在那个线程发射数据。下面通过源码分析`subscribeOn()`是怎样实现线程的切换的。

```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
    //忽略这个 if 分支吧
    if (this instanceof ScalarSynchronousObservable) {
      return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
    }
    // 重点看这个:this指的是调用线程切换方法subscribeOn()的Observerble ,
    return create(new OperatorSubscribeOn<T>(this, scheduler));
}
```

subscribeOn()方法是 Observerble 中的方法,一旦调用了该方法,就会创建出一个新的 Observerble 对象;当然还是通过create(OnSubscribe)方法来创建Observerble ;



再来看一下新创建的这个Observerble 对象中的OnSubscribe的实现类内部是如何实现的;OperatorSubscribeOn是OnSubscribe的实现类,自然也要实现call方法来触发事件了.同时一旦换了新的Observerble ,那么最终的观察者订阅的自然也就是新的Observerble 了,这一点一定要明确;那么自然call方法中的参数也就持有了原始观察者的引用.

```java
public final class OperatorSubscribeOn<T> implements Observable.OnSubscribe<T> {

    final Scheduler scheduler;   //调度器
    final Observable<T> source;  //原始Observable

  //构造函数,传入原始的被观察者和线程调度器;
  	public OperatorSubscribeOn(Observable<T> source, Scheduler scheduler) {
        this.scheduler = scheduler;
        this.source = source;
    }

    //(1)原始观察者订阅了新的Observable后,将先执行此call方法(还记得订阅函数是如何实现的吗?)
  //这个参数的final的,其实是为了给内部类调用,内部类已经在其他线程了;
  //传入的参数是原始观察者;和上一篇操作符的原理类似,也是在call方法中创建了一个代理观察者,使其与原始被观察者订阅
    @Override
    public void call(final Subscriber<? super T> subscriber) {
        final Scheduler.Worker inner = scheduler.createWorker();  //创建了一个worker对象,内部持有一个线程池
        subscriber.add(inner);

//(2)call方法中使用传入的调度器创建的Worker对象的schedule方法切换线程,传入的Action0会作为一个参数传入runnable中
        //runnable的run方法中会调用action0的call方法,然后runnable又被添加到线程池中被执行;
          inner.schedule(new Action0() {
            @Override
            public void call() {
                final Thread thread = Thread.currentThread();
          //(3)根据外层call中传来的原始观察者,创建了一个新的观察者(代理观察者),而且代理观察者持有原始观察者的引用
                    Subscriber<T> s = new Subscriber<T>(subscriber) {
                    @Override
                    public void onNext(T t) {
                        //(5) 新的(代理)观察者收到数据后直接发送给原始观察者
                        subscriber.onNext(t);
                    }
                    ...
                };
                //(4)在切换的线程中，新的观察者订阅原始Observable，用来接收数据
              //代理观察者能收到数据的前提是因为代理观察者订阅了原始被观察者;
              //其实这个订阅的动作是在新线程中执行的.
                source.unsafeSubscribe(s);
            }
        });
    }
}
```

在`call`方法中通过`scheduler.createWorker().schedule(Action0)`完成线程的切换.

简单说:这里在subscribeOn()方法中新创建了一个Observable对象(代理Observable),于是发生了原始观察者与代理被观察者订阅的情况,于是代理被观察者中的call()方法被先执行,但是代理被观察者哪里有数据呢,还不是用老方法,又创建了一个代理观察者,然后让代理观察者与原始被观察者进行订阅,一旦发生订阅,数据就发出来了,数据发出来给了代理观察者,代理观察者的onNext()方法中有调用了原始观察者的onNext()方法;这不就解决了嘛,可是如何实现的线程切换呢?



提前说一下:这个Action0对象时作为参数传入一个Runnable实例中,然后将该runnable对象传入线程池,这样就实现了线程的切换,也就是说这个Action0()中的所有动作都是在新的线程池中执行的;



上述说说的一切动作都是在scheduler.createWorker().schedule(new Action0(XXX));都是在这个Action0()中发生的.

这里涉及到两个对象:`Scheduler`和`Worker`,究竟这是怎么实现的线程切换呢?



###  Scheduler

其实在subscribeOn(Scheduler  scheduler)方法中传入的参数就是 Scheduler 对象;

由于RxJava中有多种调度器，我们就看一个简单的`Schedulers.newThread()`，其他调度器的思路是一样的.

先看一下`Schedulers`这个类,`Schedulers`就是一个调度器的管理器,大管家;



```java
public final class Schedulers {
    //各种调度器对象,看着眼熟吧.
    private final Scheduler computationScheduler;
    private final Scheduler ioScheduler;
    private final Scheduler newThreadScheduler;
  
    //单例，Schedulers被加载的时候，上面的各种调度器对象已经初始化
    private static final Schedulers INSTANCE = new Schedulers();
    
    //构造方法,在构造方法中初始化各种调度器
    private Schedulers() {
        RxJavaSchedulersHook hook = RxJavaPlugins.getInstance().getSchedulersHook();
        ...
          
        //这里只关注创建一个新的线程的调度器
        Scheduler nt = hook.getNewThreadScheduler();
        if (nt != null) {
            newThreadScheduler = nt;
        } else {
            //①.创建newThreadScheduler对象
            newThreadScheduler = RxJavaSchedulersHook.createNewThreadScheduler();
        }
      //下面其余的线程不管.......................................
    //下面是Compute线程的创建  
      Scheduler c = hook.getComputationScheduler();
        if (c != null) {
            computationScheduler = c;
        } else {
            computationScheduler = RxJavaSchedulersHook.createComputationScheduler();
        }
//下面是IO线程的创建;
        Scheduler io = hook.getIOScheduler();
        if (io != null) {
            ioScheduler = io;
        } else {
            ioScheduler = RxJavaSchedulersHook.createIoScheduler();
        }
    }
  
  
    //②. 获取NewThreadScheduler对象,也就是我们在使用调度调用的该方法来获取一个新线程的调度器;
  //我们平时使用线程切换时,就是调用的 Schedulers.io(),Schedulers.newThread()等方法来获取一个Scheduler对象的!!
    public static Scheduler newThread() {
        return INSTANCE.newThreadScheduler;
    }
    ...
}
```

接着跟踪`RxJavaSchedulersHook.createNewScheduler()`，看看`newThreadScheduler`究竟是如何创建的?

我们发现无论是IO线程,Compute线程,还是NewThread线程调度器,都是`RxJavaSchedulersHook.createXXX()`方法创建出来了,其内部是用工厂方法实现的.



最终会找到一个叫`NewThreadScheduler`的类：



```java
public final class NewThreadScheduler extends Scheduler {
    private final ThreadFactory threadFactory;
    public NewThreadScheduler(ThreadFactory threadFactory) {
        this.threadFactory = threadFactory;
    }
    @Override
    public Worker createWorker() {
        return new NewThreadWorker(threadFactory);
    }
}
```

最终看到`NewThreadScheduler`就是我们调用`subscribeOn(Schedulers.newThread() )`传入的调度器对象，通过上面的分析,我们已经明白了` Scheduler` 的产生原理

**产生`Scheduler` 并不是最终目的,而是通过`Scheduler` 产生 `Worker`,然后调用`Worker.schedule(Action0)`实现线程的切换.**







###  Worker

通过上面的分析,我们已经明白了` Scheduler` 的产生原理,产生`Scheduler` 并不是最终目的,而是通过`Scheduler` 产生 `Worker`,然后调用`Worker.schedule(Action0)`实现线程的切换.

每个调度器对象都有一个`createWorker`方法用于创建一个`Worker`对象，而`NewThreadScheduler`对应创建的`Worker`是一个叫`NewThreadWorker`的对象.

而在上面的分析中我们也看到了, `OperatorSubscribeOn`类中调用了

`final Scheduler.Worker inner = scheduler.createWorker()`方法来得到一个 Worker,然后又调用 `inner.schedule(Action0)`实现线程的切换

接下来我们跟进`schedule()`方法查看其内部的实现原理.同样,这里的` Worker` 依然是以最简单的`NewThreadWorker` 为例.这里删减了部分代码,只留取对整体结构有用的部分.

```java
public class NewThreadWorker extends Scheduler.Worker implements Subscription {
    private final ScheduledExecutorService executor;   //线程池,在下面构造函数中进行初始化.
    public NewThreadWorker(ThreadFactory threadFactory) {
        //创建一个线程池
        ScheduledExecutorService exec = Executors.newScheduledThreadPool(1, threadFactory);
        executor = exec;
    }
  
  //默认调用的是这个方法;
    @Override
    public Subscription schedule(final Action0 action) {
        return schedule(action, 0, null);
    }
    @Override
    public Subscription schedule(final Action0 action, long delayTime, TimeUnit unit) {
        return scheduleActual(action, delayTime, unit);
    }
    
  
  //重要：其实 worker.schedule()最终调用的是这个方法
    public ScheduledAction scheduleActual(final Action0 action, long delayTime, TimeUnit unit) {
        //别紧张,源码中直接将传入的 action return回来了... 这一步相对于什么也没做;
        Action0 decoratedAction = schedulersHook.onSchedule(action);
        //ScheduledAction就是一个Runnable对象，在run()方法中调用了Action0.call()
        ScheduledAction run = new ScheduledAction(decoratedAction);
        Future<?> f;
        if (delayTime <= 0) {
            f = executor.submit(run);   //将Runnable对象放入线程池中
        } else {
            f = executor.schedule(run, delayTime, unit);  //延迟执行
        }
        run.add(f);

        return run;
    }
    ...
}
```

我们发现`OperatorSubscribeOn`计划表中通过`NewThreadWorker.schedule(Action0)`，将`Action0`作为参数传入一个`Runnable`的实现类:`ScheduledAction`,然后将这个runnable放入到一个线程池中执行，这样就实现了线程的切换。



简单说:最原始的`subscribeOn()`---调用了----`create(new OperatorSubscribeOn<T>(this, scheduler))`----创建一个代理被观察者---->`OperatorSubscribeOn()`中实现了`call()`方法---->call()方法中调用了`NewThreadWorker.schedule(Action0)`----`Action0`被包装称一个`RUnnable`对象,然后`schedule()`方法内部使用了线程池,创建一个新的线程,并将包装的`Runnable`对象传递进去,这样就实现了线程的切换



步骤:

1. 原始被观察者调用subscribeOn()方法准备切换线程,(这时候还没切换呢.)产生一个代理被观察者
2. 原始订阅者订阅代理被观察者(明面代码上你能看得到的)
3. 代理被观察者的`onSubscribe.call()`方法执行,提供了一个`Runnable`对象,也就是线程已经被切换了
4. **新线程**中产生一个新的代理观察者,代理观察者订阅原始被观察者(接下来的动作也都是在新线程中执行)
5. 原始被观察者发射数据,这个动作已经是在新线程中执行了
6. 代理观察者收到数据,再将数据转发给原始观察者

看这张图,帮助理解
![image](http://note.youdao.com/yws/public/resource/3a378f8ba029c8148bde85d73d9704c7/xmlnote/1853E77FF7BA42C5ABBAA8B9671EB41B/70)

> 此处用到了多线程的知识,多线程这一块还需要总结整理;



### 多次subscribeOn()的情况

我们发现，每次使用`subscribeOn`都会产生一个新的`Observable`，并产生一个新的计划表`OnSubscribe`，目标Subscriber最后订阅的将是最后一次`subscribeOn`产生的新的`Observable`。在每个新的`OnSubscribe`的`call`方法中都会有一个产生一个新的线程，在这个新线程中订阅上一级`Observable`，并创建一个新的`Subscriber`接受数据，最终原始`Observable`将在第一个新线程中发射数据，然后传送给给下一个新的观察者，直到传送到目标观察者，所以多次调用`subscribeOn`只有第一个起作用（这只是表面现象，其实每个`subscribeOn`都切换了线程，只是最终目标`Observable`是在第一个`subscribeOn`产生的线程中发射数据的）



也就是说多次调用`subscribeOn()`方法其实不是只有第一次方法其作用,而是每次都起作用,这里说的第一次起作用其实说的是最原始的数据发射是在第一次subscribeOn()指定的线程,只不过我们很少关注中间数据的处理过程而已;


一张图理解多订阅的过程:

![image](http://note.youdao.com/yws/public/resource/3a378f8ba029c8148bde85d73d9704c7/xmlnote/2AF59C14A4AC422F83E220C8C14059A8/68)

下面是多次线程切换的伪代码

```java
//第3个subscribeOn产生的新线程
new Thread(){
    @Override
    public void run() {
        Subscriber s1 = new Subscriber();
        //第2个subscribeOn产生的新线程
        new Thread(){
            @Override
            public void run() {
                Subscriber s2 = new Subscriber();
                //第1个subscribeOn产生的新线程
                new Thread(){
                    @Override
                    public void run() {
                        Subscriber<T> s3 = new Subscriber<T>(subscriber) {
                            @Override
                            public void onNext(T t) {
                                subscriber.onNext(t);
                            }
                            ...
                        };
                        //①. 最后一个新观察者订阅原始Observable
                        Observable.subscribe(s3);
                        //②. 原始Observable将在此线程中发射数据

                              //③. 最后一个新的观察者s3接受数据

                              //④. s3收到数据后，直接发送给s2，s2收到数据后传给s1,...最后目标观察者收到数据
                         } 
                }.start();
            }
        }.start();
    }
}.start();
```


# observeOn原理

> 还是需要进一步的整理


observeOn调用的是lift操作符。lift操作符创建了一个代理的Observable，用于接收原始Observable发射的数据，然后在Operator中对数据做一些处理后传递给目标Subscriber。observeOn一样创建了一个代理的Observable，并创建一个代理观察者接受上一级Observable的数据，代理观察者收到数据之后会开启一个线程，在新的线程中，调用下一级观察者的onNext、onCompete、onError方法。


```java
public final Observable<T> observeOn(Scheduler scheduler) {
    return observeOn(scheduler, RxRingBuffer.SIZE);
}

public final Observable<T> observeOn(Scheduler scheduler, int bufferSize) {
  	return observeOn(scheduler, false, bufferSize);
}

public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    if (this instanceof ScalarSynchronousObservable) {
      return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
    }
    return lift(new OperatorObserveOn<T>(scheduler, delayError, bufferSize));
}
```

可以看到使用observeOn(Scheduler scheduler)方法时,也是传入了一个scheduler,这和subscribeOn()方法如出一辙,,随着不断深入的调用,其最终使用 lift()操作符创建了一个Observable 对象.这里先不管lift,接着上面的lift()中创建了一个OperatorObserveOn类,其源码如下:

```java
public final class OperatorObserveOn<T> implements Observable.Operator<T, T> {
    private final Scheduler scheduler;
    //创建代理观察者，用于接收上一级Observable发射的数据,而这个child就是原始观察者.
    @Override
    public Subscriber<? super T> call(Subscriber<? super T> child) {
        if (scheduler instanceof ImmediateScheduler) {
            return child;
        } else if (scheduler instanceof TrampolineScheduler) {
            return child;
        } else {
            ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError, bufferSize);
            parent.init();
            return parent;
        }
    }

 //-----------------------------------我是分割线-------------------------------------------------------- 
  /*
  先不管前面的复杂逻辑了,总之现在有了代理被观察者和代理观察者,像map那样发生了订阅,然后原始被观察者开始发数据了
  在代理观察者中,自然也有onNext,onCompleted(),onError()方法,但是在这三个方法后都调用了 schedule()函数
  */
    //代理观察者
    private static final class ObserveOnSubscriber<T> extends Subscriber<T> implements Action0 {
        final Subscriber<? super T> child;
        final Scheduler.Worker recursiveScheduler;
        final NotificationLite<T> on;
        final Queue<Object> queue;
        //接受上一级Observable发射的数据
        @Override
        public void onNext(final T t) {
            if (isUnsubscribed() || finished) {
                return;
            }
            if (!queue.offer(on.next(t))) {
                onError(new MissingBackpressureException());
                return;
            }
   //在代理观察者中,自然也有onNext,onCompleted(),onError()方法,但是在这三个方法后都调用了 schedule()函数
            schedule();
        }
        @Override
        public void onCompleted() {
            ...
            schedule();
        }
        @Override
        public void onError(final Throwable e) {
            ...
            schedule();
        }
        //开启新线程处理数据,切换线程就是在这里,重要的方法.
        protected void schedule() {
            if (counter.getAndIncrement() == 0) {
                recursiveScheduler.schedule(this);
            }
        }
      
      
        // only execute this from schedule()
        //在新线程中将数据发送给目标观察者,注意这里是观察者,其call方法是因为实现了Action0接口,什么时候调用呢?
        @Override
        public void call() {
            long missed = 1L;
            long currentEmission = emitted;
            final Queue<Object> q = this.queue;
            final Subscriber<? super T> localChild = this.child;
            final NotificationLite<T> localOn = this.on;
            for (;;) {
                while (requestAmount != currentEmission) {
                    ...
                    localChild.onNext(localOn.getValue(v));
                }
            }
        }
    }
}
```

还记得`subscribeOn()`时传入的`Scheduler`吗,这个`observeOn()`也传入了一个`Scheduler`,和之前一样,通过这个`scheduler产生一个Worker`,然后调用`Worker.schedule(Action0)`实现线程的切换.与`subscribeOn()`不同的是,这个线程切换时在代理观察者执行`onNext()`中执行的,也就是说先把线程切换过去,然后代理观察者在执行的 `actual.onNext()`方法.



我们可以参照多次subscribeOn()的图解示例,可以把第二次subscribeOn()替换成observeOn(),那么在产生的第二个代理观察者给原始观察者发消息时,本来是在其onNext()方法中直接调用原始观察者的onNext()的,但是由于有observeOn(),所以在执行onNext的时候进行了线程切换,然后在调用原始观察者的onNext()

# 总结

只要涉及到操作符，其实就是生成了一套代理的`Subscriber`(观察者)、`Observable`(被观察者)和`OnSubscribe`(计划表)。`Observable`最典型的特征就是链式调用，我们暂且将每一步操作称为一级。代理的`OnSubscribe`中的`call`方法就是让代理`Subscriber`订阅上一级`Observable`，直到订阅到原始`Observable`发射数据，代理`Subscriber`收到数据后，可能对数据做一些操作也有可能切换线程，然后将数据传送给下一级`Subscriber`，直到目标观察者接收到数据，目标观察者在那个线程接受数据取决于上一个`Subscriber`在哪一个线程调用目标观察者的方法。


