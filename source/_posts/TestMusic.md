---
title: TestMusic
date: 2018-01-23 13:02:53
tags: 其它
---

测试音乐播放器。依次使用了 iframe，audio，embed三个标签，推荐指数逐渐降低。

<!--more-->

# 使用 iframe

```html
<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=208891&auto=1&height=66"></iframe>
```

如果网易云音乐中有对应资源，推荐使用iframe播放，缺点就是网易云音乐版权太少。

 


# 使用 audio

<audio src="http://musics.gimhoy.com/mp3/2018/01/50583a8f3e36638aba61c52de18185fc.mp3" controls="controls" autoplay="autoplay" loop="loop">
</audio>

如果网易云音乐没有版权的话，推荐使用audio。
音乐外链获取方式：[gimhoy音乐盘](http://music.gimhoy.com/)


# 使用flash插件

``` 
<embed src="//music.163.com/style/swf/widget.swf?sid=25714332&type=2&auto=1&width=320&height=66" width="340" height="86"  allowNetworking="all"></embed>
```

不推荐，无法实现自动播放，页面中还需要手动点击，影响体验。
