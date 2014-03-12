---
layout: post
title: "Using the Clang Static Analyzer"
date: 2014-02-28 17:25:33 +0800
comments: true
published: false
categories: iOS开发 Clang
---

[1]:https://www.mikeash.com/pyblog/friday-qa-2009-03-06-using-the-clang-static-analyzer.html
[2]:http://llvm.org
[3]:http://clang.llvm.org/StaticAnalysis.html
[4]:http://clang.llvm.org/

注：本文译自[Using the Clang Static Analyzer][1]。

本文主要内容：

1. Clang静态分析工具的介绍
2. Clang静态分析工具的使用方法

Clang是什么？
---

Clang是[LLVM][2]项目的一部分。LLVM本质上其实是编译器和JIT虚拟机非常重要的一个框架。
在Mac OS X上有一些llvm-gcc编译器，其实就是将gcc解析器/前端嫁接到了llvm代码生成器/后端。而Clang旨在填补gcc的位置，在LLVM项目里面作为解释器/前端，这样就产生了这样一个纯净的LLVM编译器。

这样做的意义何在？为什么不直接用gcc呢？其实也很简单：

1. gcc比较古老，复杂而且效率不高。

2. 大坨的遗存下来数据包，不仅大而且不易配合使用。

3. 相比之下，Clang就显得体积小，而且代码模块化做的更好。

要说的最重要的其实是，有些进取心强的人已经为Clang写了一套静态代码分析工具。本质上来讲，它是一个会帮你检查代码错误而不是把你的代码转成机器语言的编译器。

Clang静态分析工具，the Clang Static Analyzer（大家一般都直接使用“clang”来表示它，但是我还是坚持用CSA简称，因为Clang实际上是整个前端的名字，不只是CSA而已）尚处于开发中，而且还不完整。尽管如此，它还是非常实用的。

*注：这篇文章比较久远，clang现在已经很完善了。

Where Is It?
The main CSA web page can be found at http://clang.llvm.org/StaticAnalysis.html, and it can be downloaded using the link at the bottom right. I won't link directly to the download because it's still in very active development and so the download link updates frequently.

它在哪儿？
---
它的主页在~~[http://clang.llvm.org/StaticAnalysis.html][3]~~ [http://clang.llvm.org/][4]。你可以在主页里下载。

How To Use It
Using CSA is extremely easy. It provides a scan-build command which you simply invoke at the command line, passing the command to build your code as the parameters. scan-build will do some funky business to convince gcc to pass control over to CSA as it builds, allowing CSA to analyze all of your code instead of actually getting it built.

Since an example is worth a thousand words:

    $ gcc -framework Foundation test.m
    $ scan-build gcc -framework Foundation test.m
    ANALYZE: test.m main
    test.m:5:16: warning: Value stored to 'x' is never read
        int x = 0; x = 1;
                   ^   ~
    1 diagnostic generated.
    scan-build: 1 bugs found.
    scan-build: Run 'scan-view /var/folders/YT/YTiq3QDl2RW4ME+BYnLyRU+++TM/-Tmp-/scan-build-2009-03-06-3' to examine bug reports.
    $ 
And there it is, found a bug. If you run the command it mentions at the end, it gives a really swank HTML view.

Note that the scan-build command can be used not only with gcc but also with xcodebuild and even make. Running an analysis of your Xcode project is just a single command, usually as simple as scan-build xcodebuild in your project's directory.

A Better Example
Let's actually look at some code. I made the following contrived buggy code:

    #import <Foundation/Foundation.h>
    
    static void TestFunc(char *inkind, char *inname)
    {
        NSString *kind = [[NSString alloc] initWithUTF8String:inkind];
        NSString *name = [NSString stringWithUTF8String:inname];
        if(!name)
            return;
        
        const char *kindC = NULL;
        const char *nameC = NULL;
        if(kind)
            kindC = [kind UTF8String];
        if(name)
            nameC = [name UTF8String];
        if(!isalpha(kindC[0]))
            return;
        if(!isalpha(nameC[0]))
            return;
        
        [kind release];
        [name release];
    }
Obviously this code doesn't actually do anything useful, but of course it's meant only for illustration. There are several bugs in this code. Instead of trying to find them by looking, let's ask CSA:

    $ scan-build gcc -c test.m
    ANALYZE: test.m TestFunc
    test.m:5:23: warning: Potential leak of object allocated on line 5 and store into 'kind'
        NSString *kind = [[NSString alloc] initWithUTF8String:inkind];
                          ^
    test.m:18:17: warning: Dereference of null pointer.
        if(!isalpha(nameC[0]))
                    ^~~~~~~~
    2 diagnostics generated.
    scan-build: 2 bugs found.
    scan-build: Run 'scan-view /var/folders/YT/YTiq3QDl2RW4ME+BYnLyRU+++TM/-Tmp-/scan-build-2009-03-06-6' to examine bug reports.
And there we are two bugs! They're both pretty subtle too. The object that's leaked does get released at the end of the method. The problem is simply that there are some return statements in the middle that can cause that code not to be reached. CSA is clever enough to trace out those code paths and find the problem. The other bug requires a similar depth of analysis to find, as the null dereference can only happen if a previous if statement isn't followed.

You may have noticed that it missed a bug, though. This function releases name, which points to an object that it does not own. I'm not sure why CSA missed this, but it's important to keep in mind that it's not perfect and it won't catch everything.

CSA also sometimes sees false positives. These mostly occur when doing funky cross-method memory management tricks. For example, it's common when displaying a sheet to pass an object in to the void *context parameter so that the receiver of the end-sheet message can get information out of it. Proper memory management here requires retaining the context object when making the call, and then releasing it in the callback. Previous versions of CSA would consider the initial retain a leak, since it couldn't see that it was later balanced in another method. They appear to have fixed this particular case now, but other such cases will still be around, simply because it can't be perfect.

Conclusion
The Clang Static Analyzer, although limited, is an extremely useful tool. I guarantee that if you run it for the first time on any substantial base of Cocoa code, you will be surprised and frightened at what it finds. For tracking down leaks and many other common programming errors, it is invaluable. And it's under active development as part of a project with a great deal of support from Apple, so it will only get better.