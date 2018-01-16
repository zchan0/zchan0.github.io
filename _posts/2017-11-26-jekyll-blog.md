---
layout: post
title: Jekyll 折腾笔记
date: 2017-11-26 17:48 -0500
categories: notes frontend
---

* TOC
{:toc}
在 Hexo 和 Jekyll 之间纠结了蛮久，最后还是选了 Jekyll，一来可以直接用 GitHub pages（还蛮快的，感觉比gitlab快一些，二来资源也比较丰富，感觉什么插件都有现成的。

主题使用了自带的 Minima。话说以前不觉得好看，现在看着也还挺顺眼的。重点是有人一直在维护，代码看起来也很好改。

# SEO
之前写博客除了在 SegmentFault 那里别人做好了 SEO，自己的从来没做过这方面的优化。这次主要在 Google webmaster 那里挂了号，然后提交了 sitemap。

# Archive
练习写一个 page 页面，有好几种方式。如果是要出现在 header 的导航栏部分，则可以直接在根目录下新建类似于 `about.md` 或者 `about.html` 的文件，`layout` 可以指定 `_layout` 下的文件名。

Archive 主要是将 posts 名称按时间排序，HTML 部分如下：

```html
<div class="post">
	<h2>Archive</h2>
    {% assign postsByMonthYear = site.posts | group_by_exp:"post", "post.date | date: '%b %Y'"  %}
    {% for monthYear in postsByMonthYear %}
      <h3>{{ monthYear.name }}</h3>
        <ul>
          {% for post in monthYear.items %}
            <li><a href="{{ post.url }}">{{ post.title }}</a></li>
          {% endfor %}
        </ul>
    {% endfor %}
</div>
```

Jekyll 有许多预先定义好的变量，`group_by_exp` 也是新加的一个方法，可以直接拿来用。然后定义相应 CSS 的样式即可。

# TOC
虽然 Jekyll 有生成目录的插件，但是一来在 Mac 上安装有点麻烦，有个解析 XML 的插件安装起来有些折腾。另一方面，好不容易装好之后，得知这个目录并不能跟着 Jekyll 生成，如果要在 GitHub pages 上使用，只能采取将导出后的文件 push 到`gh-pages` 分支的方式，太麻烦。

好在 GitHub 支持 kramdown，这个 Markdown 解析器自带一个目录的语法。

```
// 注意 * 不可以省去。
* TOC
{:toc}
```

这样就会生成一个 ID 为 `markdown-toc` 的元素，目录是列表样式，CSS 的话对 `#markdown-toc` 进行样式修改就可以了。

# 排版
这是耗时最久的一部分，毕竟可以一直优化 = = 主要是两方面要注意，中文排版和中英文混排。

中文排版的问题可以看看这篇 [Best Practice for Chinese Layout](https://medium.com/@bobtung/best-practice-in-chinese-layout-f933aff1728f)，有蛮多值得注意的地方。比如中文里不用斜体。具体实现有几个现成的方案，比如 [typo.css](https://github.com/sofish/typo.css) 和 [Han](https://github.com/ethantw/Han)。

我用的是 typo，用起来比较方便（就一个文件），而且注释比较清楚。最后字体选择了端媒体同款 Noto Serif，链接的样式参考了 Wired，代码高亮调了一下背景色和字体，觉得还比较顺眼了 😄

# 最后

说起来 Blog 都折腾过好多次了，以致于以前写的东西零零散散到处都是。在没有新的 Blog 方案出来之前大概会一直使用 Jekyll，毕竟各方面都比较完善（可以尽情折腾~），而且 GitHub 的支持也很给力（完全不用管服务器的问题）。接下来会慢慢把其它地方的东西挪过来，然后继续晚上主题。