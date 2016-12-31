---
title: Swift Optional
date: 2015-10-10 23:00:08
tags:
- Optional
- Swift
- iOS
- iOS basics
---
本🐶正在学习Swift.被Optional 各种？和！搞得有点晕。参考了[Ref1: Swift Optionals: When to use if let, when ? and !, when as? and as](http://www.touch-code-magazine.com/swift-optionals-use-let/), [Ref2: What is an optional value in Swift?](http://stackoverflow.com/questions/24003642/what-is-an-optional-value-in-swift)之后总结一下Optional是怎么回事，要怎么用。
本文的例子都来自于[Ref1](http://www.touch-code-magazine.com/swift-optionals-use-let/)。

#### 为什么有 Optional
在obj-c 里面我们可以用nil 来表示空值，比如
```objc
NSArray arr = nil;//这样我们就都知道arr 没有被赋值
```
但是Swift里面我们并不能将nil直接assign给变量，因为Swift 是type safe的。

Swift also introduces optional types, which handle the absence of a value. Optionals say either “there is a value, and it equals x” or “there isn’t a value at all”. Optionals are similar to using nil with pointers in Objective-C, but they work for any type, not just classes. Optionals are safer and more expressive than nil pointers in Objective-C and are at the heart of many of Swift’s most powerful features.

Optionals are an example of the fact that Swift is a type safe language. Swift helps you to be clear about the types of values your code can work with. If part of your code expects a String, type safety prevents you from passing it an Int by mistake. This enables you to catch and fix errors as early as possible in the development process.

不允许nil, 就避免了在这个变量上操作的unexpected behavior。
但还是得有个方法来表示空值啊，于是就有了Optional，Optional就代表这个变量(比如NSArray)可以装两种东西，一种是nil, 一种是自己的类型（NSArray)。
Optional 背后是类似这样的东西（经过了语法糖包装的）

```objc
enum Optional : NilLiteralConvertible {
  case None
  case Some(T)
}
// Int? 已经不是Int 而是Optional<Int> ...
var height: Optional<Int> = Optional<Int>(180)
```

#### 如何使用 Optional 变量
简单讲就是，如果你确定一定有值，那么就用！
如果你不确定，而且不在乎没有值的情况，用？
要handle 没有值的情况，用if let

! 很好理解
？一个例子就是，你可能有一个navigationController。
```objc
controller.navigationController?.pushViewController(myViewController, animated: true)
```

你想用它来push 一个view 且只当它存在才push。如果navController 存在，这一句就会执行，不然整句都会被当成nil

if let 允许你这样写
```objc
if let nav = controller.navigationController {
    nav.pushViewController(myViewController, animated: true)
} else {
    //show an alert ot something else
}
```
