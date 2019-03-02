---
layout: post
title: "关于block坑爹的retain cycle"
date: 2014-01-23 11:20:00 +0800
comments: true
categories: iOS
---

[chuan1]: http://beyondvincent.com
[chuan2]: http://beyondvincent.com/blog/2013/12/14/124-communication-patterns
[ref1]: http://blog.random-ideas.net/?p=160


我们知道在使用block时，必须避免出现`retain cycle`。如果写代码不仔细造成了`retain cycle`，就会出现内存泄露。即使Xcode有静态代码分析工具，但很多时候Xcode也不太靠谱，根本什么提示都没有，所以还是自己写代码多注意比较好。

（下文译自[The Correct Way to Avoid Capturing Self in Blocks With ARC][ref1]，有小改动）

> 2019-3-2更新:
>
> 原始链接的文章已经访问不到了😅



以前避免`retain cycle`的方法大概像这样：

```objc
@implementation Foo
@property (strong) Bar *bar;
@property (strong) Baz *baz;

-(void)aMethod
{
    __block Foo *blockSelf = self;
    bar.block = ^
    {
        [blockSelf.baz doSomething];
    };
}
@end
```

所有在block中被访问的变量都会被block保留，而上面这段代码的避免了对`self`的直接引用。如果self持有这个block——直接或者间接，都会导致`retain cycle`。因此，我们使用了`__block`关键字来创建一个对`self`的临时引用（非保留），我们在block会使用这个引用代替`self`来操纵对象。但是很不幸，ARC出现后废弃了这种方法并且将`__block`作用改成了跟`strong`关键字一样。所以，我们需要一个新的方法来解决这个问题。

一个常见的（naive）方法就是像下面的代码一样去创建一个`weak`引用：
```objc
@implementation Foo
@property (strong) Bar *bar;
@property (strong) Baz *baz;

-(void)aMethod
{
    __weak Foo *blockSelf = self;
    bar.block = ^
    {
        [blockSelf.baz doSomething];
    };
}

@end
```

其实，这个跟之前的那个方法差不多是一样的，只是关键字变成了`__weak`。用这个引用放在block里面运行，没有`retain cycle`。但是不管怎样，这样做仍然有一些问题。

问题是，当`self`在block执行期间或block执行之前被销毁了，会怎样呢？我们乍一看应该是没什么问题的：因为`blockSelf`是一个`weak`引用，它在销毁后会自动置空，而且对nil发送消息是不会发生任何情况的。对吧？

嗯，如果block中的代码就这样简单，没有复杂的消息传递就结束了，也许倒也没什么问题。但是，假如`self`在这个block执行的期间，在其他线程里面被销毁了呢？假如在你block里面工作完成到一半的时候`self`就释放置nil了呢？难道你就这样放弃block中要做的事情，就这样丢着不管了吗？是啊！你这样做也许没有什么损失。呵呵，但是这里还真有一种更加糟糕情况潜伏着...

请看下面这一段看起来感觉跟之前差不多的代码：

```objc
@implementation Foo
@property (strong) Bar *bar;
@property (strong) Baz *baz;

-(void)aMethod
{
    __weak Foo *blockSelf = self;
    bar.block = ^
    {
        [baz doSomething];
    };
}

@end 
```

可以看到，这里我们是直接访问成员变量（很糟糕的方式），而没有通过Objective-C里面的推荐的成员访问器。这样写仍然会先访问到`self`，然后才能访问它的成员变量。如果你这样做的话，runtime没有使用成员访问器，而是使用了C级别的空指针引用。换种方式来看，上面的代码会变成大概下面这个样子：

```objc
@implementation Foo
@property (strong) Bar *bar;
@property (strong) Baz *baz;

-(void)aMethod
{
    __weak Foo *blockSelf = self;
    bar.block = ^
    {
        [self->baz doSomething];
    };
}

@end 
```

所以，如果`self`在block执行的途中被销毁了，呵呵，你就会遇到一个因为使用了空指针来访问成员变量而导致的crash。这种情况，跟对nil发送消息可不一样，因为你根本就是在对一个不存在的东西发送消息。

鉴于这些问题，最安全的做法是在block外部对`self`先获取一个`weak`引用，然在block内部执行代码的开始，创建一个对`self`的`strong`引用——这样在block执行期间，你不用担心`self`被销毁。然后等你不需要用到`self`的`strong`引用时候，可以将`strongSelf`置nil，或者干脆不管，等待代码执行结束退出（ARC，你懂的）。如果你按照这个方法做了，应该不会有什么问题了。所以，代码结构大致应该是这样的：

```objc
@implementation Foo
@property (strong) Bar *bar;
@property (strong) Baz *baz;

-(void)aMethod
{
    __weak Foo *blockSelf = self;
    bar.block = ^
    {
        Foo *strongSelf = blockSelf
        [strongSelf.baz doSomething];
    };
}

@end
```

另外，澄清一点，这段代码并不能保证线程安全，你仍然需要添加`@synchronize`域或者线程锁来保证线程安全。不过可以肯定的是，这样写一定比文章中提到的存在问题的代码要安全的多！

### 经验小结
在用block的过程中，不要只对看得到`self`加`weak`引用，如果你像文章里那样直接访问成员变量，也是同样会存在`retain cycle`的问题。解决问题办法也很简单，使用文章里提到的解决方案，或者干脆把大段的代码移出block，这样反而更好些。
