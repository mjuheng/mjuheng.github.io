---
layout: wiki
title: GitError
cate1: Error
cate2: Git
description: Git的一些错误处理。
keywords: Git
---

Git的一些错误处理。

## OpenSSL SSL_read: Connection was aborted, errno 10053

Git默认限制推送的大小，运行命令更改限制大小即可 增加缓冲，解决办法：
```
git config --global http.postBuffer 524288000
```

更改网络认证设置，解决办法：
```
git config http.sslVerify "false"
```
