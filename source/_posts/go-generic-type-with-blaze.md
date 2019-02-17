---
layout: title
title: 用Blaze实现Golang泛型
date: 2019-02-09 04:45:16
tags:
- Golang
- Balze
- generic-type
categories: Golang
---

Golang是一门工程语言，语法元素很少很容易掌握。从而，只要不是太扯的人写出来的代码理应相差不大，并能够很快被人理解。泛型在现代编程语言中被广泛使用，这里不谈Go确实泛型是否是一种糟糕的设计，纯粹前段时间在`gvisor`的代码中看见一种实现泛型的方式比较新奇，因此拿来记录一下。

<!-- more -->

## 老套路
由于Go本身并不支持泛型，常见的编程套路有以下几种：

1. 尽可能的重复自己：常见食用方式即使用`vim`宏、`sublime`、`vscode`提供的多行编辑与正则替换功能，来达到Write Once, Paste Anywhere的目的。这也是广大人民最喜闻乐见的黑Go的方式，比如下面这张图：

![各路神仙利用编辑器复制替换](/images/6/go-generic.gif)

2. 尽可能的使用空接口和map：为了实践Don't Repeat Yourself的金玉良言，各路神仙纷纷使用起`interface{}`配合嵌套`map[string]interface{}`.完美,要用啥再下断言。毕竟`超过两次的就需要抽象了`，而空接口是一切抽象最终形态: 只要你想，空接口这种抽象能够装下整个宇宙。

3. 利用工具生成代码：整一个模板扣几个洞，佐上Jinja2、Mustache、Go Template啥的，基于字符串把我要的类型替换进去。啥？你说基于字符串的替换没有语义，不能用编译器检查与法还容易错？拜托，又不是不能用，要啥自行车？。至于自动化嘛，其实`go generate`还是能用的，至少比没有好对吧。

## 新亮点
`gvisor`项目里，`third_party/gvsync`这个包利用了Blaze+一个小工具，实现了一个简单的泛型方案。虽然不可能从语言级别上让出发已久的Go支持泛型，不过这也是一种比较优雅的方案了。这个方案本质上属于上面的套路3，是一种基于模板的泛型方案，但是更精致一些。

1. 依赖于构建工具`Balze`，在构建的时候按需生成代码。模板只写一份，哪个包用到这个泛型算法，就在那个中声明依赖和目标类型即可。而不是在泛型算法提供者那生成一大堆wrapper，真正做到了泛型想做到的解耦。

2. 类型安全：模板只是一个普通的Go源代码文件，可以被编译器进行语法检查，甚至运行测试。

3. 基于AST，而不是简单的字符串替换。直接操作抽象语法树，把模板源代码文件中的某一个类型替换为目标类型，不但保证序列化后的代码一定合法，而且实现了如果出现同名的元素不会被错误的替换，甚至在新的子作用域中，不会替换自作用域中同名的元素。

## 细节

要实现这个功能，需要一个命令行工具对模板进行渲染，以及构建系统的支持从而实现自动化构建。

### 渲染工具
这个渲染工具是`gvisor`为了在项目里使用而写的，并非一个专业的渲染模板。但是就功能上而言，应该已经能够胜任Go在泛型渲染上的需求。这个工具的代码位于`tools/go_generics`，主要功能就在此的3个文件之内，不到500行代码里。处理的流程大致按照以下步骤：
1. 将单个Go源码文件读入内存并利用标准库中自带的Parser解析到AST；
2. 把注释和import信息统计进`map`，以供生成代码的时候去除不需要的注释和依赖包；
3. 遍历AST，对Global作用域内的节点以及别处的引用按照Cmd Flag中的设置进行替换；
4. 遍历AST，删除原始类型的定义； 
5. 依照魔改后的AST，对照过程（2）中`map`中的信息重新生成新的Go源码。

利用这个工具，能够做到一下事情：
1. 将模板中定义的一个类型按照Go的语义重构成一个新的类型；
2. 将模板中定义的用来占位的类型以及其方法完全删除；
3. 将模板中的类型以及函数添加后缀，并修改此类型的引用（例如类型断言等）；
4. 修改全局定义的常量以及包名。

### 构建工具集成
Blaze的类Python的语法提供了极大的自由度，在`tools/go_generics/defs.blz`中定义了上述命令行工具的wrapper。利用这个封装，任何模块下的BUILD脚本都可以import这个定义文件，从而支持泛型模板的渲染。
1. `_go_template_instance_impl`实现了对命令行工具的封装，把命令行的Flag都封装成了一个配置对象。
2. 在BUILD中import上述定义文件，比如`third_party/gvsync`。在这个包里，`generic_atomicptr.go`和`generic_seqatomic.go`便是一个泛型。以后者为例，使用这个泛型的测试的BUILD文件：`third_party/gvsync/seqatomictest/BUILD`中所定义的：
```Python
go_template_instance(
    name = "seqatomic_int",
    out = "seqatomic_int.go",
    package = "seqatomic",
    suffix = "Int",
    template = "//third_party/gvsync:generic_seqatomic",
    types = {
        "Value": "int",
    },
)
```
将`third_party/gvsync/seqatomic_unsafe.go`作为模板，将Value类型换成int类型，并在导出的方法和类型后添加Int后缀。比如原本的`func SeqAtomicLoad(sc *gvsync.SeqCount, ptr *Value) Value`方法，经过渲染的结果则为`func SeqAtomicLoadInt(sc *gvsync.SeqCount, ptr *int) int`。

## 结论
Go没有泛型的支持，的确对代码的表现能力进行了极大的限制；而众多泛型的山寨版本也没一个好用的，本文提到的这种方式至少也提供了一种尝试。好处：泛型算法提供者和使用者解耦了，使用源生Go代码作为模板，支持测试和补全等；坏处：使用难度较高，心智负担不必直接支持泛型低，甚至更高。

所以，Go Sucks
