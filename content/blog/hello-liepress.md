+++
date = '2026-06-09T18:00:00+08:00'
draft = false
title = 'Hello, liepress'
+++

## 缘起

偷懒了很久，一直在瞎搞着，写写`Rust`，看看新闻，上班摸鱼。

最近，看到有个项目[ironpress](https://github.com/gastongouron/ironpress)，看它说的纯`Rust`实现了把markdown转换为pdf，很诱人呢，就试了下，结果在中文+英文时，就遇到出现了奇怪的乱码的情况。

尝试了clone下来，想调试一把，才发现根本搞不动，还是技术太菜了。

再想到，反正有AI帮忙嘛，就自己尝试撸一把，写个小工具，就有了这个项目[liepress](https://github.com/zzzdong/liepress)。

## 发展

liepress的目标只是简单的把markdown转换为pdf，所以它的架构就能简单了。

1. 首先找个库来解析markdown到AST。

2. 再遍历AST，进行布局以及分页，生成视觉元素。

3. 最后，把视觉元素渲染到pdf。

基本一切都是站在巨人的肩膀上，拿来主义。

其中，个人觉得关键的就是：

- [parley](https://crates.io/crates/parley) 来做好文本排版，它类似C生态中的pango吧，layout文本，得到每一行的数据，然后根据需要切割、分页等。

- [krilla](https://crates.io/crates/krilla) 来实现pdf渲染，它提供很便捷的接口来，可以类似绘图的方式来渲染元素到pdf中。它还自带字体裁剪，我严重怀疑ironpress的字体问题就是字体嵌入有问题导致的。

另外一点有意思的是，你可以看到：

它里面的有个visual系统，简易的分层，然后就有一个渲染器，使用不同的后端来渲染，目前支持`pdf`、`svg`和`png`。这里其实是从其他项目拿过来的，主要是想可以输出svg调试，让AI读取到分页的结果。

## 来试一试

现在，可以访问[在线转换](https://zzzdong.github.io/liepress/)来试一试效果。

虽然，目前还是只能支持一些简单的markdown语法，而且没有经过什么测试，但是，在自己用的简易中文markdown，还是挺让我自己满意的。


## 后话

ironpress中的markdown到html，再到pdf的流程，还是挺清晰的，我看看要不要参照一下，抄个类似的。


---

PS.

- liepress的实现，可以说绝大部分是AI写的，感激[TRAE](https://www.trae.cn/)，让我白嫖。当然也少不了DeepSeek的帮助，遇到难问题，还是找他来解决不少。

- 现在AI盛行了，我这半桶水，代码写不动，不知如何是好了。

- 下一次可能很快见，再分享一个小玩意项目。
