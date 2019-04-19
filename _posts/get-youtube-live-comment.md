---
title: 抓取 YouTube 直播评论
date: 2019-04-19 22:43:00
tags: 
    - YouTube Live
    - Comment
---

## 起因

最近成为了一名 DD ，但奈何一天没有 26 个小时，所以打算写个能自动录制直播和弹幕的工具 ~~监控器~~。

开始动手后，发现 B 站的视频流和弹幕是真的好拿，网上也是一堆解析文章，但 YouTube 就没这么方便了，视频流在 [youtube-dl](https://github.com/ytdl-org/youtube-dl) 里找到了方法，评论网上只能找到使用 YouTube API 的方法，所以最后还是请出 Chrome 的 F12 大法。

## 分析

一般的视频网站的弹幕用的都是 WebSocket (HTML5 播放器)，但 YouTube 真的是有钱任性，直接用 ajax 按时请求服务器。

### 入口

首先要拿到直播的 `videoId` ，一般就是 `https://www.youtube.com/watch?v=` 后面的 11 位字符串。然后抓取链接 `https://www.youtube.com/live_chat?is_popout=1&v=videoId` 的内容（必须带 `UserAgent` 要不然会跳转到不支持浏览器页面）。在抓下来的 HTML 内容中有一段 `window["ytInitialData"] = {...}` ，把花括号包括内容用正则匹提取出来按 JSON 格式进行解析。

```JSON
{
    ...
    "contents": {
        "liveChatRenderer": {
            "continuations": [{
                "invalidationContinuationData": {
                    "invalidationId": {...},
                    "timeoutMs": 10000,
                    "continuation": "continuation",
                    "clickTrackingParams": "clickTrackingParams"
                }
            }]
        }
    }
    ...
}
```

其中 `continuation` 就是下一个请求的 Token， 而 `timeoutMs` 是下个请求需间隔的毫秒数。

> 经测试 videoId 中带下划线的直播 `invalidationContinuationData` 要替换成 `timedContinuationData` 

拿到 `continuation` 后就可以开始循环请求 ~~无限 For 制~~ 了。

### 获取弹幕

带上 `UserAgent` 和入口请求下发的 Cookie 请求 `https://www.youtube.com/live_chat/get_live_chat?continuation=continuation&pbj=1` 可以获取一段 JSON。

```JSON
{
    ...
    "response": {
        "responseContext": {...},
        "continuationContents": {
            "liveChatContinuation": {
                "continuations": [{
                    "invalidationContinuationData": {
                        "invalidationId": {},
                        "timeoutMs": 10000,
                        "continuation": "...",
                        "clickTrackingParams": "..."
                    }
                }],
                "actions": [{
                    "addChatItemAction": {
                        "item": {
                            "liveChatTextMessageRenderer": {
                                "message": {
                                    "simpleText": ""
                                },
                                "authorName": {
                                    "simpleText": ""
                                },
                                ...
                                "timestampUsec": "1555680252649047"
                            }
                        }
                    }
                }]
            }
        }
    }
    ...
}
```

在 `actions` 里每一个 `addChatItemAction` 都是一条评论，其中 `message` 是评论内容，`authorName` 是发送用户。之后循环在 Sleep `timeoutMs` 后用新 `continuation` 发下一次请求。

> 这里 `invalidationContinuationData` 特性和上面相同

