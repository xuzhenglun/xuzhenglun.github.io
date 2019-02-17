title: Nexus5在Android下开启电信3G心得
date: 2014-12-11 10:20
tags:
- 电信
- Android 5.0
categories: 日志
---

流水账记录一下Nexus 5太子在破解使用电信卡的时候遇到的坑。

<!-- more -->

#需要
- 一个开启Diag的内核
- DFS程序
- ADB环境
- 需要Root以使用Nexus5 Field Test Mode程序

#准备
- 内核部分，我使用了[SykopomposStockLollipopKernel.zip](http://bbs.gfan.com/forum.php?mod=viewthread&tid=7762721&extra=&page=1)这个内核，原帖源于XDA，我没找到。已经开启了Root，Diag。

+ ROOT部分使用了CF-root的自动root。

+ Nexus5 Field Test Mode程序部分直接在Google play上下载的，最新版本已经支持了Android 5.0。破解过程遵循《NEXUS 5中文完美电信3G 2.0版》，没啥再加工的和自我创造，我要说的是几点心得。

#心得

+ 在5.0下面也可破解电信，应该无需刷回4.4.2

+ PRL可以通过短信更新，发短信内容“PRL到”10659165，将受到6条短信，并且提示更新成功。当然前提是能发短信了才能用这个方法更新PRL。

+ 老是显示禁用漫游提示符是因为SID没写，找到NAM选项卡HOME下的SID，写入自己所在归属的的SID，可以搜索下，能找到的。写入立即可以修复禁用漫游提示符的问题，但是如此修改后虽然提示搜到了电信信号但是还是不能打电话发短信，那说明变回了NV-only，需要再改这个。这个变回原理未知，无规律变回。但是SID和这项一起修改貌似可以减少变回的概率。别的分析在《NEXUS 5中文完美电信3G 2.0版》也有说明，可参考。

+ APN默认是荷兰的一家运营商，同样可以通过修改APN来解决上网问题，不需换卡。我的卡是换Nexus5才换的小卡，所以要换也没有别的卡可以换了，实测这个不影响根本问题，是同样可以使用电信的。

  修改APN方面，可以使用海卓。但是在4.0以后得系统中，海卓不再拥有权限修改APN。有个变通的方法是将其变成系统程序。利用钛备份高级版功能，转化为系统APP，然后将/system/app下的海卓apk剪切到system/priv-app下，即可恢复权限，自动修改APN。
 
  PS：变成荷兰之后，因祸得福一件事情。虽然微信等APP自动填写的电话会变成+31要修改，多点击两次略麻烦，但是，Google服务也会把你当作荷兰用户。所以无需root，无需安装local report enabler。修改语言清空Google搜索的数据，即可食用各项谷歌服务，Google now，fit点击就送@。@

+ 刷完之后没有3G，但是可以打电话，发短信。不能彩信，上网。设置了APN均无效。信号的确是CDMA-EVOD Rev.a 但是信号不咋地，很容易掉到1格。

  这个问题消耗了我大约6个小时时间。网络上大多评论换卡的问题，或者是字库被修过等硬件问题。诶，险些让人失去信心啊。
  其实貌似关键在于DATA选项卡下面的HDR AN和HDR AN LONG上，经过一晚上仅仅修改上面的PPP和PAP失败之后，第二天早上将HDR AN和HDR AN LONG清空之后再写入，问题莫名其妙的好了。

  PS：在不停重试写DATA的时候，PWD密码和PPP，PAP改写的还是要写的，读不出来是正常的。

  在一切OK之后，3G信号变成了满格，一切正常，彩信也正常。

+ 从4.4.X升级到5.0无需重新破解，无需备份还原等操作，只需要下载Factory image解压后和ADB合体，或者安装ADB之后cd到解压的文件夹，把flash-all.bat/flash-all.sh中的fastboot后面的-w参数删除，直接线刷即可升级到最新的系统。 

