---
title: Volley源码解析-缓存
date: 2017-05-23 14:48:12
tags: 框架设计之美
---

# Cache 接口

我们再来回顾一下一个 Request 被添加到 RequestQueue 后，首先是被添加到 mCacheQueue 中，而不是添加到 mNetworkQueue，因为这个请求可能之前执行过，先看一下缓存中有没有，如果没有或者过期，那么再将该请求从 mCacheQueue 中移除，并添加到 mNetworkQueue。

那么 Cache 中保存了哪些信息呢？

代码过多这里就不贴出来了，首先作为缓存，肯定要指定一个缓存文件夹以及其保存的位置，接着每个缓存最好是以键值对的形式存在，使用 HashMap 结构来保存，这样才能快速的通过 key 查找 value，key 对应的是一个 request 的 url，value 对应的是 request 得到的 response；

接下来，response 中需要保存哪些数据呢？首先肯定要保存响应体中的数据，使用字节数组保存。同时还应该保存这个请求的 Etag，有效时长等和 HTTP 缓存相关的字段，以及其它的响应头信息，这些信息被封装在一个 Entry 的类中，该类是 Cache 接口中的静态内部类

```java
public static class Entry {
    /** The data returned from cache. */
    public byte[] data;

    /** ETag for cache coherency. */
    public String etag;

    /** Date of this response as reported by the server. */
    public long serverDate;

    /** TTL for this record. */
    public long ttl;

    /** Soft TTL for this record. */
    public long softTtl;

    /** Immutable response headers as received from server; must be non-null. */
    public Map<String, String> responseHeaders = Collections.emptyMap();

    /** True if the entry is expired. */
    public boolean isExpired() {
        return this.ttl < System.currentTimeMillis();
    }

    /** True if a refresh is needed from the original data source. */
    public boolean refreshNeeded() {
        return this.softTtl < System.currentTimeMillis();
    }
}
```





## DiskBasedCache

前面讲了 Cache 的整体实现思路，接下来设计到具体如何读写缓存，这就用到了 Cache 的实现类 DiskBasedCache

这里只列出方法名和内部实现流程，就不一行一行的写了，因为太多了！！！

- `private final Map<String, CacheHeader> mEntries = new LinkedHashMap<String, CacheHeader>(16, .75f, true)`是整个缓存的 map

- `CacheHeader`是map 中保存的 value，DiskBasedCache 中的静态内部类，是缓存文件中内容的一个概述，不包含 Cache 中 Entry 类中的 data 数据，因为这是要读取到内存中的，如果带上响应体数据，那么会占用很大一部分内存。

   Cache.Entry 和 DiskBasedCache.CacheHeader 之间的关系：**Entry = CacheHeader + byte[] data**

  - `writeHeader(OutputStream os)`:直接看其是如何将 CacheHeader 写到到文件中的：

    ```java
    public boolean writeHeader(OutputStream os) {
    	try {
    		writeInt(os, CACHE_MAGIC);
    		writeString(os, key);
    		writeString(os, etag == null ? "" : etag);
    		writeLong(os, serverDate);
    		writeLong(os, ttl);
    		writeLong(os, softTtl);
    		writeStringStringMap(responseHeaders, os);
    		os.flush();
    		return true;
    	} catch (IOException e) {
    		VolleyLog.d("%s", e.toString());
    		return false;
    	}
    }
    ```

  - `readHeader(InputStream is)`和上面的 write 方法是相反的，怎么写的怎么读

    ```java
    public static CacheHeader readHeader(InputStream is) throws IOException {
    	CacheHeader entry = new CacheHeader();
    	int magic = readInt(is);
    	if (magic != CACHE_MAGIC) {
    		// don't bother deleting, it'll get pruned eventually
    		throw new IOException();
    	}
    	entry.key = readString(is);
    	entry.etag = readString(is);
    	if (entry.etag.equals("")) {
    		entry.etag = null;
    	}
    	entry.serverDate = readLong(is);
    	entry.ttl = readLong(is);
    	entry.softTtl = readLong(is);
    	entry.responseHeaders = readStringStringMap(is);
    	return entry;
    }
    ```

    这里有一个很有意思的**写缓存文件的技巧**，因为这里涉及到了写 int，long，String，那么如何去写呢？ 方法是：如果你写的是 int，那么就用 4 字节来表示，如果是 long，那么就用 8 字节来表示，这样在读的时候，只要是读 int，那么直接向后读 4 个字节，然后拼装成 int 即可；至于字符串，统一将其转换成 UTF8 格式的字节数据，并得到数组长度，首先写入 long 类型的长度，然后再将字节数组写入，在读取的时候先读取 8 字节的 long 值，得到后续字节数组的长度，然后在读取后面相应长度的字节数组，转换为字符串即可。

- `public synchronized void initialize()`初始化上面的 map，遍历缓存文件夹，读取每个缓存文件，然后不断的调用 CacheHeader 的 readHeader 方法读取 CacheHeader，并添加到 map 中

- `public synchronized void put(String key, Entry entry)`首先调用 CacheHeader 的 writeHeader 方法将数据写入缓存文件，在将 data 数据额外写入缓存文件；同时将该 entry 对应的 entryHeader 添加到 map 中；

- `public synchronized Entry get(String key)`先从 map 中通过 key 得到 CacheHeader，在根据 key 得到缓存文件名（将 key 的字符串均分为前后两部分，前半部分的 hashCode 和后半部分的 hashCode 拼接起来），然后读取缓存文件中的 data 数据，和 CacheHeader 组合起来得到 Entry。

- ` public synchronized void remove(String key)`先从 map 中删除掉 key 对应的数据，然后删除缓存文件中对应的文件

- `public synchronized void clear()`：清空缓存，可以在应用的设置中提供这一项。有一个对应的 ClearCacheRequest，专门负责清空缓存。



# CacheDispatcher 线程

## 构造方法

CacheDispatcher 线程是会不断的执行读缓存的操作，如果能读到缓存，那么将数据用 ResponseDelivery 发送到 UI 线程，否则将 request 提交给 NetworkDispatcher 来处理。

首先看其构造方法，传入了在 RequestQueue 中创建的 mCacheQueue, mNetworkQueue, mCache, mDelivery 四个变量。注意这个 mCache 是在 RequestQueue 的构造方法中传入的第一个参数 Cache。也就是说 CacheDispatcher 内部只有这几个成员变量。

```java
public CacheDispatcher(
        BlockingQueue<Request> cacheQueue, BlockingQueue<Request> networkQueue,
        Cache cache, ResponseDelivery delivery) {
    mCacheQueue = cacheQueue;
    mNetworkQueue = networkQueue;
    mCache = cache;
    mDelivery = delivery;
}
```



## run 方法

接下来看一下其 run 方法：

```java
@Override
public void run() {
    if (DEBUG) VolleyLog.v("start new dispatcher");
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

    // Make a blocking call to initialize the cache.
    mCache.initialize();

    while (true) {
        try {
            // Get a request from the cache triage queue, blocking until
            // at least one is available.
            final Request request = mCacheQueue.take();
            request.addMarker("cache-queue-take");

            // If the request has been canceled, don't bother dispatching it.
            if (request.isCanceled()) {
                request.finish("cache-discard-canceled");
                continue;
            }

            // Attempt to retrieve this item from cache.
            Cache.Entry entry = mCache.get(request.getCacheKey());
            if (entry == null) {
                request.addMarker("cache-miss");
                // Cache miss; send off to the network dispatcher.
                mNetworkQueue.put(request);
                continue;
            }

            // If it is completely expired, just send it to the network.
            if (entry.isExpired()) {
                request.addMarker("cache-hit-expired");
                request.setCacheEntry(entry);
                mNetworkQueue.put(request);
                continue;
            }

            // We have a cache hit; parse its data for delivery back to the request.
            request.addMarker("cache-hit");
            Response<?> response = request.parseNetworkResponse(
                    new NetworkResponse(entry.data, entry.responseHeaders));
            request.addMarker("cache-hit-parsed");

            if (!entry.refreshNeeded()) {
                // Completely unexpired cache hit. Just deliver the response.
                mDelivery.postResponse(request, response);
            } else {
                // Soft-expired cache hit. We can deliver the cached response,
                // but we need to also send the request to the network for
                // refreshing.
                request.addMarker("cache-hit-refresh-needed");
                request.setCacheEntry(entry);

                // Mark the response as intermediate.
                response.intermediate = true;

                // Post the intermediate response back to the user and have
                // the delivery then forward the request along to the network.
                mDelivery.postResponse(request, response, new Runnable() {
                    @Override
                    public void run() {
                        try {
                            mNetworkQueue.put(request);
                        } catch (InterruptedException e) {
                            // Not much we can do about this.
                        }
                    }
                });
            }

        } catch (InterruptedException e) {
            // We may have been interrupted because it was time to quit.
            if (mQuit) {
                return;
            }
            continue;
        }
    }
}
```

第 4 行，设置当前线程的优先级为：Process.THREAD_PRIORITY_BACKGROUND，标准后台线程，相对来说优先级还是比较低的。

第 7 行，因为这里是 Cache，需要先进行一些初始化，这里调用了 Cache 的 initialize 方法，内部逻辑在上面已经说过了。

第 9 行，死循环，表明该线程不一直不断的从 mCacheQueue 中取 request

第 13 行，从 BlockingQueue 中取出一个 request，如果没有的话，就会被阻塞，也不会占用系统资源，赞。

第 17 行，如果该请求在执行之前，又被其他线程调用了 cancle 方法，那么就取消该 request 的执行。

第 23 行，从缓存中查看是否有该 request 的缓存，如果不存在，表明是一个新的请求，那么直接将该请求添加到 mNetworkQueue 中，下面的逻辑不用执行，重新从 BlockingQueue 中取 request

第 32 行，能走到这一行，表明缓存不为空，判断该缓存是否已经过期，这里判断的标准是：`this.ttl < System.currentTimeMillis();`现在 HTTP1.1 的版本里面保存的都是存活时间，而不是绝对时间，但是考虑到我们的客户端可能会退出，所以无法用时间段来统计，这里还是转换成相对于客户端的绝对时间，再和当前时间进行比较。如果缓存过期了，那么需要将该 request 添加到 mNetworkQueue 中重新进行网络请求，但是在此之前，还要把缓存中的一些信息("If-None-Match"<=>"etag"，"If-Modified-Since"<=>"serverDate")附加到 request 中

第 41 行，能走到这一步，说明我们的缓存命中并且是可用的，此时需要把缓存中的数据读取出来，包装成 Response

`Response<?> response = request.parseNetworkResponse(new NetworkResponse(entry.data, entry.responseHeaders));`

第 45 行，虽然现在缓存已被转换为 Response，但是还要进行一步新鲜度验证，但是这一步新鲜度验证时多余的，因为 Entry 中的 softTtl 和 ttl 的值是相同的，而前面已经验证了是否过期，所以这一步是多余的，直接使用 ResponseDelivery 分发响应，关于响应的分发，请参阅[Volley 源码解析-线程切换]()

如果不是新鲜的，那么首先将该 response 分发到 UI 线程显示，同时立即执行网络请求，获取最新的响应。
