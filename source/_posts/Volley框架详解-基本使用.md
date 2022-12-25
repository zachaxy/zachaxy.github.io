---
title: Volley框架详解-基本使用
date: 2017-05-20 21:52:01
tags: 框架设计之美
---

> 最近在往博客上整理一些之前看过的框架，翻到了已经推出很久的网络请求框架—— Volley。这是在 13 年由谷歌推出的一款轻量级的 Android 异步网络请求框架和图片加载框架。虽然现在市面上已经出现了比 Volley 更好的网络请求框架——Okhttp，和图片请求框架——Glide，但是个人认为 Volley 的源码比其它框架的源码更加优美，所以 Volley 的源码非常值得初学者拿来分析练手。



# Volley 简介

Volley 是 Google 推出的 Android 异步网络请求框架和图片加载框架。在 Google I/O 2013 大会上发布。

Volley 的特点：特别适合**数据量小，通信频繁**的网络操作。



# Volley 的主要特点

1. 扩展性强。Volley 中大多是**基于接口**的设计，可配置性强。
2. 一定程度符合 Http 规范，包括返回 ResponseCode(2xx、3xx、4xx、5xx）的处理，请求头的处理，缓存机制的支持等。并支持重试及优先级定义。
3. 默认 Android2.3 及以上基于 HttpURLConnection，2.3 以下基于 HttpClient 实现。所以兼容性强。



# Volley 的基本用法

在使用 Volley 之前，不要忘记在你的 manifest 文件中添加网络权限。



##  （一）创建请求队列

首先需要创建一个 RequestQueue 对象，用来保存所有的网路请求 request，其获取方法：

```java
RequestQueue mQueue = Volley.newRequestQueue(context);  
```

这里要注意的是：RequestQueue 内部的设计适合高并发，因此我们不必为每一次 HTTP 请求都创建一个 RequestQueue 对象，这是非常浪费资源的，基本上一个 RequestQueue 对象就足够了。所以我们将 RequestQueue 的获取放在自定义的 Application 中：

```java
public class MyApp extends Application {

    public static RequestQueue queues;

    @Override
    public void onCreate() {
        super.onCreate();
        queues = Volley.newRequestQueue(getApplicationContext());
    }

    public static RequestQueue getHttpQueues() {
        return queues;
    }
}
```

## （二）创建请求

Volley 自带的请求有：

1. StringRequest
2. JsonRequest
3. ImageRequest

一般我们请求的数据都是当做字符串来返回的，如果返回的结果使我们自定义的简单的结果，那么直接解析返回的字符串即可，但如果返回的结果是 Json 或者 Xml，那么就需要我们手动将返回的字符串转换为对应的 Json 或 Xml，好在 Volley 已经为我们实现了 Json，后面我们会学习如何自定义 Xml 的请求。



常见的请求又分为两种，一种是 GET 请求，一种是 POST 请求，这里以 StringRequest 为例，分别创建 GET 请求和 POST 请求：

### GET 请求

```java
String url = "http://op.juhe.cn/onebox/phone/query?tel=18220197835&key=xxxxxxxxxxxxxxxxxxx";
//使用 get 方法;传入 URL;请求成功的监听回调;请求失败的监听回调;
StringRequest request = new StringRequest(Request.Method.GET, url, new Response.Listener<String>() {
    @Override
    public void onResponse(String s) {
        //System.out.println(s);
        Log.e("volley_GetByString", "onResponse: " + s);
    }
}, new Response.ErrorListener() {
    @Override
    public void onErrorResponse(VolleyError volleyError) {

    }
});

request.setTag("abc");
MyApp.getHttpQueues().add(request);
```

创建 GET 请求，只需要在创建的 StringRequest 的构造方法的第一个参数中传入常量 Request.Method.GET 即可；第二个参数是所请求的 URL，第三个参数是请求成功的回调，这是一个接口，需要我们实现 onResponse 方法，该方法中传入的参数就是服务器给我们回调的结果；第四个参数是请求失败的回调，也是一个接口，需要我们实现 onErrorResponse，我们可以根据自己的业务需求，分别在请求成功或者失败的回调中处理不同的逻辑；



至此，一个请求就已经创建好了，如何发送请求呢？很简单，只需要将该请求添加到我们在上一步中创建的 RequestQueue 中即可。这里我们为 request 打上了一个标签，这个标签有什么用？在 Volley 与 Activity 生命周期的联动中会用到，别急。

### POST 请求

```java
        String url = "http://op.juhe.cn/onebox/phone/query?";
        StringRequest request = new StringRequest(Request.Method.POST, url, new Response.Listener<String>() {
            @Override
            public void onResponse(String s) {
                PhoneMsg msg;
                Gson gson = new Gson();
                msg = gson.fromJson(s, PhoneMsg.class);
                Log.e("###", "volley_PostByString: " + msg.reason + "<>" + msg.result.city + "<>" + msg.error_code);
            }
        }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError volleyError) {

            }
        }) {
            //StringRequest 的父类中的方法,这里我们是创建了一个匿名 StringRequest,重写了其 getParams 方法;
            //当使用 post 方法时,会自动回调该方法,得到一个 map,并将其中的 key-value 以 key1=value1&key2=value2& 的形式拼接起来
            @Override
            protected Map<String, String> getParams() throws AuthFailureError {
                Map<String, String> hashMap = new HashMap<>();
                hashMap.put("tel", "18220197835");
                hashMap.put("key", "xxxxxxxxxxxxxxxxxx");
                return hashMap;
            }
        };

        request.setTag("abc");
        MyApp.getHttpQueues().add(request);
```

POST 请求和 GET 请求类似，只不过在创建的 StringRequest 第一个参数改为常量 Request.Method.POST 即可，第二个参数 URL 和 get 不同了，因为我们这里是用 POST 方法发送请求，那么请求的 key-value 都不会出现在 url 中，而是重写了 StringRequest 的 getParams 方法，创建了一个 map，将请求的 key-value 保存到 map 中，然后将该 map 返回即可，在 post 请求发送时会自动回调 getParams 方法,并得到的 map 中的 key-value 以 key1=value1&key2=value2& 的形式拼接起来，添加到请求体中。第三个参数和第四个参数与 GET 请求相同。



# Volley 与 Activity 生命周期的联动

在创建了请求之后，将请求添加到 RequestQueue 中，

考虑这样一种场景，如果我们在当前的 Activity 中创建了大量的 request，但是这时突然用户有点击了 back 键或者跳转到了另一个 Activity，那么还未执行的 request 此时就没有必要去执行了，如何取消还未执行的请求呢？注意我们在每个 request 中都绑定了一个 tag，`request.setTag("abc");`，如果想取消这些 request，只需要在 Activity 的 onStop 回调中调用 cancelAll 方法，取消特定的 tag，那么之前发出的请求将不会被执行。

```java
@Override
protected void onStop() {
    super.onStop();
    MyApp.getHttpQueues().cancelAll("abc");
}
```

# 自定义请求

前面也介绍了 Volley 的一个特点就是高扩展性，官方也为我们提供了 StringRequest，JsonRequest 等现成的 request，但是如果我们的业务用的是我们一种公司自定义的形式，那么我们就有必要自定义请求来解析返回的结果。

首先要明确的是：最原始的请求返回的结果都是字节流，我们正是通过对字节流的转换，才得到了我们想要的请求，这就是自定义 request 的本质。

JsonRequest 的数据解析是利用 Android 本身自带的 JSONObject 和 JSONArray 来实现的，配合使用 JSONObject 和 JSONArray 就可以解析出任意格式的 JSON 数据。但是使用 JSONObject 还是太麻烦了，还有很多方法可以让 JSON 数据解析变得更加简单，比如说 GSON。那么接下来我们自定义一个 GsonRequest。

```java
public class GsonRequest<T> extends Request<T> {
    private Response.Listener<T> mListener;
    private Gson mGson;
    private Class<T> mClass;

    public GsonRequest(int method, String url, Class<T> clazz, Response.Listener<T> listener,
                       Response.ErrorListener elistener) {
        super(method, url, elistener);
        mGson = new Gson();
        mClass = clazz;
        mListener = listener;
    }

    public GsonRequest(String url, Class<T> clazz, Response.Listener<T> listener,
                       Response.ErrorListener errorListener) {
        this(Method.GET, url, clazz, listener, errorListener);
    }


    /*
    这是响应最原始的面貌,得到的就是一个 NetWorkResponse,至于应用层想得到什么类型的响应,需要在这里进行转换;
    这个过程和 StringRequest 的构造很像,具体可以参考 StringRequest;
     */
    @Override
    protected Response parseNetworkResponse(NetworkResponse response) {
        try {
            //这一句是固定格式,先将 response 的 data 域拿出来,其是字节,要转换成字符串,同时要获取该字节的编码;
            String jsonString = new String(response.data, HttpHeaderParser.parseCharset(response.headers));
            return Response.success(mGson.fromJson(jsonString, mClass), HttpHeaderParser.parseCacheHeaders(response));
        } catch (UnsupportedEncodingException e) {
            return Response.error(new ParseError(e));
        }
    }

   
    @Override
    protected void deliverResponse(T response) {
        mListener.onResponse(response);
    }
}
```

自定义 request 需要继承 Request 类，而 Request 是一个抽象类，需要我们实现其抽象方法，包含两个：parseNetworkResponse 和 deliverResponse。

第一个要实现的方法 parseNetworkResponse，该方法是得到服务器响应时回调的结果，其参数是执行网络请求得到的响应 NetworkResponse，我们可以通过 NetworkResponse.data 拿到包含在响应中的字节数组，然后通过 HttpHeaderParser.parseCacheHeaders(response)获取响应中的编码格式，将该字节数组转换为对应的字符串。如果字符串解析成功，接下来我们就可以调用 Response 中的 success 方法，将结果传入进去，如果解码失败，那么调用 Response 的 error 方法。



第二个要实现的方法是 deliverResponse，这个方法也必须实现,该方法的参数是从上一步 parseNetworkResponse 中 Response.success()第一个参数得到的响应,我们要把这个响应分发下去;注意这一步说明我们已经得到成功的响应了,而之前 errorListener 的响应则是在网络出错或者服务器出错的情况下分发的 是由 Volley 顶层实现的。



我们看一下Volley自带的StringRequest是如何实现的：

```java
public class StringRequest extends Request<String> {
    private final Listener<String> mListener;

    /**
     * Creates a new request with the given method.
     *
     * @param method the request {@link Method} to use
     * @param url URL to fetch the string at
     * @param listener Listener to receive the String response
     * @param errorListener Error listener, or null to ignore errors
     */
    public StringRequest(int method, String url, Listener<String> listener,
            ErrorListener errorListener) {
        super(method, url, errorListener);
        mListener = listener;
    }

    /**
     * Creates a new GET request.
     *
     * @param url URL to fetch the string at
     * @param listener Listener to receive the String response
     * @param errorListener Error listener, or null to ignore errors
     */
    public StringRequest(String url, Listener<String> listener, ErrorListener errorListener) {
        this(Method.GET, url, listener, errorListener);
    }

    @Override
    protected void deliverResponse(String response) {
        mListener.onResponse(response);
    }

    @Override
    protected Response<String> parseNetworkResponse(NetworkResponse response) {
        String parsed;
        try {
            parsed = new String(response.data, HttpHeaderParser.parseCharset(response.headers));
        } catch (UnsupportedEncodingException e) {
            parsed = new String(response.data);
        }
        return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response));
    }
}
```

是不是相同的套路，同样是拿到了服务器的字节数组之后，直接转换为了字符串，然后就返回了，这个我们自定义的GsonRequest和StringRequest的套路是一致的。

至此，关于Volley的基本使用就讲完了，具体的使用请参考[这里](https://github.com/zachaxy/android-project-demo/blob/master/TestVolley/app/src/main/java/com/zx/testvolley/MainActivity.java)，下一篇，会进入Volley的源码，详细解剖其内部的基本实现。