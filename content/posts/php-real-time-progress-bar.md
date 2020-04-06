---
title: "PHP 实时进度条"
date: 2017-12-14T19:30:00+08:00
tags: ["PHP", "Ajax", "进度条"]
---

之前遇到一个场景，需要在服务端运行一个长时间的任务，这种情况下肯定不能让用户一直看转圈圈，
之前的人的解决方案是开启一个 `iframe` 然后当后台每处理完一条数据打印 `处理完第 X 条数据`。
其实这个解决方案可以说是最快捷直观的 ~~对写代码的来说~~ ，但总是想处理的更好一些，
比如改成一个进度条。

<!--more-->

去网上了解了一圈，发现基本上主流解决方案都是把进度写入到一个文件，然后前端定时请求获取进度。
但这样会造成大量的 `AJAX` 请求，这让我强迫症很难受。不过搜索的的时候看到有人说 `AJAX`
有一个 `readyState` ，当状态为 `LOADING` 的时候可以拿到已获取的部分数据。
根据这个特性，可以让服务器每处理完一条数据直接打印一个 `JSON` 数据表示当前处理的进度。

### 实现

在实现的时候遇到了个坑点：

> 服务端不一定会实时输出

所以针对这个需要做一些 workaround

- 使用 `ob_flush` 和 `flush`
- 添加头 `X-Accel-Buffering: no`
- `AJAX` 接受时对接受字符串进行 `JSON` 提取拼接，因为不一定是一次完整的 `echo` 被发送过来

当然最重要的 `set_time_limit(0)` 不要忘了。

经过我的测试，在 `nginx` 和 `php-fpm` 环境下基本没问题了。~~有问题别找我~~

### Talk is cheap. Show me the code.

> 咕咕咕

参考

- [XMLHttpRequest](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest#%E5%B1%9E%E6%80%A7)
- [flush](http://php.net/manual/zh/function.ob-flush.php)
- [X-Accel](https://www.nginx.com/resources/wiki/start/topics/examples/x-accel/)
