---
title: "如何保护你的客户端应用远离被破解"
published: true
---

这篇博客主要记录了我在反破解方面的一些见闻以及思考.

+ 最基础的一点是服务端要能够封禁用户账户.

+ 核心业务能放在服务端实现的尽量放在服务端实现.

For example:

![2019-05-17-qq-chat-log.png]({{ site.baseurl }}{% link /img/2019/05-17-qq-chat-log.png %})

+ Android 客户端后台检查更新时上传 applicationId, versionCode, versionName 等参数,
并在检查更新失败时抑制用户可见的提示. 这样做可以使 applicationId 被修改的应用能通过破解者的简单调试,
但服务端能够对被破解的应用推送强制更新.
例如, 原应用的 applicationId = com.domain.sample, versionCode = 1, 被破解后 applicationId
= com.domain.sampl, versionCode = 23, 则可以在服务端配置 com.domain.sampl 的强制更新,
将破解版的用户其转化为正版应用的用户.

For example: 蓝色内容为正版用户检查更新时创建的 redis 缓存, 黑色的则是破解版用户.

![2019-05-21-redis-log.png]({{ site.baseurl }}{% link /img/2019/05-21-redis-log.png %})

