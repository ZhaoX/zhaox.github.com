---
layout: post
title: "图片格式那么多，哪种更适合你？"
description: ""
category: MultiMedia
tags: [Image]
---
{% include JB/setup %}

本文介绍和比较几种常见图片文件格式的优缺点，并介绍不同的文件格式对Web应用程序性能的影响。

####有损vs无损

图片文件格式有可能会对图片的文件大小进行不同程度的压缩，图片的压缩分为有损压缩和无损压缩两种。

- 有损压缩。指在压缩文件大小的过程中，损失了一部分图片的信息，也即降低了图片的质量，并且这种损失是不可逆的，我们不可能从有一个有损压缩过的图片中恢复出全来的图片。常见的有损压缩手段，是按照一定的算法将临近的像素点进行合并。 
- 无损压缩。只在压缩文件大小的过程中，图片的质量没有任何损耗。我们任何时候都可以从无损压缩过的图片中恢复出原来的信息。

####索引色vs直接色

计算机在表示颜色的时候，有两种形式，一种称作索引颜色([Index Color](https://en.wikipedia.org/wiki/Indexed_color))，一种称作直接颜色([Direct Color](https://en.wikipedia.org/wiki/Color_depth#Direct_color))。

- 索引色。用一个数字来代表（索引）一种颜色，在存储图片的时候，存储一个数字的组合，同时存储数字到图片颜色的映射。这种方式只能存储有限种颜色，通常是256种颜色，对应到计算机系统中，使用一个字节的数字来索引一种颜色。
- 直接色。使用四个数字来代表一种颜色，这四个数字分别代表这个颜色中红色、绿色、蓝色以及透明度。现在流行的显示设备可以在这四个维度分别支持256种变化，所以直接色可以表示2的32次方种颜色。当然并非所有的直接色都支持这么多种，为压缩空间使用，有可能只有表达红、绿、蓝的三个数字，每个数字也可能不支持256种变化之多。

####点阵图vs矢量图

- 点阵图，也叫做位图，像素图。构成点阵图的最小单位是象素，位图就是由象素阵列的排列来实现其显示效果的，每个象素有自己的颜色信息，在对位图图像进行编辑操作的时候，可操作的对象是每个象素，我们可以改变图像的色相、饱和度、明度，从而改变图像的显示效果。点阵图缩放会失真，用最近非常流行的沙画来比喻最恰当不过，当你从远处看的时候，画面细腻多彩，但是当你靠的非常近的时候，你就能看到组成画面的每粒沙子以及每个沙粒的颜色。
- 矢量图，也叫做向量图。矢量图并不纪录画面上每一点的信息，而是纪录了元素形状及颜色的算法，当你打开一付矢量图的时候，软件对图形象对应的函数进行运算，将运算结果[图形的形状和颜色]显示给你看。无论显示画面是大还是小，画面上的对象对应的算法是不变的，所以，即使对画面进行倍数相当大的缩放，其显示效果仍然相同(不失真)。

####BMP

BitMap的缩写，是无损的、既支持索引色也支持直接色的、点阵图。

这是一种比较老的图片格式。BMP是无损的，但同时这种图片格式几乎没有对数据进行压缩，所以BMP格式的图片通常具有较大的文件大小。虽然同时支持索引色和直接色是一个优点，但是太大的文件格式格式导致它几乎没有用武之地，现在除了在Windows操作系统中还比较常见之外，我们几乎看不到它。

![BMP vs GIF](http://zhaox.github.io/assets/images/BMPvsGIF.png)

从上图中可以看到，在同样的图片质量下，BMP格式的图片文件大小是GIF格式的很多倍。

####GIF

全称Graphics Interchange Format，采用LZW压缩算法进行编码。是无损的、采用索引色的、点阵图。

GIF是无损的，采用GIF格式保存图片不会降低图片质量。但得益于数据的压缩，GIF格式的图片，其文件大小要远小于BMP格式的图片。文件小，是GIF格式的优点，同时，GIF格式还具有支持动画以及透明的优点。但，GIF格式仅支持8bit的索引色，即在整个图片中，只能存在256种不同的颜色。

GIF格式适用于对色彩要求不高同时需要文件体积较小的场景，比如企业Logo、线框类的图等。因其体积小的特点，现在GIF被广泛的应用在各类网站中。

![GIF vs JPEG](http://zhaox.github.io/assets/images/GIFvsJPEG.png)

####JPEG

JPEG是有损的、采用直接色的、点阵图。

JPEG图片格式的设计目标，是在不影响人类可分辨的图片质量的前提下，尽可能的压缩文件大小。这意味着JPEG去掉了一部分图片的原始信息，也即是进行了有损压缩。JPEG的图片的优点，是采用了直接色，得益于更丰富的色彩，JPEG非常适合用来存储照片，用来表达更生动的图像效果，比如颜色渐变。

与GIF相比，JPEG不适合用来存储企业Logo、线框类的图。因为有损压缩会导致图片模糊，而直接色的选用，又会导致图片文件较GIF更大。

![JPEG vs GIF](http://zhaox.github.io/assets/images/JPEGvsGIF.png)

####PNG-8

PNG全称Portable Network Graphics，PNG-8是PNG的索引色版本。PNG-8是无损的、使用索引色的、点阵图。

PNG是一种比较新的图片格式，PNG-8是非常好的GIF格式替代者，在可能的情况下，应该尽可能的使用PNG-8而不是GIF，因为在相同的图片效果下，PNG-8具有更小的文件体积。除此之外，PNG-8还支持透明度的调节，而GIF并不支持。
现在，除非需要动画的支持，否则我们没有理由使用GIF而不是PNG-8。当然了，PNG-8本身也是支持动画的，只是浏览器支持得不好，不像GIF那样受到广泛的支持。

![PNG-8 vs GIF](http://zhaox.github.io/assets/images/PNG8vsGIF.png)

可以看到PNG-8具有更好的透明度支持。

####PNG-24

PNG-24是PNG的直接色版本。PNG-24是无损的、使用直接色的、点阵图。

无损的、使用直接色的点阵图，听起来非常像BMP，是的，从显示效果上来看，PNG-24跟BMP没有不同。PNG-24的优点在于，它压缩了图片的数据，使得同样效果的图片，PNG-24格式的文件大小要比BMP小得多。当然，PNG24的图片还是要比JPEG、GIF、PNG-8大得多。

虽然PNG-24的一个很大的目标，是替换JPEG的使用。但一般而言，PNG-24的文件大小是JPEG的五倍之多，而显示效果则通常只能获得一点点提升。所以，只有在你不在乎图片的文件体积，而想要最好的显示效果时，才应该使用PNG-24格式。

另外，PNG-24跟PNG-8一样，是支持图片透明度的。

####SVG

全称Scalable Vector Graphics，是无损的、矢量图。

SVG跟上面这些图片格式最大的不同，是SVG是矢量图。这意味着SVG图片由直线和曲线以及绘制它们的方法组成。当你放大一个SVG图片的时候，你看到的还是线和曲线，而不会出现像素点。这意味着SVG图片在放大时，不会失真，所以它非常适合用来绘制企业Logo、Icon等。

![SVG vs PNG-8](http://zhaox.github.io/assets/images/SVGvsPNG8.png)
![SVG vs PNG-8](http://zhaox.github.io/assets/images/SVGvsPNG8_1.png)

SVG是很多种矢量图中的一种，它的特点是使用XML来描述图片。借助于前几年XML技术的流行，SVG也流行了很多。使用XML的优点是，任何时候你都可以把它当做一个文本文件来对待，也就是说，你可以非常方便的修改SVG图片，你所需要的只需要一个文本编辑器。

SVG并非只能绘制简单的Logo类的图片，它可以绘制出精致的图片的，比如下面这涨，嗯。

![SVG](http://zhaox.github.io/assets/images/SVG.png)

很多人想要这个图片的SVG版本。贴个链接在这里吧：https://upload.wikimedia.org/wikipedia/commons/1/14/Mahuri.svg

####WebP

WebP是谷歌开发的一种新图片格式，WebP是同时支持有损和无损压缩的、使用直接色的、点阵图。

从名字就可以看出来它是为Web而生的，什么叫为Web而生呢？就是说相同质量的图片，WebP具有更小的文件体积。现在网站上充满了大量的图片，如果能够降低每一个图片的文件大小，那么将大大减少浏览器和服务器之间的数据传输量，进而降低访问延迟，提升访问体验。

- 在无损压缩的情况下，相同质量的WebP图片，文件大小要比PNG小26%；
- 在有损压缩的情况下，具有相同图片精度的WebP图片，文件大小要比JPEG小25%~34%；
- WebP图片格式支持图片透明度，一个无损压缩的WebP图片，如果要支持透明度只需要22%的格外文件大小。

想象Web上的图片之多，百分之几十的提升，是非常非常大的优化。只可惜，目前只有Chrome浏览器和Opera浏览器支持WebP格式，所以WebP的应用并不广泛。为了使用更先进的技术，比如WebP图片格式，来压缩互联网上传输的数据流量，谷歌甚至提供了Chrome Data Compression Proxy，设置了Chrome Data Compression Proxy作为Web代理之后，你访问的所有网站中的图片，在经过Proxy的时候，都会被转换成WebP格式，以降低图片文件的大小。

####reference
[http://stackoverflow.com/questions/2336522/png-vs-gif-vs-jpeg-vs-svg-when-best-to-use](http://stackoverflow.com/questions/2336522/png-vs-gif-vs-jpeg-vs-svg-when-best-to-use)
[https://zh.wikipedia.org/wiki/%E5%8F%AF%E7%B8%AE%E6%94%BE%E5%90%91%E9%87%8F%E5%9C%96%E5%BD%A2](https://zh.wikipedia.org/wiki/%E5%8F%AF%E7%B8%AE%E6%94%BE%E5%90%91%E9%87%8F%E5%9C%96%E5%BD%A2)
[https://developers.google.com/speed/webp/](https://developers.google.com/speed/webp/)
[High Performance Browser Networking](High Performance Browser Networking) by Ilya Grigorik
