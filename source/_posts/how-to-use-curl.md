---
title: curl 使用指南
date: 2019-09-07 23:22:40
tags: 工具
---
curl是一种命令行工具，作用是发出网络请求，然后得到和提取数据，显示在"标准输出"（stdout）上面。它支持多种协议，下面将介绍 curl 的简单使用

<!-- more -->

# 上手

    curl https://github.com/

curl 后面直接跟一个url，那么将会返回该网站的response。

# 查看具体的相应
- `-i` 参数可以显示http response的头信息，连同网页代码一起。
- `-I` 参数则是只显示http response的头信息。
- `-v` 参数可以显示一次http通信的整个过程，包括端口连接和http request头信息。

如果 `-v` 参数还不够详细，可以使用 `--trace` 将相应保存在文件中查看

    curl --trace output.txt https://github.com/
    curl --trace-ascii output.txt https://github.com/


# 添加参数
上述只是查看一个响应，接下来我们通过添加参数的方式，去构造一个请求，并获取相应。

curl 默认的 HTTP 方法是 GET，使用`-X`参数可以支持其他方法。

## GET 请求
如果要对 GET 请求添加参数只要在 url 后面拼接参数就可以了，这里要注意的是如果参数需要编码，必须显示的指明
    curl -G --data-urlencode "key1=value1" --data-urlencode "key2=value2" https://github.com/
其中 `-G` 表明将接下来的参数拼接到url 后面， 而 `--data-urlencode ` 表明需要对后面的 k-v 进行编码，每添加一个k-v 对，都需要显示指明编码

## POST 请求
使用 `-X` 后面指定方法，

    curl -X POST --data "key1=value1" --data-urlencode "key2=value2" https://github.com/

## 文件上传

假定文件上传的表单是下面这样：
```html
<form method="POST" enctype='multipart/form-data' action="xxx">
　　　　<input type=file name=upload>
　　　　<input type=submit name=press value="OK">
</form>
```

那么文件上传的命令为：

    curl --form upload=@localfilename --form press=OK  https://github.com/


## auth 认证
有些网域需要HTTP认证，这时curl需要用到`-u`参数。

    curl -i -u your_username https://api.github.com/users/defunkt
    Enter host password for user your_username:


## 添加头信息
有时需要在http request之中，自行增加一个头信息。`-H` 或者 `--header`参数就可以起到这个作用。

    curl -H "Content-Type:application/json" https://github.com/
    

## 添加 User Agent字段

这个字段是用来表示客户端的设备信息。服务器有时会根据这个字段，针对不同设备，返回不同格式的网页，比如手机版和桌面版。

iPhone4的User Agent是

　　Mozilla/5.0 (iPhone; U; CPU iPhone OS 4_0 like Mac OS X; en-us) AppleWebKit/532.9 (KHTML, like Gecko) Version/4.0.5 Mobile/8A293 Safari/6531.22.7

curl可以这样模拟：

    curl --user-agent "your agent" https://github.com/

# more
    curl -help

一切尽在说明中～