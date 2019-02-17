title: Go Get Golang.org失败的原因
date: 2015-11-30 21:36
tags:
- GFW
- Golang
categories: Golang
---

最近在碰到一个很奇怪的问题，Go get工具在下载golang.org/x/tools下源代码的时候总是失败，但是以前并没有这个问题。Chrome已经使用了Shadowsocks穿墙，但是直接访问这个URL后发现在显示`Nothing to see here`之后就跳转到了文档页面。本以为是Google Code关闭带来的影响。不过时间过去了一周，并没有改善，奇怪的是老外对此并没有哀嚎遍野。为此有必要看一下`go get`到底发生了啥了。

<!-- more -->

无意中翻到一篇讲述了Go get的过程的文章：`https://texlution.com/post/golang-canonical-import-paths/`.

文章的意思很简单，大概说的是Go get工具会去所Import的路径下载代码，如果没有，会寻找meta标签中的别名/镜像Repo。

    ➜ curl golang.org/x/tools/cmd/rename
    <!DOCTYPE html>
    <html>
    <head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
    <meta name="go-import" content="golang.org/x/tools git https://go.googlesource.com/tools">
    <meta name="go-source" content="golang.org/x/tools https://github.com/golang/tools/ https://github.com/golang/tools/tree/master{/dir} https://github.com/golang/tools/blob/master{/dir}/{file}#L{line}">
    <meta http-equiv="refresh" content="0; url=https://godoc.org/golang.org/x/tools/cmd/rename">
    </head>
    <body>
    Nothing to see here; <a href="https://godoc.org/golang.org/x/tools/cmd/rename">move along</a>.
    </body>

而我们之前看到的`Nothing to see here`只是一个假象，meta标签并不被浏览器显示。傻傻的以为Repo真的被迁移了。


这才发现到了学校之后一直用的SS-local+SwitchOmega上网，并没有配置OpenWrt的透明代理，这才主观上感觉最近Golang.org才挂掉。

# 总结

不是Golang.org代码迁移了，是撞墙了。


# 解决方案：


 1. Proxychains+Shadowsocks的临时使用方案
 2. OpenWrt+Shadowsocks透明代理一次部署，一劳永逸方案
 3. 从Github上Get下来，一条条的手动下载。
