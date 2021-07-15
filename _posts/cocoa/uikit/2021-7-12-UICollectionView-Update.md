---
title: UICollectionView Update
author: strayRed
date: 2021-7-12 17:21:00 +0800
categories: [ios, cocoa, uikit]
tags: [ios, cocoa, uikit]
---

| Action |               Notes               |          Index Path Semantics          |
| :----: | :-------------------------------: | :------------------------------------: |
| Delete |    Descending IndexPath order     |             Before updates             |
| Insert |     Ascending IndexPath order     |             After updates              |
|  Move  |                                   | From: Before updates To: After updates |
| Reload | **Decompose: Delete and inserts** |             Before updates             |

`Reload`和`Move`都可以拆分为`Delete`和`Insert`，所以在实际更新CollectionView时，需要避免同时对同一个`IndexPath`进行`Delete`和`Insert`操作。

通过以下几种方式更新UICollectionView会出现异常：

- Move and Delete the same location
- Move and Insert to the same location 
- Move more than 1 location to the same location 
- Referencing an invalid IndexPath

对于数据源的更新，同样需要遵循以下几个准则：

- Decompose Move into Delete and Insert updates 
- Combine all Delete and Insert updates
-  Process Delete updates first, in descending order 
- Process Insert updates last, in ascending order 