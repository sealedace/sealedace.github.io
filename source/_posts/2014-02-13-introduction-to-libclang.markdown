---
layout: post
title: "Introduction to libclang"
date: 2014-02-13 18:37:07 +0800
comments: true
published: false
categories: iOS开发
---

[1]:https://www.mikeash.com/pyblog/friday-qa-2014-01-24-introduction-to-libclang.html
[2]:http://llvm.org/svn/llvm-project/cfe/trunk/include/clang-c
[3]:http://clang.llvm.org/doxygen/group__CINDEX.html

注：本文译自[Introduction to libclang][1]。


Clang编译器除了它一般正常的功能，同时它也有一套很清晰的API能将它作为一个库在你的代码中来使用它。今天，我就将Clang作为一个库来作个基本介绍。

##获取libclang库

Clang的库版本，创造性的叫做libclang（很显然的= =）。Xcode使用libclang很广泛，并且将之嵌入到dylib里去，这样一来，开发者就很方便的就可以使用了。你当然不会想在正式的应用开发中使用它，但这至少是一个很好的方法去开始并试验。这个库文件的路径很隐藏很深，你可以使用下面这个命令来获取它的完整路径：
```bash
	echo `xcode-select --print-path`/Toolchains/XcodeDefault.xctoolchain/usr/lib/libclang.dylib
```

当然，你需要添加这个目录到你的runpath，这样动态链接才能在runtime的时候找到这个库。如果你是通过终端来做build，你可以在库的路径前加上`-rpath`参数。链接的时候你需要将参数传递完整，像下面这样：
```bash
    clang `xcode-select --print-path`/Toolchains/XcodeDefault.xctoolchain/usr/lib/libclang.dylib -rpath `xcode-select --print-path`/Toolchains/XcodeDefault.xctoolchain/usr/lib ...
```
##获取头文件
坏消息是，Xcode没有提供这个库的头文件。好消息是因为使用的是稳定的API版本，所以你可以从Clang项目的版本库里面获取头文件。libclang的C版本头文件可以在这里找到：[http://llvm.org/svn/llvm-project/cfe/trunk/include/clang-c/][2]。你也可以使用svn获取一份本地拷贝：

    svn export http://llvm.org/svn/llvm-project/cfe/trunk/include/clang-c/
##文档
Documentation for libclang can be found here:
libclang的相关文档可以在这里找到：[http://clang.llvm.org/doxygen/group__CINDEX.html][3]。
C版本的API已经很完善了，并且以OO的风格去写，很容易看懂。
The C API is done in a fairly reasonable object-oriented style and is fairly easy to follow. I couldn't find any master overview document that discusses how to get started, but that's what this article is for!

Getting Started
The top-level object where everything else starts is called an index. Although it does more, libclang was originally built to help with code completion and indexing source files, and it looks like the name came from that. Creating an index is pretty easy:

    CXIndex index = clang_createIndex(0, 0);
The two parameters are boolean options. The first one determines whether declarations from PCH files are excluded. that doesn't really matter for my purposes, since I'm not using a PCH, but not excluding them seems like a decent start. The second parameter determines whether diagnostics are printed when parsing source code. If set, libclang will print warnings and errors just like the compiler would. I disabled this so that I can take control of how diagnostics are shown.

Next comes parsing a translation unit. In C terminology, a translation unit is basically a single compiled source file. Parsing a translation unit with libclang is much like compiling a file. It's so similar that you give the function command-line arguments just like you'd give them to clang on the command line. Let's start off with the arguments, which are just a couple of include paths needed to make the code compile properly:

    const char *args[] = {
        "-I/usr/include",
        "-I."
    };
It also wants to know how many arguments there are, which I compute from the array:

    int numArgs = sizeof(args) / sizeof(*args);
Next, parse the file. The function takes the index to work with, the file to parse, and the command line arguments. It also takes unsaved files to take into account, which I'm not going to use here. This allows you to, for example, #include files that you have in memory but haven't saved to disk. Finally, it takes a set of options for the parse, which allows you to do things like specify that the file is incomplete, that the results are intended for serialization, and other such things. No options are required here. This is the complete call:

    CXTranslationUnit tu = clang_parseTranslationUnit(index, "libclang.m", args, numArgs, NULL, 0, CXTranslationUnit_None);
Now we have a translation unit loaded. What do we do with it?

Examining Diagnostics
First, I want to print any diagnostics produced when parsing the file. I could have had this for free by passing 1 to clang_createIndex, but what fun would that be? More to the point, although printing these diagnostics at the command line isn't all that useful, this allows you to do more interesting things like show these diagnostics in a UI, or use them to annotate a source file, or anything else that doesn't involve simply printing them.

The first step is to find out just how many diagnostics were produced by the translation unit:

    unsigned diagnosticCount = clang_getNumDiagnostics(tu);
Then loop through all of them, fetching each one in turn:

    for(unsigned i = 0; i < diagnosticCount; i++) {
        CXDiagnostic diagnostic = clang_getDiagnostic(tu, i);
I want to print the location of each diagnostic, as well as the text. Getting the location is easy:

        CXSourceLocation location = clang_getDiagnosticLocation(diagnostic);
What do we do with a CXSourceLocation, though? Fortunately, it's fairly easy to turn this into line and column numbers:

        clang_getSpellingLocation(location, NULL, &line, &column, NULL);
The two NULL parameters can be used to obtain the file where the diagnostic occurred and its absolute offset within the file. There are several concepts of "location" that can be obtained from a CXSourceLocation, depending on exactly how you want to treat macro expansions and #line directives. clang_getSpellingLocation produces the final spot in the source file where the diagnostic occurred, which seems the most natural, at least for a basic exploration of libclang.

Finally, I want to know what the diagnostic actually says:

        CXString text = clang_getDiagnosticSpelling(diagnostic);
For reasons which aren't entirely clear to me, the term "spelling" is frequently used in the libclang API to indicate the textual content of an item. It's a bit odd, but not hard to work with, just something to be aware of when trying to find a function.

We don't know how to print a CXString, but fortunately it's easy to turn that into a C string by calling clang_getCString. The program can then print the diagnostic:

        fprintf(stderr, "%u:%u: %s\n", line, column, clang_getCString(text));
Finally, clean up the diagnostic string:

        clang_disposeString(text);
    }
Walking the Tree
Source code gets parsed into a tree, where the textual code is represented as a hierarchy of objects. A local variable is contained within a scope which is contained within a function which is contained within a file, for example. libclang produces this tree and allows us to walk through it to find things. To illustrate how that works, I'll show code that prints every variable declaration in the file.

Rather than expose the tree directly, libclang provides API that will walk the tree, invoking a callback at every node. There's a version that takes a function pointer, and fortunately there's another version that takes a block. Each node of the tree is represented as a CXCursor. The function takes an arbitrary cursor and walks its children, and then there's a convenient helper to get the top-level cursor for a translation unit:

    clang_visitChildrenWithBlock(clang_getTranslationUnitCursor(tu), ^(CXCursor cursor, CXCursor parent){
Each cursor has a kind that indicates just what this thing is. We want variable declarations, so check for that:

        if(clang_getCursorKind(cursor) == CXCursor_VarDecl) {
The first thing we'll do is find out where the cursor is located. Cursors don't directly expose a location, but rather a range. Of course, a range is just a start and end, and I'll grab the start location as the cursor's location:

            CXSourceRange range = clang_getCursorExtent(cursor);
            CXSourceLocation location = clang_getRangeStart(range);
Next, I want to pull useful information out of that CXSourceLocation. The call is much like before, although I want to extract the file as well, not just the line and column numbers. This is because libclang will find not only variable declarations in my own file, but also in headers that it includes, and I want to know where each declaration was:

            CXFile file;
            unsigned line;
            unsigned column;
            clang_getFileLocation(location, &file, &line, &column, NULL);
A CXFile by itself isn't enough. Instead, I want the file's name:

            CXString filename = clang_getFileName(file);
I also want the variable's name, which is just the text content (or "spelling") of the cursor itself:

            CXString name = clang_getCursorSpelling(cursor);
Now I'm ready to print the information:

            fprintf(stderr, "%s:%u:%u: found variable %s\n", clang_getCString(filename), line, column, clang_getCString(name));
And clean up the strings:

            clang_disposeString(name);
            clang_disposeString(filename);
        }
Finally, the block returns a value which indicates whether traversal should halt, skip children, or recurse into children. We want to visit everything, so we'll tell libclang to recurse:

        return CXChildVisit_Recurse;
    });
Let's try it out. Here's the output it produces on my file:

    libclang.m:9:5: found variable index
    libclang.m:11:5: found variable args
    libclang.m:15:5: found variable numArgs
    libclang.m:17:5: found variable tu
    libclang.m:19:5: found variable diagnosticCount
    libclang.m:20:9: found variable i
    libclang.m:21:9: found variable diagnostic
    libclang.m:23:9: found variable location
    libclang.m:24:9: found variable line
    libclang.m:24:24: found variable column
    libclang.m:27:9: found variable text
    libclang.m:34:13: found variable range
    libclang.m:35:13: found variable location
    libclang.m:37:13: found variable file
    libclang.m:38:13: found variable line
    libclang.m:39:13: found variable column
    libclang.m:42:13: found variable filename
    libclang.m:44:13: found variable name
It works!

Cleaning Up
Both the index and translation unit are objects that we hold onto and are responsible for disposing of. Cleaning them up is, fortunately, easy:

    clang_disposeTranslationUnit(tu);

    clang_disposeIndex(index);
Conclusion
libclang provides a nice interface for using Clang's knowledge of C and similar languages to find information about source files. It's possible to create and walk the abstract syntax tree, generate errors and warnings, and even perform autocompletion. I've only scratched the surface of what's possible, but I hope it's enough to get you started.