---
title: RxJava源码详解-基本使用
date: 2017-04-03 09:15:24
tags: 框架设计之美
---



# 观察者 VS 被观察者
以灯泡和开关为例子,这个例子中,开关是被观察者,灯泡是观察者,灯泡观察到开关执行了响应的操作,才执行响应的亮/灭的响应;

被观察者(Observable):事件的产生源头
观察者(Observer):事件的处理方;注册感兴趣的事件,一旦事件发生改变,观察者再做出相应的响应

在事件的起点到终点的传递过程中,我们可以进行相应的转换/加工/过滤等操作


在源码中,你可能会看到观察者有时候用Observer,有时候用Subscriber
其实:Observer是观察者的接口， Subscriber是实现这个接口的抽象类,
因此两个类都可以被当做观察者，由于 Subscriber 在 Observer 的基础上做了一些拓展，加入了新的方法，一般会更加倾向于使用Subscriber。
`abstract class Subscriber<T> implements Observer<T>, Subscription`

# RxJava 的使用

# 创建被观察者
这里分为三种方法:普通模式,偷懒模式1,偷懒模式2

## 普通模式
这里分为两种方法:普通模式,偷懒模式1;

## 订阅
建立观察者和被观察者的联系
`switcher.subscribe(light);`


这里可能存在一些疑惑,一般的写法是观察者订阅被观察者,而 RxJava 怎么反过来了
是这样的，台灯观察开关，逻辑是没错的，而且正常来看就应该是light.subscribe(switcher)才对，之所以“开关订阅台灯”，是为了保证 **流式API调用风格**


注: **当调用订阅操作即调用Observable.subscribe(Observe)方法的时候，被观察者才真正开始发出事件**

### 关于流式API调用风格

```
//这就是RxJava的流式API调用
Observable.just("On","Off","On","On")
    //在传递过程中对事件进行过滤操作
     .filter(new Func1<String, Boolean>() {
                @Override
                public Boolean call(String s) {
                    return s！=null;
                }
            })
    .subscribe(mSubscriber);
```

上面就是一个非常简易的RxJava流式API的调用：同一个调用主体一路调用下来，一气呵成。
所以为了保证流式API的调用风格,才用了这种反人类的逻辑;

由于被观察者产生事件，是事件的起点，那么开头就是用Observable这个主体调用来创建被观察者，产生事件，
为了保证流式API调用规则，就直接让Observable作为唯一的调用主体，一路调用下去。

# 操作符
操作符的分类:

- 转换类操作符:(map flatMap concatMap flatMapIterable switchMap scan groupBy...)；
- 过滤类操作符:(fileter take takeLast takeUntil distinct distinctUntilChanged skip skipLast ...)；
- 组合类操作符:(merge zip join combineLatest and/when/then switch startSwitch...)。

接下来就挑选几个常用的操作符进行讲解:

## map

提供一对一的转换,例如提供的是 url 路径,需要的是对应的 bitmap


## flatmap
将每个Observable产生的事件里的信息再包装成新的Observable传递出来,并且破除的多层嵌套的难题,
因为FlatMap可以再次包装新的Observable,而每个Observable都可以使用from(T[])方法来创建自己，这个方法接受一个列表，然后将列表中的数据包装成一系列事件。

还记得创建被观察者时的 from 方法吗,被观察者提供一个数组给 from 方法,被观察者都能将数组中的每个元素转换为被观察者,然后执行观察者的方法.

flatMap 每使用一个 from 方法,就该表一次 for 循环,将一个数组中的元素转换为单个被观察者,仅仅使用from,就该表一层循环,多层嵌套的循环就使用多次from()方法
是的代码看起来并没有嵌套的那么复杂,背后的原理就是使用from

所谓 flat就是:我们传入的是一个年级(包含多个班,每个班都多个同学,打印每个同学的姓名),第一步用form已经将年级拆成了多个班,但这不是终极目的,而是在将每个班的每个同学对应过来;回想之前的处理逻辑,每产生一个事件,不等待,直接发出去,但是现在不同,要等一个班的from结束之后(攒齐这个班所有同学,然后再一次发出去),接下来在解析下一个班

## concatMap
concatMap()解决了flatMap()的交叉问题，它能够把发射的值连续在一起.

## flatMapIterable
flatMapIterable()和flatMap()几乎是一样的，不同的是flatMapIterable()它转化的多个Observable是使用Iterable作为源数据的。(示例代码如下)并没有看出来什么区别,还是使用flatmap吧
```java
Observable.from(communities)
        .flatMapIterable(new Func1<Community, Iterable<House>>() {
            @Override
            public Iterable<House> call(Community community) {
                return community.houses;
            }
        })
        .subscribe(new Action1<House>() {

            @Override
            public void call(House house) {

            }
        });

```

## switchMap
switchMap()和flatMap()很像，除了一点：每当源Observable发射一个新的数据项（Observable）时，它将取消订阅并停止监视之前那个数据项产生的Observable，并开始监视当前发射的这一个。(也不懂什么使用场景,只知道是只监听当前的事件,只要一发送,过去的事件不在监听,只管理现在的事件)


## scan
scan()对一个序列的数据应用一个函数，并将这个函数的结果发射出去作为下个数据应用合格函数时的第一个参数使用.
```java
Observable.just(1, 2, 3, 4, 5)
        .scan(new Func2<Integer, Integer, Integer>() {
            @Override
            public Integer call(Integer integer, Integer integer2) {
                return integer + integer2;
          }
        }).subscribe(new Action1<Integer>() {
    @Override
    public void call(Integer integer) {
        System.out.print(integer+“ ”);
    }
});
```
输出结果是:1,3,6,10,15
1 = 1
3 = 1+2
6 = 3+3
10 = 6+4
15 = 10+5;
实现的函数是两个整数相加,第一个整数时,默认第0个数是0,后面进来一个数就和当前计算的结果相加;

## groupBy
groupBy()将原始Observable发射的数据按照key来拆分成一些小的Observable，然后这些小Observable分别发射其所包含的的数据，和SQL中的groupBy类似。实际使用中，我们需要提供一个生成key的规则（也就是Func1中的call方法），所有key相同的数据会包含在同一个小的Observable中。另外我们还可以提供一个函数来对这些数据进行转化，有点类似于集成了flatMap。这只是在顺序上进行了调整,在观察者方收到数据的顺序是相同key的是临近的.

```java
List<House> houses = new ArrayList<>();
houses.add(new House("中粮·海景壹号", "中粮海景壹号新出大平层！总价4500W起"));
houses.add(new House("竹园新村", "满五唯一，黄金地段"));
houses.add(new House("中粮·海景壹号", "毗邻汤臣一品"));
houses.add(new House("竹园新村", "顶层户型，两室一厅"));
houses.add(new House("中粮·海景壹号", "南北通透，豪华五房"));
Observable<GroupedObservable<String, House>> groupByCommunityNameObservable = Observable.from(houses)
        .groupBy(new Func1<House, String>() {

            @Override
            public String call(House house) {
            //生成key的规则,根据communityName相同的分为一组;
                return house.communityName;
            }
        });
```

执行结果:

```
小区:中粮·海景壹号; 房源描述:中粮海景壹号新出大平层！总价4500W起
小区:中粮·海景壹号; 房源描述:毗邻汤臣一品
小区:中粮·海景壹号; 房源描述:南北通透，豪华五房
小区:竹园新村; 房源描述:满五唯一，黄金地段
小区:竹园新村; 房源描述:顶层户型，两室一厅
```



## **Filter**

**filter(Func1)**用来过滤观测序列中我们不想要的值，只返回满足条件的值

还是拿前面文章中的小区Community[] communities来举例，假设我需要赛选出所有房源数大于10个的小区，我们可以这样实现：

```java
Observable.from(communities)
        .filter(new Func1<Community, Boolean>() {
            @Override
            public Boolean call(Community community) {
                return community.houses.size()>10;
            }
        }).subscribe(new Action1<Community>() {
    @Override
    public void call(Community community) {
        System.out.println(community.name);
    }
});
```



## take

**take(int)**用一个整数n作为一个参数，从原始的序列中发射前n个元素

```java
Observable.from(communities)
    .take(10)
    .subscribe(new Action1<Community>() {
      @Override
      public void call(Community community) {
        System.out.println(community.name);
      }
  });
```

**但是后面的数据怎么办?**

## takeLast

**takeLast(int)**同样用一个整数n作为参数，只不过它发射的是观测序列中后n个元素。

## takeUntil

**takeUntil(Observable)**订阅并开始发射原始Observable，同时监视我们提供的第二个Observable。如果第二个Observable发射了一项数据或者发射了一个终止通知，takeUntil()返回的Observable会停止发射原始Observable并终止。

```java
Observable<Long> observableA = Observable.interval(300, TimeUnit.MILLISECONDS);
Observable<Long> observableB = Observable.interval(800, TimeUnit.MILLISECONDS);

observableA.takeUntil(observableB)
        .subscribe(new Subscriber<Long>() {
            @Override
            public void onCompleted() {
                System.exit(0);
            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(Long aLong) {
                System.out.println(aLong);
            }
        });

try {
    Thread.sleep(Integer.MAX_VALUE);
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

程序的输出:0,1



**takeUntil(Func1)**还可以通过Func1中的call方法来判断是否需要终止发射数据。

```java
Observable.just(1, 2, 3, 4, 5, 6, 7)
                .takeUntil(new Func1<Integer, Boolean>() {
                    @Override
                    public Boolean call(Integer integer) {
                        return integer >= 5;
                    }
                }).subscribe(new Action1<Integer>() {
            @Override
            public void call(Integer integer) {
                System.out.println(integer);
            }
        });
```

输出结果:1,2,3,4



 

# 线程调度

异步是相对于主线程来讲的子线程操作，在这里我们不妨使用线程调度这个概念更加贴切。

Scheduler:调度器,用于线程控制
- Scheduler.immediate()  默认当前线程,不写就是这个,一般不写
- Scheduler.newThread()  每次都创建新线程去执行,消耗大,不建议用
- Scheduler.io()  IO密集型任务(eg:异步阻塞IO操作),默认是一个 CacheThreadScheduler,从线程池中去取一个线程,
- Scheduler.computation() CPU密集计算线程,线程池中的线程数和CPU核心数一致,多用于处理图形界面大量的计算或者事件循环;
- Scheduler.trampoline() 当其它排队的任务完成后,当前线程排队开始执行;
- AndroidScheduler.mainThread() Android 的 UI 线程

实际上线程调度只有subscribeOn（）和observeOn（）两个方法。对于初学者，只需要掌握两点：

1. subscribeOn（）它指示Observable在一个指定的调度器上创建（只作用于被观察者创建阶段）。只能指定一次，如果指定多次则以第一次为准
2. observeOn（）指定在事件传递（加工变换）和最终被处理（观察者）的发生在哪一个调度器。可指定多次，每次指定完都在下一步生效。

线程调度掌握到这个程度，在入门阶段时绝对够用的了。

```java
 Observable.just(getFilePath())
           //指定在新线程中创建被观察者 Observable
          .subscribeOn(Schedulers.newThread())
      //将接下来执行的线程环境指定为io线程,必须要先制定下面操作的线程,然后再指定操作;
          .observeOn(Schedulers.io())
            //map就处在io线程
          .map(mMapOperater)
            //将后面执行的线程环境切换为主线程，
            //但是这一句 observeOn 依然执行在io线程
          .observeOn(AndroidSchedulers.mainThread())
          //指定线程无效，但这句代码本身执行在主线程,一个操作只能制定一个 subscribeOn,多了以第一个为准;
          .subscribeOn(Schedulers.io())
          //执行在主线程
          .subscribe(mSubscriber);
```