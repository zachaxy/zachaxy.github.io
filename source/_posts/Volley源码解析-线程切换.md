---
title: Volley源码解析-线程切换
date: 2017-05-28 17:52:23
tags: 框架设计之美
---

# ResponseDelivery 接口

无论是执行网络请求还是从本地文件中读取缓存，都是放在子线程中去执行的，那么读取到响应之后，要做的工作就是把响应从子线程分发到 UI 线程，本节将学习 Volley 中是如何实现这一功能的。

这里用到了一个接口：**ResponseDelivery**，其中定义了一些方法，用来将响应从子线程发送的 UI 线程。

- `public void postResponse(Request<?> request, Response<?> response)`：将 response 发送到 UI 线程；
- `public void postResponse(Request<?> request, Response<?> response, Runnable runnable)`：和上一个方法功能类似，只不过多加了一个 Runnable，其作用是在 response 被发送到 UI 线程之后，执行该 runnable
- `public void postError(Request<?> request, VolleyError error)`：在网络请求过程中出现了错误，Volley 会将该错误封装成一个 VolleyError，调用该方法可以将该 VolleyError 发送的 UI 线程，由具体的业务处理不同的错误，显示不同的提示信息；

#  ExecutorDelivery 实现类

## 构造方法

而 ResponseDelivery 只有一个实现类——ExecutorDelivery，无论是在 NetworkDispatcher 中还是 CacheDispatcher 中得到了 response，都会调用该类的 postResponse 方法，将结果发送到 UI 线程，ExecutorDelivery 是何时被创建的呢？答案就是创建 RequestQueue 时，在 RequestQueue 的构造方法中。ExecutorDelivery 的创建方式是：

```java
new ExecutorDelivery(new Handler(Looper.getMainLooper()));
```

这里看到构造方法中传入了一个 Handler，并且传入的是 MainLooper，那么首先子线程向 UI 线程发送数据的机制肯定是通过 Handler 无疑了。

## 重要成员变量

ExecutorDelivery 中还有一个很重要的成员变量`Executor mResponsePoster`这是一个线程池，其创建过程也是在构造方法中：

```java
public ExecutorDelivery(final Handler handler) {
    // Make an Executor that just wraps the handler.
    mResponsePoster = new Executor() {
        @Override
        public void execute(Runnable command) {
            handler.post(command);
        }
    };
}
```

可以看出，ExecutorDelivery 中线程池 mResponsePoster 的作用就是在线程中调用 handler，发送 runnable，通过这种形式使得 runnable 在 UI 线程中执行。

## 内部类——Runnable 子类

上面提到线程池的内部是通过 handler，发送 runnable，通过这种形式使得 runnable 在 UI 线程中执行，那么这个 runnable 对象又是如何与 request 和 response 关联的呢？这就用到了 ExecutorDelivery 中的一个内部类 ResponseDeliveryRunnable

```java
private class ResponseDeliveryRunnable implements Runnable {
    private final Request mRequest;
    private final Response mResponse;
    private final Runnable mRunnable;

    public ResponseDeliveryRunnable(Request request, Response response, Runnable runnable) {
        mRequest = request;
        mResponse = response;
        mRunnable = runnable;
    }

    @SuppressWarnings("unchecked")
    @Override
    public void run() {
        // If this request has canceled, finish it and don't deliver.
        if (mRequest.isCanceled()) {
            mRequest.finish("canceled-at-delivery");
            return;
        }

        // Deliver a normal response or error, depending.
        if (mResponse.isSuccess()) {
            mRequest.deliverResponse(mResponse.result);
        } else {
            mRequest.deliverError(mResponse.error);
        }

        // If this is an intermediate response, add a marker, otherwise we're done
        // and the request can be finished.
        if (mResponse.intermediate) {
            mRequest.addMarker("intermediate-response");
        } else {
            mRequest.finish("done");
        }

        // If we have been provided a post-delivery runnable, run it.
        if (mRunnable != null) {
            mRunnable.run();
        }
   }
}
```

通过 ResponseDeliveryRunnable 的构造方法，将 request，response 传入，同时还可以在传入一个 Runnable 对象，这在 ResponseDelivery 的第二个接口中有提到过，该 Runnable 会在 response 被发送到 UI 线程后立即执行。

那么 ResponseDeliveryRunnable 类中的 run 方法则是整个线程切换的核心了；

要明确一点是，如果能执行到这里，那么 request 一定是得到 response 了

首先，判断该 request 是不是在得到请求还为发送到 UI 线程之前被取消了，如果是的话，执行 request 的 finish 方法

接着，如果 response 成功返回了，那么就调用 request 的 deliverResponse 方法，这个方法也很眼熟吧，这是我们在自定义 request 时必须要实现的方法，这个方法内部一般是回调 request 中的 listener 的 onResponse，否则，如果该 response 没有成功返回，会调用 request 的 deliverError 方法，而这个方法内部 request 的默认实现是调用 ErrorListener 的 onErrorResponse 方法。

最后调用 request 的 finish 方法，表示该请求已经执行结束了，同时，如果 ResponseDeliveryRunnable 的构造方法中的第三个参数 runnable 不为空，立即执行该 runnable 的 run 方法。

至此，在子线程中的 response 就被发送到 UI 线程中去了。其实说到这里，你可能发现，在 Android 体系中，很多框架都要在子线程中执行任务，然后将结果传输到 UI 线程，其处理的方式大多类似，那就是 "线程池" + "Handler" 的组合 。较为知名的框架`RxJava`，`EventBus`等都是这样实现的。关于 RxJava 的实现方式，请参考之前的一片文章[RxJava源码详解-线程切换原理](https://zachaxy.github.io/2017/04/03/RxJava%E6%BA%90%E7%A0%81%E8%AF%A6%E8%A7%A3-%E7%BA%BF%E7%A8%8B%E5%88%87%E6%8D%A2%E5%8E%9F%E7%90%86/)

