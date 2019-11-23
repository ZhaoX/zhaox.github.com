---
layout: post
title: "Emoji来龙去脉"
description: "The History of Emoji"
category: Web
tags: [Web, Unicode, Emoji]
---
{% include JB/setup %}

1997年，日本人发明，定义在unicode的私有区域。

此时两个字节可以表示emoji。

IOS 4在日本支持emoji，使用的是这种私有编码。

2010年，unicode 6.0正式支持emoji，所有emoji重新编码。

此时的emoji所在的编码范围超出了2个字节。

IOS 5开始，支持unicode的emoji，放弃对日本人的私有emoji支持。

所以，同一个emoji，尤其是老的那些emoji，有两个编码。

不同系统的支持情况，不一样。

emoji的编码虽然相同，但是样子都是有专利的，因此各个系统，展示起来可能各不相同。

因为unicode的emoji超出了2个字节的表示范围，很多系统需要字符编码才能支持emoji。比如mysql，在更新了utf8mb4之后才支持。

日本人的编码，其实也不止一种，如果想要你的emoji在所有系统都能展示，需要你在展示的时候做转换。

emoji的发展历程、unicode编码演化、以及各种系统对字符串编码格式支持的变迁，三者交互在一起，引发各种有趣的问题。几乎每一次emoji不能展示，背后都与这三者相关。

因为涉及到系统底层的编码支持，很多时候问题不好解决。

本来想细细地写一篇文章，最近身体不适，精力不允许了。感兴趣的阅读下我列出的参考资料，很有意思。

### Reference

[https://gist.github.com/mranney/1707371#file-emoji_sad-txt](https://gist.github.com/mranney/1707371#file-emoji_sad-txt)
[https://en.wikipedia.org/wiki/Emoji](https://en.wikipedia.org/wiki/Emoji)
[http://doc.cat-v.org/bell_labs/utf-8_history](http://doc.cat-v.org/bell_labs/utf-8_history)
[https://emojipedia.org/unicode-6.0/](https://emojipedia.org/unicode-6.0/)
[https://stackoverflow.com/questions/7856775/how-to-convert-the-old-emoji-encoding-to-the-latest-encoding-in-ios5](https://stackoverflow.com/questions/7856775/how-to-convert-the-old-emoji-encoding-to-the-latest-encoding-in-ios5)
[https://en.wikipedia.org/wiki/Private_Use_Areas](https://en.wikipedia.org/wiki/Private_Use_Areas)
[https://www.iamcal.com/emoji-in-web-apps/](https://www.iamcal.com/emoji-in-web-apps/)
