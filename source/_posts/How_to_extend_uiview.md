title: 如何extend UIView
date: 2015-10-14 23:06:45
tags:
- extend UIView
- iOS
- iOS basics
---

主要是自己搞了好久才搞对，所以在这里记一下。。。
extend 出来的class里面要自己load 一下相应的nib

```objc
@implementation CutsomeUIView

- (id)init
{
    self = [[NSBundle mainBundle] loadNibNamed:NSStringFromClass(self.class) owner:self options:nil][0];
    return self;
}

- (id)initWithFrame:(CGRect)frame
{
    self = [[NSBundle mainBundle] loadNibNamed:NSStringFromClass(self.class) owner:self options:nil][0];
    if (self) {
        self.frame = frame;
    }
    return self;
}

@end
```
然后xib 里面把class 改成 CustomeUIView。注意是改View的class, 不是File’s Owner… 我之前搞成File’s Owner，然后又把所有的outlet都拉到File’s Owner里面，找了半个小时才找到哪里错了。。。

这样就完成啦～🐶
