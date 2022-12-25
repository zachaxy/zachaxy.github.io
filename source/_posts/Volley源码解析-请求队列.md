---
title: Volley源码解析-请求队列
date: 2017-05-22 19:23:17
tags: 框架设计之美
---

# 整体理解

[上一节](https://zachaxy.github.io/2017/05/20/Volley%E6%A1%86%E6%9E%B6%E8%AF%A6%E8%A7%A3-%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8/#自定义请求)，我们介绍了 Volley 的基本使用，再次回顾一下其整体流程：

1. 创建 RequestQueue 队列
2. 创建 XXXRequest
3. 将 2 中创建的 request 添加到 1 中创建的 RequestQueue 队列中

整个流程就这么简单，添加到 RequestQueue 队列之后，队列中的请求就会自动被执行，我们只需要等待请求中的回调方法被调用即可。那么其内部是如何实现的呢？猜想：后台一定有一个子线程，不断的轮询 RequestQueue，如果其中有请求，那么就会执行网络请求，因为这是在子线程中执行，所以不会阻塞 UI 线程，等服务器有响应后，执行请求中的回调。所以如果让我们来设计一套网络请求框架的话，最基本的一定是这个思路。

那还等什么，马上打开源码验证一下我们的猜想对不对吧！等等，似乎还少了什么，有可能会出现这种情况，如果我们重复的执行某一个相同的请求（eg：不断的打开相同的图片），图片资源一般在短时间内很少改变的，如果我们重复执行网络请求，似乎不是很合理，所以对于相同的网络请求（相同的 url），我们应该将结果缓存起来，这样也符合 http 协议中缓存的思想，这样就避免消耗过多的流量。



那么接下来，我们就按照 Volley 使用的三步流程开始，进入源码，剖析 Volley 的内部流程，首先介绍的是创建请求队列：

# 创建 RequestQueue 队列

在介绍 RequestQueue 队列之前，这里先介绍几个关键的类，以便于我们更好的理解 Volley。



- **HttpStack：**处理 Http 请求，返回请求结果。目前 Volley 中有基于 HttpURLConnection 的`HurlStack`和 基于 Apache HttpClient 的`HttpClientStack`。关于 HttpStack 的详细理解，请参考[Volley 源码解析-网络]()
- **Volley：**Volley 对外暴露的 API，通过 newRequestQueue 方法新建并启动一个请求队列`RequestQueue`。
- **Request：**表示一个请求的抽象类。`StringRequest`、`JsonRequest`、`ImageRequest` 都是它的子类，表示某种类型的请求。
- **ResponseDelivery：**返回结果分发接口，目前只有基于`ExecutorDelivery`的在入参 handler 对应线程内进行分发。
- **Network：**调用`HttpStack`处理请求，并将结果转换为可被`ResponseDelivery`处理的`NetworkResponse`。
- **Cache：**缓存请求结果，Volley 默认使用的是基于 sdcard 的`DiskBasedCache`。`NetworkDispatcher`得到请求结果后判断是否需要存储在 Cache，`CacheDispatcher`会从 Cache 中取缓存结果。


创建请求队列是我们在应用程序中是这样调用的：

```java
RequestQueue mQueue = Volley.newRequestQueue(context);
```

其内部调用的 Volley 类中的静态方法，源码如下：

```java
public static RequestQueue newRequestQueue(Context context) {
    return newRequestQueue(context, null);
}
```

我们默认值传入了第一个参数 Context，第二个参数 HttpStack 默认为 null，接下来调用下面的方法：

```java
public static RequestQueue newRequestQueue(Context context, HttpStack stack) {
    File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

    String userAgent = "volley/0";
    try {
        String packageName = context.getPackageName();
        PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
        userAgent = packageName + "/" + info.versionCode;
    } catch (NameNotFoundException e) {
    }

    if (stack == null) {
        if (Build.VERSION.SDK_INT >= 9) {
            stack = new HurlStack();
        } else {
            // Prior to Gingerbread, HttpUrlConnection was unreliable.
            // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
            stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
        }
    }

    Network network = new BasicNetwork(stack);
    Cache cache = new DiskBasedCache(cacheDir);

    RequestQueue queue = new RequestQueue(cache, network);
    queue.start();

    return queue;
}
```

第 2 行，创建一个文件夹，保存网络的请求缓存，默认路径是在：“/data/data/com.xxx.xxx（当前包）/cache/volley/”

第 4 行，创建了 User Agent，其默认值是："volley/0"，接下来获取本应用的包名，如果可以获取到，那么将 User Agent 设置为：App 的 packageName/versionCode，这个 User Agent 用来设置 http 请求头的 User-Agent 字段。

第 12 行，因为我们传入的 HttpStack 为 null，所以就根据当前系统的版本(即 API Level >= 9)，采用基于 HttpURLConnection 的 HurlStack，如果小于 9，采用基于 HttpClient 的 HttpClientStack。我们知道在 Android 中默认有两种。关于为何在不同的版本使用不同的 HttpStack，参考  [Android 访问网络，使用 HttpURLConnection 还是 HttpClient？](http://blog.csdn.net/guolin_blog/article/details/12452307)，关于 HttpStack 的详细讲解，请参考[Volley 源码解析-网络]()

第 22 行，创建了一个 NetWork 对象，其实 NetWork 是一个接口，其实现类只有 BasicNetwork；简单介绍一下 NetWork 接口，其内部只有一个方法 performRequest，而实现类 BasicNetwork 其内部调用 HttpStack 处理请求来实现 NetWork 中的方法，并将结果转换为可被 ResponseDelivery 处理的 NetworkResponse。

第 23 行，创建了一个 Cache 对象，其实 Cache 也是一个接口，其实现类只有 DiskBasedCache；简单介绍一下 Cache，Volley 默认使用的是基于 sdcard 的 DiskBasedCache。Cache 接口中包含了对于缓存文件中初始化，添加缓存，超找缓存，删除缓存等常用操作。并不执行网络请求。

第 25 行，终于看到了 RequestQueue 的创建了，其构造方法将上面创建的 Cache 和 Network 的实例传入，**注意这里传入的是父类引用（接口），也就是说执行网络请求或者缓存读取的业务逻辑都是以接口的形式来提供的，所以后面的编码都是基于接口的，而不针对具体实现类，从而达到了高扩展性。**

在 Volley 中的静态方法 newRequestQueue 中，创建 RequestQueue 的方法是使用了`new RequestQueue(cache, network)`，那么进入源码看一下其是如何创建的：

```java
public RequestQueue(Cache cache, Network network) {
    this(cache, network, DEFAULT_NETWORK_THREAD_POOL_SIZE);
}

public RequestQueue(Cache cache, Network network, int threadPoolSize) {
  	this(cache, network, threadPoolSize,new ExecutorDelivery(new Handler(Looper.getMainLooper())));
}

public RequestQueue(Cache cache, Network network, int threadPoolSize,ResponseDelivery delivery) {
	mCache = cache;
	mNetwork = network;
	mDispatchers = new NetworkDispatcher[threadPoolSize];
	mDelivery = delivery;
}
```

源码中创建队列是使用的 RequestQueue 最简单的构造方法，通过一些列不断的调用，最终调用了四个参数的方法，这里先关注一下**ResponseDelivery**这个类，该类的作用是将网络请求的结果或者从缓存读取数据的结果分发到主线程。具体实现在后面分析。

不要被 RequestQueue 的类名所迷惑，RequestQueue 本身并不是一个队列，而是其内部包含了若干队列，这些队列分别有不同的作用，具体内部队列在下一节介绍；

第 26 行，执行了 queue 的 start 方法，这里轻描淡写的一句 start 确包含了整个 Volley 框架的所有业务流程。先简单说一下 start 内部的流程，RequestQueue 中其实包含了两种线程，一种用来执行网络请求，一种用来从本地文件中读取缓存，这两种线程不断的从 RequestQueue 中取出 request，然后执行该 request。这个 start 方法内部其实就是将这两类线程运行起来。

第 28 行，返回最终创建的 RequestQueue。





# 向 RequestQueue 中添加请求

虽然我们创建了 RequestQueue 队列之后，其内部的线程就已经开启了，这里先不讨论细节，在接下来开启线程的部分讨论。因为创建了队列后，队列中并没有 request，所以不会执行任何操作。这里先看一下我们使用 Volley 的第三步，向队列中添加请求，调用的是 RequestQueue 中的 add 方法：

在介绍 add 方法之前，这里还有两个重要的变量需要介绍一下，分别是：mCacheQueue 和 mNetworkQueue；这是两个优先级队列，是 RequestQueue 的成员变量，其直接在 RequestQueue 创建的时候进行初始化：

这里设计为优先级队列，主要是考虑到我们的 request 是有优先级的，有的请求可能想立即被执行，那么就将其优先级调大一些，从而获得优先执行权。

```java
/** The cache triage queue. */
private final PriorityBlockingQueue<Request> mCacheQueue =
    new PriorityBlockingQueue<Request>();

/** The queue of requests that are actually going out to the network. */
private final PriorityBlockingQueue<Request> mNetworkQueue =
    new PriorityBlockingQueue<Request>();
```

同时还有两个成员变量，mWaitingRequests 和 mCurrentRequests；

```java
private final Map<String, Queue<Request>> mWaitingRequests = new HashMap<String, Queue<Request>>();
private final Set<Request> mCurrentRequests = new HashSet<Request>();
```

上述四个变量的作用：

- **mCacheQueue**：
- **mNetworkQueue**：保存需要真正执行网络请求的 request
- **mWaitingRequests**：
- **mCurrentRequests**：HashSet，用来保存添加到 RequestQueue 中的请求，只要该请求还没有响应，就一直在 mCurrentRequests 中保存。



接下来我们进入 add 方法中看一下其具体的流程：

```java
public Request add(Request request) {
	// Tag the request as belonging to this queue and add it to the set of current requests.
	request.setRequestQueue(this);
	synchronized (mCurrentRequests) {
		mCurrentRequests.add(request);
	}

	// Process requests in the order they are added.
	request.setSequence(getSequenceNumber());
	request.addMarker("add-to-queue");

	// If the request is uncacheable, skip the cache queue and go straight to the network.
	if (!request.shouldCache()) {
		mNetworkQueue.add(request);
		return request;
	}

	// Insert request into stage if there's already a request with the same cache key in flight.
	synchronized (mWaitingRequests) {
		String cacheKey = request.getCacheKey();
		if (mWaitingRequests.containsKey(cacheKey)) {
			// There is already a request in flight. Queue up.
			Queue<Request> stagedRequests = mWaitingRequests.get(cacheKey);
			if (stagedRequests == null) {
				stagedRequests = new LinkedList<Request>();
			}
			stagedRequests.add(request);
			mWaitingRequests.put(cacheKey, stagedRequests);
			if (VolleyLog.DEBUG) {
				VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
			}
		} else {
			// Insert 'null' queue for this cacheKey, indicating there is now a request in
			// flight.
			mWaitingRequests.put(cacheKey, null);
			mCacheQueue.add(request);
		}
		return request;
	}
}
```

第 2 行，该 request 与当前 RequestQueue 进行关联绑定，标识该 request 是被添加到哪个 RequestQueue 中的。

第 5 行，将 request 添加到 mCurrentRequests 队列中。只要该请求还没有响应，就一直在 mCurrentRequests 中保存。

第 9、10 行，对添加进来的 request 设置一个序列号，作为区分不同的请求。addMarker 方法也会在后面的源码中经常遇到，其实就是为当前的 request 添加一个状态的描述，此时添加的描述为"add-to-queue"，表明该请求才刚被添加进来，还未执行。

第 13 行，判断该 request 是否是可以缓存的，如果我们不做特殊设置，所有的 request 默认都是可以缓存的，这样做是为了减少流量的使用。当然如果你的应用对实时性要求很高，每次请求都要获取服务器最新的数据，那么可以将调用`request.setShouldCache(false);`取消缓存，这样的话，该 request 就会被添加到 mNetworkQueue 队列中，那么 RequestQueue 的 add 方法就结束了

第 19 行，能走到这一步，说明 request 是可以被缓存的，那么首先获取该 request 的 getCacheKey 方法，该方法简单的返回了该请求的 url，接下来就判断 mWaitingRequests(HashMap)中是否存在以该 request 的 url 作为 key 的值，这里分两种情况来说，如果当前 request 是一个全新的请求，那么 mWaitingRequests 中肯定不存在该 key 的，所以走 else 分支，如果存在，那么说明我们之前执行过相同的请求，走 if 分支；接下来分别看一下这两个分支

第 31 行，这是一个全新的请求，那么在 mWaitingRequests 中，将该 request 的 url 作为 key 添加进去，但是其 value 设置 null，并将该 request 添加到 mCacheQueue 中。

第 21 行，这不是一个新的请求，之前执行过。那么拿到该 key 对应的 value，其 value 值是一个`LinkedList<Request>`,然后将该请求添加到这个 value 中。这里要注意的是：如果是第二次执行这个请求，会走到该 if 分支，但是从 mWaitingRequests 中获取 value 时是空的，因为第一次执行该请求时，走的是 else 分支，当时只是把 key 添加进来了，value 是空的，所以如果是第二次执行的话，获取的是一个空的 LinkedList，此时需要重新创建一个 LinkedList，并将自己添加进去，那么及诶按来如果是第三次，第四次等添加该请求时，就可以直接拿到 LinkedList，然后直接添加进去即可。



**总结**

request 被添加到 RequestQueue 中时，所经历的过程：

![](http://oi8e3sh89.bkt.clouddn.com/image/%E6%A1%86%E6%9E%B6/Volley%E5%86%85%E9%83%A8%E6%B5%81%E7%A8%8B.png)





# 开启请求队列中的线程

前面我们讲解了 RequestQueue 的创建时，调用了其 start 方法，这个方法虽然只有一句，但是其内部却是整个框架的核心，一起来看一下 start 方法：

```java
public void start() {
    stop();  // Make sure any currently running dispatchers are stopped.
    // Create the cache dispatcher and start it.
    mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
    mCacheDispatcher.start();

    // Create network dispatchers (and corresponding threads) up to the pool size.
    for (int i = 0; i < mDispatchers.length; i++) {
        NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,mCache, mDelivery);
        mDispatchers[i] = networkDispatcher;
        networkDispatcher.start();
    }
}
```

这里涉及到两个重要的线程，我们在文章刚一开始根据 Volley 的使用方法，猜想其内部实现时也说到了，Volley 内部一定有一个线程不断的从 RequestQueue 中出请求，然后执行该请求。同时提供了缓存机制，就需要另一个线程从本地缓存中去取缓存。这分别对应了两个线程调度器：**CacheDispatcher**和**NetworkDispatcher**。

这里先说一下这两个调度器的作用：

- **CacheDispatcher：**（extends Thread），用于调度处理走缓存的请求。启动后会不断从缓存请求队列中取请求处理，队列为空则等待，请求处理结束则将结果传递给`ResponseDelivery`去执行后续处理。当结果未缓存过、缓存失效或缓存需要刷新的情况下，该请求都需要重新进入`NetworkDispatcher`去调度处理。
- **NetworkDispatcher**：（extends Thread），用于调度处理走网络的请求。启动后会不断从网络请求队列中取请求处理，队列为空则等待，请求处理结束则将结果传递给`ResponseDelivery`去执行后续处理，并判断结果是否要进行缓存。

那么在 start 方法中：

第 3 行，首先创建了一个 mCacheDispatcher 线程的实例，然后让该线程运行起来。

第 8 行，创建了 4 个 NetworkDispatcher 的线程实例，然后让着 4 个线程运行起来。这里为什么是 4 个，这是因为在创建 RequestQueue 的时候使用的是最简单的构造方法，只传入了 Cache 和 Network 两个参数。而 threadPoolSize 的参数使用了默认的 4，如果你想改变，可以使用 RequestQueue 三个参数的构造方法，第三个参数来指定 NetworkDispatcher 线程的数量。

目前总共 5 个线程就这样运行起来了，后面会用两个章节来分别讲解 Volley 中的网络请求（Network）和缓存（Cache）相关的部分，敬请期待。