---
title: raft
slug: raft
date: 2021-04-30
tags: ["Raft", "8.624"]
categories: ["Algorithm"]
image: 
draft: true
---



### 算法流程

```sequence
Client -> Server1: 发送请求
Server1 -> Server2: 备份log
Server1 -> Server3: 备份log
Server2 -> Server1: 确认收到log
Server1 -> Server1: 已获得多数\n本地commit
Server1 -> Client: 返回请求
Server3 -> Server1: 确认收到log
Server1 -> Server2: 通知Leader已commit
Server1 -> Server3: 通知Leader已commit
```

