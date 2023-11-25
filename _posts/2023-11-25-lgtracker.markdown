---
layout: post
title: Luogu Tracker 开发日志（回忆）
description: 利用洛谷的低能 API 从零开始构建一个洛谷版的 cftracker
date: 2023-11-25 11:41:10 +0800
tag: luogu golang javascript vuejs vite
locale: zh_CN
---

本项目仓库位于 <https://github.com/wxh06/lgtracker>，这篇博客回忆了从立项到开发的完整过程。

洛谷的 dinner API 大家有目共睹，很喜欢 5ab 老师说过的：

> 感觉这东西做不出什么花来啊<br />毕竟洛谷的 api 太 dinner 了
>
> 我还以为洛谷的 dinner api 做不出来这种东西

## 可行性

最初是 11 月 4 日，5ab 老师问大家是否有兴趣写 lgtracker。当时我进行了简单地可行性分析。

事实表明，洛谷赛时的提交不算通过（我在北京清华园时参加了 [LGR-148-Div.3](https://www.luogu.com.cn/contest/101050#scoreboard) 并通过了前两题，但并没有出现在我的[个人中心](https://www.luogu.com.cn/user/108135#practice)）。

通过查阅 [luogu-api-docs](https://0f-0b.github.io/luogu-api-docs/) ，可以找到洛谷提供的接口包含：

1. 根据比赛、题目、用户三者任意组合筛选提交记录（API [列出记录](https://0f-0b.github.io/luogu-api-docs/records#列出记录)，用于“[评测记录](https://www.luogu.com.cn/record/list)”页面）
2. 根据用户获得非赛时 AC 的题（API [获取用户](https://0f-0b.github.io/luogu-api-docs/users#获取用户)，用于“个人中心”页面的“练习”选项卡）

我提出的实践逻辑是：“首先从用户页获得所有非赛时 AC 的题（2），然后根据用户参加的比赛获取赛时的通过情况（1），再合并两者”。

但是这种实践的缺陷也很明显：

- 若用户在赛时有大量提交，则需要分多次请求获取（20 条测评记录为一页）。较高的瞬时频率，可能会触发洛谷的频次限制，导致获取效率显著降低。
- 要获取用户的参赛记录只能通过 elo 的接口（API [获取历史等级分](https://0f-0b.github.io/luogu-api-docs/users#获取历史等级分)，用于“个人中心”页面“练习”选项卡的“比赛等级分趋势图”），因此 5ab 老师指出这样无法获取到 unrated 的比赛信息；而符合需求的[列出参加过的比赛](https://0f-0b.github.io/luogu-api-docs/contests#列出参加的比赛)接口不能获取当前登录用户以外他人的数据。

> 5ab：哎洛谷的 api。

11 月 6 日周一，我提出“按比赛一次性全爬完就好了”，直接[从排行榜爬取](https://0f-0b.github.io/luogu-api-docs/contests#获取排行榜)所有比赛的赛时数据，包含每场所有参赛者的各题分数；再额外从用户个人中心获取赛后的通过情况。

通过调研，发现 [cftracker](https://github.com/mbashem/cftracker) 也是预先爬取好所有比赛和题目信息的。

## 实现

11 月 6 日动工，创建仓库并[初始化 Vue](https://github.com/wxh06/lgtracker/commit/43fdea5e68b64a2c3e090b3814a4092cf7f8455b)。

在观察了 Vite 的项目结构后，我决定把比赛数据作为 JSON 放在 `src/` 的源代码中，直接在 Vue 组件中导入用于展示比赛信息；获取到的排行榜按用户合并后，拆分为每个用户一个文件放进 `public/` 作为静态资源，用户键入 UID 再按需 `fetch`。

### 爬取

11 月 7 日与 8 日主要是在 [luogu-cli](https://github.com/wxh06/luogu-cli) 中实现比赛相关的接口；11 月 9 日中午用 Go 完成了数据爬取的实现。

其间遇到的麻烦有：

- `/fe/api/` 路径下的 API 需要 `__client_id` cookie：[填 40 位 `0`](https://github.com/wxh06/luogu-cli/blob/a3113a9755c90544a9db33db2920428ff3c8ff46/pkg/luogu/request.go#L29) 就能正常[获取排行榜](https://0f-0b.github.io/luogu-api-docs/contests#获取排行榜)了，甚至不需要 `_uid`。
- 上述 API 的频率限制：[生成一个随机的 40 位十六进制数](https://github.com/wxh06/luogu-cli/blob/2880147e066b5a2b47a734531fd952b010ebb9d4/pkg/luogu/request.go#L34-L38)就好了，猜测洛谷是将 API 层的频率限制绑定到 `__client_id` 的。
- 如果用户在赛时提交的题目为空集时，低能洛谷 API 在会在该用户的题号到分数映射的位置[返回一个空数组 `[]`](https://github.com/0f-0b/luogu-api-docs/blob/3556d6d815035537d8055631c24988d0f8519809/luogu-api.d.ts#L552) 而非空对象 `{}`，导致 Go 不能成功根据 interface 解析 JSON。最后通过[引入](https://github.com/wxh06/luogu-cli/commit/526d3c9e6f64a300a7e9548c1b70efa84eba44f6) [`json.RawMessage`](https://pkg.go.dev/encoding/json#RawMessage) 延缓这一小部分数据的解析并在使用前[调用 `json.Unmarshal` 时判断 `error` 的返回值](https://github.com/wxh06/lgtracker/blob/70e926cd59e10315288831ec66f02fb2d034efff/fetch.go#L44-L47)解决了这个问题。

### 前端

11 月 9 日晚自习结束前，完成了基本的前端展示。

// 关于 Vite SSG + PrimeVue 的踩坑经历待补。
