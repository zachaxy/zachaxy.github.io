---
title: Volley源码解析-网络
date: 2017-05-24 22:31:01
tags: 框架设计之美
---

-------

# NetWork 接口

一个 request 并不是一开始就被添加进 mNetworkQueue 中的，而是先添加进 mCacheQueue，如果缓存不存在或者缓存失效，才会走网络路线，这里一旦用到了网络，就用到了 NetWork 接口，该接口是整个 Volley 库的网络请求的最外层的封装，其内部只包含了一个方法：

```java
public interface Network {
    /**
     * Performs the specified request.
     * @param request Request to process
     * @return A {@link NetworkResponse} with data and caching metadata; will never be null
     * @throws VolleyError on errors
     */
    public NetworkResponse performRequest(Request<?> request) throws VolleyError;
}
```

执行网络请求，返回 NetWorkResponse，而该接口只有一个实现类——BasicNetwork，那么就看一下 BasicNetwork 中是如何实现该方法，执行网络请求的。



## BasicNetwork

```java
@Override
public NetworkResponse performRequest(Request<?> request) throws VolleyError {
    long requestStart = SystemClock.elapsedRealtime();
    while (true) {
        HttpResponse httpResponse = null;
        byte[] responseContents = null;
        Map<String, String> responseHeaders = new HashMap<String, String>();
        try {
            // Gather headers.
            Map<String, String> headers = new HashMap<String, String>();
            addCacheHeaders(headers, request.getCacheEntry());
            httpResponse = mHttpStack.performRequest(request, headers);
            StatusLine statusLine = httpResponse.getStatusLine();
            int statusCode = statusLine.getStatusCode();

            responseHeaders = convertHeaders(httpResponse.getAllHeaders());
            // Handle cache validation.
            if (statusCode == HttpStatus.SC_NOT_MODIFIED) {
                return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED,
                        request.getCacheEntry().data, responseHeaders, true);
            }

            // Some responses such as 204s do not have content.  We must check.
            if (httpResponse.getEntity() != null) {
              responseContents = entityToBytes(httpResponse.getEntity());
            } else {
              // Add 0 byte response as a way of honestly representing a
              // no-content request.
              responseContents = new byte[0];
            }

            // if the request is slow, log it.
            long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
            logSlowRequests(requestLifetime, request, responseContents, statusLine);

            if (statusCode < 200 || statusCode > 299) {
                throw new IOException();
            }
            return new NetworkResponse(statusCode, responseContents, responseHeaders, false);
        } catch (SocketTimeoutException e) {
            attemptRetryOnException("socket", request, new TimeoutError());
        } catch (ConnectTimeoutException e) {
            attemptRetryOnException("connection", request, new TimeoutError());
        } catch (MalformedURLException e) {
            throw new RuntimeException("Bad URL " + request.getUrl(), e);
        } catch (IOException e) {
            int statusCode = 0;
            NetworkResponse networkResponse = null;
            if (httpResponse != null) {
                statusCode = httpResponse.getStatusLine().getStatusCode();
            } else {
                throw new NoConnectionError(e);
            }
            VolleyLog.e("Unexpected response code %d for %s", statusCode, request.getUrl());
            if (responseContents != null) {
                networkResponse = new NetworkResponse(statusCode, responseContents,
                        responseHeaders, false);
                if (statusCode == HttpStatus.SC_UNAUTHORIZED ||
                        statusCode == HttpStatus.SC_FORBIDDEN) {
                    attemptRetryOnException("auth",
                            request, new AuthFailureError(networkResponse));
                } else {
                    // TODO: Only throw ServerError for 5xx status codes.
                    throw new ServerError(networkResponse);
                }
            } else {
                throw new NetworkError(networkResponse);
            }
        }
    }
}
```

第 11 行，执行了 addCacheHeaders 方法，因为执行网络请求分两种情况，第一种：全新的请求；第二种：超时的缓存，也就是说缓存过期了，再次请求时需要附带上一次请求所带的`If-None-Match`或者`If-Modified-Since`等信息（如果有的话），如果是全新的请求，则没有这些；

第 12 行，利用 HttpStack 执行其 performRequest 方法，这是网络请求的核心。关于 HttpStack，请参考中的讲解；

第 13、14 行获取执行网络请求后的状态行和状态码

第 16 行，将 12 行中执行的网络请求的 响应的头部保存到一个 map 中，作为响应头

第 18 行，首先做一个特殊的处理，如果状态码是 304，也就是表明当前缓存依然可用，那么直接从缓存中读出数据，然后封装成 NetworkResponse，直接返回

第 24 行，判断响应中的响应体是否为空，因为例如像 204 表示响应执行成功，但没有数据返回，浏览器不用刷新，不用导向新页面，所以我们为 response 的内容添加一个长度为 0 的字节数组；否则读取响应中的响应体，将其转换为字节数组，然后赋给 responseContents

第 39 行，最终返回由 statusCode, responseContents, responseHeaders 组成的 NetworkResponse。

**注意：第 36 行，这里判断了如果响应码是在 200 范围之外的，直接抛出了 IOException，除了在 18 行处理的 304，其它非 2XX 的响应均不能处理，这也是 Volley 的一个缺陷。**



## 关于 HttpStack

这是一个接口，该接口中只有一个方法 performRequest，该方法的作用是：根据所给的参数，执行 HTTP 请求，并返回服务器的响应。

这个接口有两个实现类：HurlStack 和 HttpClientStack；

```java
public interface HttpStack {
    /**
     * Performs an HTTP request with the given parameters.
     *
     * <p>A GET request is sent if request.getPostBody() == null. A POST request is sent otherwise,
     * and the Content-Type header is set to request.getPostBodyContentType().</p>
     *
     * @param request the request to perform
     * @param additionalHeaders additional headers to be sent together with
     *         {@link Request#getHeaders()}
     * @return the HTTP response
     */
    public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
        throws IOException, AuthFailureError;

}
```

我们知道在 Android 中主要提供了两种方式来进行 HTTP 操作，分别是：HttpURLConnection 和 HttpClient。而 Volley 作为一个网络请求库，其底层网络请求的实现正是依靠这两种实现方式，所以 HttpStack 的两个实现类中，HurlStack 是基于 HttpURLConnection ，而 HttpClientStack 则基于 HttpClient 。不过 HttpURLConnection 和 HttpClient 的用法还是稍微有些复杂的，如果不进行适当封装的话，很容易就会写出不少重复代码。那么接下来看一下 HurlStack 和 HttpClientStack 是如何对 HttpURLConnection 和 HttpClient 进行封装的，这里选 HurlStack 类，主要看其如何实现 HttpStack 中的 performRequest 方法：

```java
@Override
public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
        throws IOException, AuthFailureError {
    String url = request.getUrl();
    HashMap<String, String> map = new HashMap<String, String>();
    map.putAll(request.getHeaders());
    map.putAll(additionalHeaders);
    if (mUrlRewriter != null) {
        String rewritten = mUrlRewriter.rewriteUrl(url);
        if (rewritten == null) {
            throw new IOException("URL blocked by rewriter: " + url);
        }
        url = rewritten;
    }
    URL parsedUrl = new URL(url);
    HttpURLConnection connection = openConnection(parsedUrl, request);
    for (String headerName : map.keySet()) {
        connection.addRequestProperty(headerName, map.get(headerName));
    }
    setConnectionParametersForRequest(connection, request);
    // Initialize HttpResponse with data from the HttpURLConnection.
    ProtocolVersion protocolVersion = new ProtocolVersion("HTTP", 1, 1);
    int responseCode = connection.getResponseCode();
    if (responseCode == -1) {
        // -1 is returned by getResponseCode() if the response code could not be retrieved.
        // Signal to the caller that something was wrong with the connection.
        throw new IOException("Could not retrieve response code from HttpUrlConnection.");
    }
    StatusLine responseStatus = new BasicStatusLine(protocolVersion,
            connection.getResponseCode(), connection.getResponseMessage());
    BasicHttpResponse response = new BasicHttpResponse(responseStatus);
    response.setEntity(entityFromConnection(connection));
    for (Entry<String, List<String>> header : connection.getHeaderFields().entrySet()) {
        if (header.getKey() != null) {
            Header h = new BasicHeader(header.getKey(), header.getValue().get(0));
            response.addHeader(h);
        }
    }
    return response;
}
```

第 4 行，拿到请求中的 url 字符串

第 5 行，创建一个 Map，并将 request 中的 headers 都添加到该 map 中，（我们在构造 request 的时候，可以重写其中的 getHeaders 方法，在内部创建一个 Map，然后把 HTTP 请求中需要额外定制的 header 添加到该 map 中），同时把 performRequest 方法中的第二个参数 additionalHeaders 中的所有键值对也都添加到 map 中。

第 16 行，创建了 HttpURLConnection，这里设置了连接超时时间，读取数据的超时时间，如果我们使用的是 HTTPS 协议，那么还要对该连接设置 SSL 相关的属性

```java
private HttpURLConnection openConnection(URL url, Request<?> request) throws IOException {
    HttpURLConnection connection = createConnection(url);

    int timeoutMs = request.getTimeoutMs();
    connection.setConnectTimeout(timeoutMs);
    connection.setReadTimeout(timeoutMs);
    connection.setUseCaches(false);
    connection.setDoInput(true);

    // use caller-provided custom SslSocketFactory, if any, for HTTPS
    if ("https".equals(url.getProtocol()) && mSslSocketFactory != null) {
        ((HttpsURLConnection)connection).setSSLSocketFactory(mSslSocketFactory);
    }

    return connection;
}
```

第 17 行，将我们之前第 5 行的 map 中所有的 HTTP 头部设置添加到连接中。

第 20 行，设置 HTTP 的请求方法。如果我们在创建 request 时没有设置请求方法，那么需要根据该 request 中是否包含 body，如果包含，则设置为 post，如果不包含，则设置为 get。但是在创建 request 时不设置请求方法，是极为不负责的形式，我们在编码时不要这么做。

```java
@SuppressWarnings("deprecation")
/* package */ static void setConnectionParametersForRequest(HttpURLConnection connection,
		Request<?> request) throws IOException, AuthFailureError {
	switch (request.getMethod()) {
		case Method.DEPRECATED_GET_OR_POST:
			// This is the deprecated way that needs to be handled for backwards compatibility.
			// If the request's post body is null, then the assumption is that the request is
			// GET.  Otherwise, it is assumed that the request is a POST.
			byte[] postBody = request.getPostBody();
			if (postBody != null) {
				// Prepare output. There is no need to set Content-Length explicitly,
				// since this is handled by HttpURLConnection using the size of the prepared
				// output stream.
				connection.setDoOutput(true);
				connection.setRequestMethod("POST");
				connection.addRequestProperty(HEADER_CONTENT_TYPE,
						request.getPostBodyContentType());
				DataOutputStream out = new DataOutputStream(connection.getOutputStream());
				out.write(postBody);
				out.close();
			}
			break;
		case Method.GET:
			// Not necessary to set the request method because connection defaults to GET but
			// being explicit here.
			connection.setRequestMethod("GET");
			break;
		case Method.DELETE:
			connection.setRequestMethod("DELETE");
			break;
		case Method.POST:
			connection.setRequestMethod("POST");
			addBodyIfExists(connection, request);
			break;
		case Method.PUT:
			connection.setRequestMethod("PUT");
			addBodyIfExists(connection, request);
			break;
		default:
			throw new IllegalStateException("Unknown method type.");
	}
}
```

这里看一下其中的 post 方法，除了设置请求方法为 post 之外，还要把请求体添加上去，这就用到了 addBodyIfExists 方法

```java
private static void addBodyIfExists(HttpURLConnection connection, Request<?> request)
		throws IOException, AuthFailureError {
	byte[] body = request.getBody();
	if (body != null) {
		connection.setDoOutput(true);
		connection.addRequestProperty(HEADER_CONTENT_TYPE, request.getBodyContentType());
		DataOutputStream out = new DataOutputStream(connection.getOutputStream());
		out.write(body);
		out.close();
	}
}
```

这里先获取 request 中的 body（字节数组），如果 body 不为空，那么为连接添加"Content-Type"的头部，其值默认为“application/x-www-form-urlencoded”，默认编码方式为"UTF-8"。拼接起来一项请求头是："Content-Type:application/x-www-form-urlencoded; charset=UTF-8"。

（如果值为：x-www-form-urlencoded，那么其请求体中的数据采用的是键值对，形如：key1=val1&key2=val2），关于 HTTP 相关知识，请参考[HTTP 包结构]()



额，刚刚扯得有点远了，再回到 performRequest 方法中，前面是阐述了一个 HTTP 的**request**是如何构造的，那么接下来就该执行 HTTP 请求了

第 23 行，这是一个阻塞的方法，一直等待服务器的响应。当然我们在 request 中已经设置了等待时间。

第 29 行，拿到 HTTP 的状态行和响应体，最终包装成 BasicHttpResponse；

第 33 行，拿到响应头部，应该是一个以 key-value 形式的 map，一次解析出响应头，添加到 response 中，最终返回。



# NetworkDispatcher 线程

## 构造方法

首先看一下 NetworkDispatcher 的构造方法，传入了在 RequestQueue 中创建的 mNetworkQueue, mNetwork, mCache, mDelivery，其中 mNetwork 是 RequestQueue 的构造方法中传入的第二个参数 Network；mCache 是在 RequestQueue 的构造方法中传入的第一个参数 Cache，mDelivery 用来将子线程中得到的响应发送要主线程。

```java
public NetworkDispatcher(BlockingQueue<Request> queue,
        Network network, Cache cache,
        ResponseDelivery delivery) {
    mQueue = queue;
    mNetwork = network;
    mCache = cache;
    mDelivery = delivery;
}
```



## run 方法

接下来看一下 NetworkDispatcher 线程的 run 方法：

```java
@Override
public void run() {
	Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
	Request request;
	while (true) {
		try {
			// Take a request from the queue.
			request = mQueue.take();
		} catch (InterruptedException e) {
			// We may have been interrupted because it was time to quit.
			if (mQuit) {
				return;
			}
			continue;
		}

		try {
			request.addMarker("network-queue-take");

			// If the request was cancelled already, do not perform the
			// network request.
			if (request.isCanceled()) {
				request.finish("network-discard-cancelled");
				continue;
			}

			// Tag the request (if API >= 14)
			if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
				TrafficStats.setThreadStatsTag(request.getTrafficStatsTag());
			}

			// Perform the network request.
			NetworkResponse networkResponse = mNetwork.performRequest(request);
			request.addMarker("network-http-complete");

			// If the server returned 304 AND we delivered a response already,
			// we're done -- don't deliver a second identical response.
			if (networkResponse.notModified && request.hasHadResponseDelivered()) {
				request.finish("not-modified");
				continue;
			}

			// Parse the response here on the worker thread.
			Response<?> response = request.parseNetworkResponse(networkResponse);
			request.addMarker("network-parse-complete");

			// Write to cache if applicable.
			// TODO: Only update cache metadata instead of entire record for 304s.
			if (request.shouldCache() && response.cacheEntry != null) {
				mCache.put(request.getCacheKey(), response.cacheEntry);
				request.addMarker("network-cache-written");
			}

			// Post the response back.
			request.markDelivered();
			mDelivery.postResponse(request, response);
		} catch (VolleyError volleyError) {
			parseAndDeliverNetworkError(request, volleyError);
		} catch (Exception e) {
			VolleyLog.e(e, "Unhandled exception %s", e.toString());
			mDelivery.postError(request, new VolleyError(e));
		}
	}
}
```

第 3 行，设置当前线程的优先级为：Process.THREAD_PRIORITY_BACKGROUND，后台线程，相对来说优先级还是比较低的。

第 5 行，可以看到该线程是个死循环。

第 8 行，从 BlockingQueue 中取出一个 request，如果没有的话，就会被阻塞，也不会占用系统资源，赞。

第 22 行，如果该 request 还未被执行，但是在其它线程取消了，那么调用 request 的 finish 方法，其主要功能是将该 request 从 RequestQueue 的 mCurrentRequests 中移除掉当前 request。

第 33 行，如果能走到这一步，就表明真的要执行网络请求了，还记的 NetworkDispatcher 的构造方法中传入的 Network 对象吗，调用了它的 performRequest 方法，但是真正的执行者是 Network 的实现类 BasicNetwork。具体流程已经在上面进行了分析。

第 38 行，如果响应是 304 并且该 request 已经分发了，那么调用 request 的 finish 方法，该方法是将 RequestQueue 中 mWaitingRequests 队列中取出相同的 request，然后添加到 mCacheQueue 中。

第 44 行，调用 request 的 parseNetworkResponse 方法对 33 得到的 networkResponse 进行解析，至于具体怎么解析，还记的我们自定义请求时需要重写的方法，就是这个方法，所以具体如何解析 networkResponse 完全由你自己决定

第 49 行，执行缓存相关的操作，具体实现请参考[Volley 源码解析-缓存](https://zachaxy.github.io/2017/05/23/Volley%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90-%E7%BC%93%E5%AD%98/)

第 55 行，将该 request 标记为已分发

第 56 行，将该 request 进行分发，调用了 mDelivery 的方法，这个 mDelivery 是在 NetworkDispatcher 的构造方法中传入进来的，具体创建是在 RequestQueue 的构造方法中，期创建方式是：`new ExecutorDelivery(new Handler(Looper.getMainLooper()))`看到这相信各位也能猜个大概了，我们前面也说了这个 ResponseDelivery 的作用是分发 response，我们在应用中请求网络数据，首先在子线程中执行网络请求，得到了响应之后，肯定要将结果返回到主线程，如何发呢？答案就是 Handler，而 ResponseDelivery 正是包装了一个 Handler；具体如何实现请参考[Volley 源码解析-线程切换]()



# 总结

至此，一个完整的网络请求的过程就分析完了，这里在总结一下网络请求的过程：

需要执行网络的 request 无非是两种情况：

- 第一种：一个新的请求，之前没有执行过，自然也没有对应的缓存，所以必须要走 NetworkDispatcher 这个线程；
- 第二种：之前执行过该请求，所以先从缓存中取，但是缓存不可用（缓存过期了或者不存在了），所以也必须要走 NetworkDispatcher 这个线程；

走网络路线，有一个接口 Network，该接口内部封装了执行网络操作的所有方法，其唯一实现类是 BasicNetwork，而 BasicNetwork 中有包含了 HttpStack，HttpStack 接口的实现类才是真正实现了真正执行网络请求的功能（HttpURLConnection 和 HttpClient）
