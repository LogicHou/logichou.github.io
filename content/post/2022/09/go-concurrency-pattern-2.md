---
title: "Go常见的并发模式 - 退出模式"
date: 2022-09-18T08:01:51+08:00
draft: false
toc: true
---

# 分离模式

1）一次性任务

2）常驻后台执行一些特定任务、如监视(motion)、观察(watch)等。其实现通常采用 for{...} 或 for{ select{...} } 代码段形式，并多以定时器（timer）或事件（event）驱动执行。