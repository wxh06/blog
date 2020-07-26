---
layout:     post
title:      "中国大陆安装 Puppeteer Chromium 失败的解决方法"
subtitle:   "使用镜像安装 Puppeteer Chromium"
date:       2020-07-26 23:00:00 +0800
categories: javascript
---

## 起因

前段时间贡献 [AliasIO/wappalyzer][wappalyzer-gh]，按照 [Quick start][quick-start] [setup][] 运行 `yarn install` 的时候在 `[1/1] puppeteer` 花费了很长时间，最终由于安装 Chromium 失败而报错：`ERROR: Failed to download Chromium r686378!`

<details><summary>

`ERROR: Failed to download Chromium r686378! Set "PUPPETEER_SKIP_CHROMIUM_DOWNLOAD" env variable to skip download.`

</summary>

```
➜  RemoteWorking git:(master) yarn install
yarn install v1.22.4
[1/4] Resolving packages...
[2/4] Fetching packages...
info There appears to be trouble with your network connection. Retrying...
[3/4] Linking dependencies...
warning "@nuxtjs/eslint-config > eslint-plugin-vue@5.2.3" has incorrect peer dependency "eslint@^5.0.0".
warning "@nuxtjs/eslint-config > eslint-plugin-vue > vue-eslint-parser@5.0.0" has incorrect peer dependency "eslint@^5.0.0".
warning "@nuxtjs/eslint-module > eslint-loader@4.0.2" has unmet peer dependency "webpack@^4.0.0 || ^5.0.0".
[4/4] Building fresh packages...
error /root/RemoteWorking/node_modules/puppeteer: Command failed.
Exit code: 1
Command: node install.js
Arguments: 
Directory: /root/RemoteWorking/node_modules/puppeteer
Output:
ERROR: Failed to download Chromium r686378! Set "PUPPETEER_SKIP_CHROMIUM_DOWNLOAD" env variable to skip download.
{ Error: Client network socket disconnected before secure TLS connection was established
    at TLSSocket.onConnectEnd (_tls_wrap.js:1092:19)
    at Object.onceWrapper (events.js:286:20)
    at TLSSocket.emit (events.js:203:15)
    at endReadableNT (_stream_readable.js:1129:12)
    at process._tickCallback (internal/process/next_tick.js:63:19)
  -- ASYNC --
    at BrowserFetcher.<anonymous> (/root/RemoteWorking/node_modules/puppeteer/lib/helper.js:111:15)
    at Object.<anonymous> (/root/RemoteWorking/node_modules/puppeteer/install.js:64:16)
    at Module._compile (internal/modules/cjs/loader.js:776:30)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:787:10)
    at Module.load (internal/modules/cjs/loader.js:653:32)
    at tryModuleLoad (internal/modules/cjs/loader.js:593:12)
    at Function.Module._load (internal/modules/cjs/loader.js:585:3)
    at Function.Module.runMain (internal/modules/cjs/loader.js:829:12)
    at startup (internal/bootstrap/node.js:283:19)
    at bootstrapNodeJSCore (internal/bootstrap/node.js:622:3)
  code: 'ECONNRESET',
  path: null,
  host: 'storage.googleapis.com',
  port: 443,
  localAddress: undefined }
info Visit https://yarnpkg.com/en/docs/cli/install for documentation about this command.
➜  RemoteWorking git:(master) 
```

</details>

## 经过

查阅其[源代码][downloadURLs]：

```javascript
var downloadURLs = {
    linux: 'https://storage.googleapis.com/chromium-browser-snapshots/Linux_x64/%d/chrome-linux.zip',
    mac: 'https://storage.googleapis.com/chromium-browser-snapshots/Mac/%d/chrome-mac.zip',
    win32: 'https://storage.googleapis.com/chromium-browser-snapshots/Win/%d/chrome-win32.zip',
    win64: 'https://storage.googleapis.com/chromium-browser-snapshots/Win_x64/%d/chrome-win32.zip',
};
```

官方文档 [Puppeteer API Tip-Of-Tree § Environment Variables][environment-variables]：

> - `PUPPETEER_DOWNLOAD_HOST` - overwrite URL prefix that is used to download Chromium. Note: this includes protocol and might even include path prefix. Defaults to `https://storage.googleapis.com`.

可以发现 Puppeteer 默认从谷歌 [`https://storage.googleapis.com/chromium-browser-snapshots/`][chromium-browser-snapshots] 下载 Chromium，但是 `storage.googleapis.com` 在中国大陆受封锁，故下载安装失败。

## 结果

以下两种解决方案任选其一

### 环境变量

修改 `PUPPETEER_DOWNLOAD_HOST` 环境变量：

```bash
export PUPPETEER_DOWNLOAD_HOST=https://storage.googleapis.com.cnpmjs.org  # 使用 cnpmjs.org 提供的反向代理（实时更新）
# 或者
export PUPPETEER_DOWNLOAD_HOST=https://npm.taobao.org/mirrors  # 使用阿里淘宝的缓存镜像（定时更新）
```

### `.npmrc`

在 `~/.npmrc` 中追加 `puppeteer_download_host` 选项（同样适用于 yarn）：

```properties
puppeteer_download_host = https://storage.googleapis.com.cnpmjs.org
# 或者
puppeteer_download_host = https://npm.taobao.org/mirrors
```

[wappalyzer-gh]: https://github.com/AliasIO/wappalyzer
[quick-start]: https://github.com/AliasIO/wappalyzer#quick-start
[setup]: https://www.wappalyzer.com/docs/dev/setup

[downloadURLs]: https://github.com/puppeteer/puppeteer/blob/2b50d8cc32e023cbe9b149b79b0fda147d0c9ac6/utils/ChromiumDownloader.js#L27,L32
[environment-variables]: https://github.com/puppeteer/puppeteer/blob/main/docs/api.md#environment-variables
[chromium-browser-snapshots]: https://storage.googleapis.com/chromium-browser-snapshots/
