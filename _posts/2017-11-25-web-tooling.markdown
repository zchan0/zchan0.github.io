---
layout: post
title:  "为什么前端需要构建工具"
date: 2017-11-25 19:42:20 -0500
categories: notes frontend
---

每次说到前端的时候都会有前辈说前端水深，谨慎入坑 = = 这么说的原因有很多，比如前端的技术更新很快呀，各种各样的框架还没来得及学就已经过时了，一个又一个的工具，简直眼花缭乱。作为刚涉足前端的萌新，我想我自己的一个小项目出发，说说为什么前端需要那么多构建工具。

当我在写完 Udacity 前端课程的一个叫记忆游戏的项目时，我找[朋友](https://timehub.cc/)帮我看了下[代码](https://github.com/zchan0/MemoryGame)，得到的反馈是这样的：

> js 是五六年前的写法，可以查查最近的标准，尽量使用新写法，用 babel 打包的话就可以写各种酷炫用法了

说到 JavaScript 的标准，最近几年一直在更新，不过浏览器支持的速度并不能完全跟上（考究 ES5，ES6，ES.Next 的历史的话可以参考这篇[文章](https://huangxuan.me/2015/09/22/js-version/)）。参考了下目前浏览器的[兼容情况](https://kangax.github.io/compat-table/es6/)，我决定先尝试下 ES6 的`class`和`module`的写法，毕竟模块化有利于代码复用，又增加了代码的可读性。

写完就是第一次发现**需要**工具的时候了。因为现在的浏览器并不能很好的支持 ES6 的语法，所以如果要让浏览器支持某些还未实现的 ES6 的 feature，首先需要一个 transpiler，它能够把代码中 ES6 的语法转换成浏览器可接受 ES6 之前的语法。比较常用的就是 Babel 了。

单独使用 Babel 很简单，用`npm`安装后直接用`cli`。

```bash
npx babel script.js --out-file script-compiled.js
```

或者不用`npx`，使用`npm run script`甚至直接用 `./node_modules/.bin/babel` 加上要转译的文件名即可。

这么做之后，原来的 JavaScript 只有一个文件 `app.js`，现在把代码分到几个文件，这样`app.js`就从几百行减少到了几十行，其它的每个文件的代码也不长。这样减少单个文件的长度也算是增加代码可读性的一种方式，顺便“物理隔离”了。

```bash
// 改造后的 js 目录结构
js/
├── app.js
├── controller.js
├── model.js
└── stopwatch.js
```

这个问题解决之后，我又会发现了一个新问题：原来只要引用一个 `app.js`，现在我们要引用四个文件了。引用多个文件最明显的一个问题是，网页需要多次请求资源文件，等待的时间会增加。另外，在[《ES6 in Depth: Modules》](https://hacks.mozilla.org/2015/08/es6-in-depth-modules/)中介绍了`import`的过程，这个过程其实是略耗费时间的，文章中提到一般可以提前完成这一步。

如果可以把所有的 JavaScript 文件都合并就好了。这时候就要用到传说中的 Gulp 和 Webpack 了。

> [Gulp](http://gulpjs.com/) is a **build tool** that helps you out with several tasks when it comes to web development. It's often used to do front end tasks like:
>
> - Spinning up a web server
> - Reloading the browser automatically whenever a file is saved
> - Using preprocessors like Sass or LESS
> - Optimizing assets like CSS, JavaScript, and images
>
> Webpack is a **bundler tool**.

简单来说，Gulp 是管理各种任务的工具，Webpack 可以用来打包（不过现在通过 loaders Webapck 也可以做到很多 Gulp 的工作了）。一个简单的 Webpack 配置文件如下：

```javascript
var path = require('path'); // 引用这个module

module.exports = {
    entry: './js/app.js', // 主要需要打包的文件
    output: {
        path: path.resolve(__dirname, 'docs'),
        filename: 'bundle.js'
    },
    module: {
        loaders: [{
            test: /\.js$/,
            exclude: /node_modules/,
            loader: 'babel-loader'
        }]
    }
};
```

这里要注意的是，`output.path` 使用的**绝对路径**，指定的是打包后的文件所存放的文件夹的路径。而另一个`output.publicPath`，则是相对于`output.path`中`html`的路径，默认是`''`。

后面用到 BrowserSync 时遇到的一个问题，就是因为我将`output.publicPath`设置成了和`output.path`相同的路径，导致本地服务器无法启动。对于我的目录情况（HTML被复制到了`docs`文件夹），使用默认值就可以了。

> Simple rule: The URL of your [`output.path`](https://webpack.js.org/configuration/output/#output-path) from the view of the HTML page.

Webpack loaders 可以让 Webpack 使用预先写好的包，对有关的资源文件进行处理，比如 Babel，装一个`babel-loader`。除此之外，仍需要先安装 Babel，并且指定`preset`（可以通过`.babelrc`来设置，也可以在`webpack.config`的loader里设置`query`）。

```
babel-preset-es2015
babel-core
```

另外，在 `webpackconfig` 文件中，`bebel-loader` 的 `-loader`可以不用加。`test`有点像 scanner，将符合条件的文件名传给 loader，只是对象是文件路径和文件名，遵循 glob。

如果没有 gulp 或者其它 task runner，就可以在 `package.js` 的 `script`里来定义`"build":"webpack --progress --watch" `，然后 `npm run build` 就可以实现打包了。

> 这里有两个问题：
>
> 1. loaders加载的时机？是在打包之前还是打包之后呢？
> 2. loaders处理的文件，是所有跟`import`有关的module的文件？还是什么？
>

现在已经完成了转译和打包，我们又引入了一个新的问题。原来修改了代码之后马上就能在浏览器中看到效果，现在却要先执行转译和打包的命令。我们当然希望能省掉这一步操作，对 hot-reloading（文件发生改动，包括HTML，CSS 和 JavaScript 时，能及时看到效果）的需求就应运而生了。

要实现 hot-reloading，要做两件事，一是起一个本地的服务器，二是能持续打包并通知这个服务器更新。

如果只使用 Webpack 的话，有一个`webpack-dev-server` 的工具可以做这两件事。但是，对于 CSS 和 HTML，可能还需要 preprocessors（比如 Sass 和 HTML 模版引擎），而对于静态文件的处理仍然由 Gulp 来做比较好[^1]。

综合考虑后，我采取的方案是，将打包 JavaScript 的工作交给 Webpack，task runner 仍由 Gulp 来做，由 BrowserSync 来起一个本地服务器。参考[这篇博客](http://jsramblings.com/2016/07/16/hot-reloading-gulp-webpack-browsersync.html)和 CSS-Trick 的[这篇文](https://css-tricks.com/combine-webpack-gulp-4/)。

> 其实这里也有说法是，单纯用 Webpack 的速度更快 [^2]。并且其实 Webpack 也是可以完成有关静态资源的处理的（通过loaders和plugins）。但是考虑到对更多的（主要是旧的）项目，还是采取上面的方式。
>

这次直接用的 Gulp 4，好处是不用装 `runSequence` 来按顺序执行任务了，可以直接使用两个新的函数：

```javascript
gulp.series()
gulp.parallel()
```

需要注意的是，`gulp.task` 需要对异步任务进行支持，即仍然返回 gulp stream、promise，或者接受一个 callback，这样可以 chain task，具体可以看[文档](https://github.com/lisposter/gulp-docs-zh-cn/blob/master/API.md#%E5%BC%82%E6%AD%A5%E4%BB%BB%E5%8A%A1%E6%94%AF%E6%8C%81)。比如[`gulpfile.js`](https://github.com/zchan0/MemoryGame/blob/master/gulpfile.js) 里 `clean` 和 `webpack` 的写法。

最后，BrowserSync 可以通过 `middleware`（Webpack已经提供的 `webpack-dev-middleware`和`webpack-hot-middleware`）来实现。

```javascript
gulp.task('browserSync', function() {
    browserSync.init({
        server: {
            baseDir: "./docs",

            middleware: [
                webpackDevMiddleware(bundler, {
                    publicPath: webpackConfig.output.publicPath,
                    stats: { colors: true }
                }),
                webpackHotMiddleware(bundler)
            ]
        }
    });
});
```

这里 `webpackDevMiddleware` 用到了 `output.publicPath`，需要注意前面说的路径问题。

以上，就是常见的构建工具的使用场景和简单使用注意事项了。

---

[^1]: http://jsramblings.com/2016/07/16/hot-reloading-gulp-webpack-browsersync.html
[^2]: http://www.alloyteam.com/2016/01/webpack-use-optimization/



