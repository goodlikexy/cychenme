---
title: "Linux 内核看 socket 底层的本质：I/O 模型"
date: 2026-07-27
description: "从 socket 收包的两阶段模型出发，比较阻塞、非阻塞、I/O 复用、信号驱动和异步 I/O，并解释 select、poll、epoll 与 LT/ET 的工程取舍。"
tags:
  - "Linux"
  - "Socket"
  - "I/O 模型"
  - "epoll"
externalArticle: "/articles/linux-socket-io-models/"
layout: article-link
managedBy: beautiful-article-publisher
---

这是一篇使用 Beautiful Article 生成的独立网页文章。
