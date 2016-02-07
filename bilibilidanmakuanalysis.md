---
title: 哔哩哔哩弹幕分析
date: 2015-08-03 15:27:54
tags:
  - 哔哩哔哩
  - 弹幕
  - 撒驱魔粉
---

最近打算写一个工具用来将 B 站弹幕转换为 ASS 字幕，所以在这里整理下一下 B 站弹幕的相关信息。

#获取弹幕

B 站的弹幕都是以 XML 格式储存在服务器中，可以通过请求 `http://comment.bilibili.com/cid.xml` 获取。

至于 `cid` 可以通过以下方法获取。

1、B 站 API 。
----------------

B 站提供了一系列 [API](http://docs.bilibili.cn/wiki) 方便开发者开发第三方客户端，当然可以通过这个方法获取相关信息。至于如何使用，因为没有 `AppKey` 所以就不实验了。

[备用地址](http://www.fuckbilibili.com/biliapi.html)

2、播放页面。
-----------------

通过查看视频播放页面的源代码可以找到这样一段代码：

```html
<div class="scontent" id="bofqi">
  <div id='player_placeholder' class='player'></div>
     <script type='text/javascript'>
        EmbedPlayer('player', "http://static.hdslb.com/play.swf", "cid=123&aid=123");
     </script>
</div>
```

其中 `cid=123&aid=123` 就是该视频的 `cid` 和 `aid` 。

3、HTML5 页面。

B 站提供了 HTML5 播放器，所以可以通过访问这个链接获取封面、弹幕和视频文件地址信息。

通过请求 `http://www.bilibili.com/m/html5?aid=123` 将会返回：

```json
{
  "img":"http://hdslb.com/video/123.jpg",
  "cid":"http://comment.bilibili.com/123.xml",
  "src":"http://acgvideo.com/123.mp4"
}
```

其中 `cid` 节点后面的地址就是弹幕文件的地址。

#分析弹幕

在获取的 XML 文件中可以看到许多类似的代码：

```html
<d p="1619.4599609375,1,25,16777215,1434797363,0,f9b8d3d4,934194846">233333</d>
<d p="1124.3100585938,1,25,16777215,1434797366,0,3ffff45a,934194882">yoooooo</d>
<d p="1136.0999755859,1,25,16777215,1434797377,0,3ffff45a,934195149">yooooo</d>
<d p="1642.5100097656,1,25,16777215,1434797386,0,f9b8d3d4,934195346">233333333</d>
```

每个 `<d>` 标签包含一条弹幕，其中属性 `p` 包含弹幕的相关信息。

>第一项是弹幕所在时间，单位为秒。

>第二项是弹幕类型，其中 1~3 为滚动弹幕，4 为底端弹幕，5 为顶端弹幕，6 为逆向弹幕，7 为精确弹幕，8 为高级弹幕。

>第三项是弹幕字体大小，其中 25 为中，18 为小。

>第四项是弹幕颜色，格式是十进制的 RGB 颜色。

>第五项是弹幕的发送时间，使用的是 Unix 时间戳。

>第六项是弹幕池，其中 0 为普通弹幕，1 为字幕弹幕，2 为特殊弹幕。

>第七项是发送者的 ID 的 CRC32b 加密，可以用来屏蔽发送者。[详细参考](http://www.fuckbilibili.com/bilidanmaku.html)

>第八项是弹幕在数据库的 ID ，可能是用于历史弹幕。


<br />
弹幕的获取在这里已经完成，剩下的工作就是将这些信息转换为 Ass 字幕文件格式的了。
<br />
<br />
说的好像真的这么简单似的。。。。。
<br />
<br />
<br />
<br />
以及，[撒驱魔粉~](http://www.bilibili.com/video/av2272040/)！