---
title: CALayer hittest 遍历顺序
date: 2016-01-23 12:36:54
tags:
- CALayer
- iOS
- method swizzling
---
CALayer 的 hittest 可以返回一个点所在点最远的layer。那么这个检查sublayer的顺序是怎么样的呢？
为了得到这个顺序，我们先用method swizzling 在 [CALayer hittest:] 里面加上一行打印当前layer 的代码
```objc
@implementation CALayer(hitTestTracking)

static dispatch_once_t hitTestTrackingOnceToken;

+ (void)load
{
    dispatch_once(&hitTestTrackingOnceToken, ^{
        Class class = [self class];
        SEL oriSelector = @selector(hitTest:);
        SEL swSelector = @selector(xl_hitTest:);
        
        Method oriMethod = class_getInstanceMethod(class, oriSelector);
        Method swMethod = class_getInstanceMethod(class, swSelector);
        
        BOOL addedMethod = class_addMethod(class, oriSelector, method_getImplementation(swMethod), method_getTypeEncoding(swMethod));
        if (addedMethod) {
            class_replaceMethod(class, swSelector, method_getImplementation(oriMethod), method_getTypeEncoding(oriMethod));
        } else {
            method_exchangeImplementations(oriMethod, swMethod);
        }
    });
}

- (CALayer *)xl_hitTest:(CGPoint)p {
    CALayer *hittedLayer = [self xl_hitTest:p];
    NSLog(@"%@ %@ %@ hitted: %@", NSStringFromSelector(_cmd), self.name, self, (hittedLayer == nil ? @"NO" : @"YES"));
    return hittedLayer;
}
```

为了方便再加上打印sublayers的方法
```objc
- (void)printSubLayers {
    [self printSubLayersWithNumberOfIndentation:0];
}

- (void)printSubLayersWithNumberOfIndentation:(NSInteger)numberOfIndentation {
    NSInteger n = numberOfIndentation;
    while (n--) {
        printf("----");
    }
    printf("%s %s\n", self.name.UTF8String, self.description.UTF8String);
    for (CALayer *layer in self.sublayers) {
        [layer printSubLayersWithNumberOfIndentation:numberOfIndentation + 1];
    }
}
```

然后在view controller 随便创建一个view, 加入一些 layer 并调用 hittest 来进行
```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    self.layerView.layer.name = @"layer 0";
    
    CALayer *layer1 = [CALayer layer];
    layer1.backgroundColor = [UIColor grayColor].CGColor;
    layer1.frame = self.layerView.bounds;
    layer1.name = @"layer 1";
    
    CALayer *layer2 = [CALayer layer];
    layer2.frame = self.layerView.bounds;
    layer2.name = @"layer 2";
    
    CALayer *layer3 = [CALayer layer];
    layer3.frame = CGRectMake(100, 100, 5, 5);
    layer3.name = @"layer 3";
    
    CALayer *layer11 = [CALayer layer];
    layer11.frame = self.layerView.bounds;
    layer11.name = @"layer 11";
    
    CALayer *layer111 = [CALayer layer];
    layer111.frame = self.layerView.bounds;
    layer111.name = @"layer 111";
    
    CALayer *layer12 = [CALayer layer];
    layer12.frame = self.layerView.bounds;
    layer12.name = @"layer 12";
    
    CALayer *layer21 = [CALayer layer];
    layer21.frame = self.layerView.bounds;
    layer21.name = @"layer 21";
    
    CALayer *layer31 = [CALayer layer];
    layer31.frame = CGRectMake(100, 100, 5, 5);
    layer31.name = @"layer 31";
    
    [self.layerView.layer addSublayer:layer1];
    [self.layerView.layer addSublayer:layer2];
    [self.layerView.layer addSublayer:layer3];
    [layer1 addSublayer:layer11];
    [layer1 addSublayer:layer12];
    [layer11 addSublayer:layer111];
    [layer2 addSublayer:layer21];
    [layer3 addSublayer:layer31];
    
    CGPoint p = [self.view convertPoint:CGPointZero fromView:self.layerView];
    [self.layerView.layer printSubLayers];
    CALayer *layerHitted = [self.layerView.layer hitTest:p];
    NSLog(@"layer hitted: %@ %@", layerHitted.name, layerHitted);
}
```
得到如下输出：
```objc
layer 0 <CALayer: 0x7fc62be1e6e0>
----layer 1 <CALayer: 0x7fc62be26ef0>
--------layer 11 <CALayer: 0x7fc62be2c720>
------------layer 111 <CALayer: 0x7fc62be2c740>
--------layer 12 <CALayer: 0x7fc62be10e50>
----layer 2 <CALayer: 0x7fc62be2c6e0>
--------layer 21 <CALayer: 0x7fc62be10e70>
----layer 3 <CALayer: 0x7fc62be2c700>
--------layer 31 <CALayer: 0x7fc62be10e90>
2016-01-24 00:31:50.167 learnLayer[1966:258255] hitTest: layer 111 <CALayer: 0x7fc62be2c740> hitted: YES
2016-01-24 00:31:50.169 learnLayer[1966:258255] hitTest: layer 11 <CALayer: 0x7fc62be2c720> hitted: YES
2016-01-24 00:31:50.170 learnLayer[1966:258255] hitTest: layer 12 <CALayer: 0x7fc62be10e50> hitted: YES
2016-01-24 00:31:50.170 learnLayer[1966:258255] hitTest: layer 1 <CALayer: 0x7fc62be26ef0> hitted: YES
2016-01-24 00:31:50.170 learnLayer[1966:258255] hitTest: layer 21 <CALayer: 0x7fc62be10e70> hitted: YES
2016-01-24 00:31:50.170 learnLayer[1966:258255] hitTest: layer 2 <CALayer: 0x7fc62be2c6e0> hitted: YES
2016-01-24 00:31:50.171 learnLayer[1966:258255] hitTest: layer 31 <CALayer: 0x7fc62be10e90> hitted: NO
2016-01-24 00:31:50.171 learnLayer[1966:258255] hitTest: layer 3 <CALayer: 0x7fc62be2c700> hitted: NO
2016-01-24 00:31:50.172 learnLayer[1966:258255] hitTest: layer 0 <CALayer: 0x7fc62be1e6e0> hitted: YES
2016-01-24 00:31:50.172 learnLayer[1966:258255] layer hitted: layer 21 <CALayer: 0x7fc62be10e70>
```
从上面我们可以看到，hittest 是用的DFS来遍历图层树，而且不存在短路设置(layer3, layer 31)(其实有短路的，当maskToBounds ＝ YES 的时候就可以有短路了，会在接下来的部分里说明)，所有的layer都会被检查一遍，但是最后返回的结果是在遍历中满足条件的最后一个layer (layer 21)。

接着试着将layer31 的位置改一下使得测试点处于layer31 里面：
```objc
layer31.frame = CGRectMake(-100, -100, 5, 5);
```

输出变成了：
```objc
layer 0 <CALayer: 0x7fb920e9b5b0>
----layer 1 <CALayer: 0x7fb920e90420>
--------layer 11 <CALayer: 0x7fb920e8b730>
------------layer 111 <CALayer: 0x7fb920e8b750>
--------layer 12 <CALayer: 0x7fb920e8df90>
----layer 2 <CALayer: 0x7fb920e06460>
--------layer 21 <CALayer: 0x7fb920e8dfb0>
----layer 3 <CALayer: 0x7fb920e8b680>
--------layer 31 <CALayer: 0x7fb920e90c50>
2016-01-24 00:39:13.951 learnLayer[2076:266220] hitTest: layer 111 <CALayer: 0x7fb920e8b750> hitted: YES
2016-01-24 00:39:13.952 learnLayer[2076:266220] hitTest: layer 11 <CALayer: 0x7fb920e8b730> hitted: YES
2016-01-24 00:39:13.953 learnLayer[2076:266220] hitTest: layer 12 <CALayer: 0x7fb920e8df90> hitted: YES
2016-01-24 00:39:13.953 learnLayer[2076:266220] hitTest: layer 1 <CALayer: 0x7fb920e90420> hitted: YES
2016-01-24 00:39:13.953 learnLayer[2076:266220] hitTest: layer 21 <CALayer: 0x7fb920e8dfb0> hitted: YES
2016-01-24 00:39:13.953 learnLayer[2076:266220] hitTest: layer 2 <CALayer: 0x7fb920e06460> hitted: YES
2016-01-24 00:39:13.953 learnLayer[2076:266220] hitTest: layer 31 <CALayer: 0x7fb920e90c50> hitted: YES
2016-01-24 00:39:13.953 learnLayer[2076:266220] hitTest: layer 3 <CALayer: 0x7fb920e8b680> hitted: YES
2016-01-24 00:39:13.953 learnLayer[2076:266220] hitTest: layer 0 <CALayer: 0x7fb920e9b5b0> hitted: YES
2016-01-24 00:39:13.954 learnLayer[2076:266220] layer hitted: layer 31 <CALayer: 0x7fb920e90c50>
```
最终的结果变成了layer31，从这里我们就可以知道为什么不存在简单的短路设置（如果当前图层的位置的大小不满足条件就不遍历其子图层）了，因为子图层的位置与父图层并不存在直接关系，一个不在父图层中的点仍可能在其子图层中。

让我们再来把layer3 的maskToBounds 设为 YES:
```objc
layer3.masksToBounds = YES;
```
再跑一遍，输出又变了：
```objc
layer 0 <CALayer: 0x7fb8f1488f80>
----layer 1 <CALayer: 0x7fb8f1403cd0>
--------layer 11 <CALayer: 0x7fb8f1489a20>
------------layer 111 <CALayer: 0x7fb8f148b880>
--------layer 12 <CALayer: 0x7fb8f148b8a0>
----layer 2 <CALayer: 0x7fb8f14879e0>
--------layer 21 <CALayer: 0x7fb8f148b8c0>
----layer 3 <CALayer: 0x7fb8f1489a00>
--------layer 31 <CALayer: 0x7fb8f1489de0>
2016-01-24 00:47:55.756 learnLayer[2097:272073] hitTest: layer 111 <CALayer: 0x7fb8f148b880> hitted: YES
2016-01-24 00:47:55.757 learnLayer[2097:272073] hitTest: layer 11 <CALayer: 0x7fb8f1489a20> hitted: YES
2016-01-24 00:47:55.757 learnLayer[2097:272073] hitTest: layer 12 <CALayer: 0x7fb8f148b8a0> hitted: YES
2016-01-24 00:47:55.757 learnLayer[2097:272073] hitTest: layer 1 <CALayer: 0x7fb8f1403cd0> hitted: YES
2016-01-24 00:47:55.757 learnLayer[2097:272073] hitTest: layer 21 <CALayer: 0x7fb8f148b8c0> hitted: YES
2016-01-24 00:47:55.758 learnLayer[2097:272073] hitTest: layer 2 <CALayer: 0x7fb8f14879e0> hitted: YES
2016-01-24 00:47:55.758 learnLayer[2097:272073] hitTest: layer 3 <CALayer: 0x7fb8f1489a00> hitted: NO
2016-01-24 00:47:55.772 learnLayer[2097:272073] hitTest: layer 0 <CALayer: 0x7fb8f1488f80> hitted: YES
2016-01-24 00:47:55.772 learnLayer[2097:272073] layer hitted: layer 21 <CALayer: 0x7fb8f148b8c0>
```
我们可以看到当开启maskToBounds 之后，便开始有短路了，layer31 不再出现在遍历树里面。
