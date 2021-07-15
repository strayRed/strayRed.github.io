---
title: Image and Graphics Best Practices
author: strayRed
date: 2021-7-15 17:21:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---

对于UIImage对象中的图片数据，其是被压缩后保存在data buffer中的，如果需要将这部分数据显示到图形界面上，其需要进行解压并且写入Frame Buffer。

![](./assets/img/attachments/image_and _graphics_best_practices_1.png)

![image_and _graphics_best_practices_1](/Volumes/Data/blog/assets/img/attachments/image_and _graphics_best_practices_1.png)