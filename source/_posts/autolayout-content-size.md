---
title: AutoLayout Content Size
date: 2015-11-15 12:27:24
tags:
- iOS
---
主要记下Intrinsic Size, Content Hugging/Compression Resistance Priority 是干什么的。

#### Intrinsic Size

一个view 原本应该有的size，比如一个text label 就是text label 显示完整的text 需要的size, image view 就是这个image 的size。有了Intrinsic Size 之后，再specify 了关于location 的constriant, autolayout 就可以帮你摆放这个view 了。

#### Content Compression Resitance Priority

一个View 反缩小的Priority。

#### Content Hugging Priority

一个View 反放大的Priority。
