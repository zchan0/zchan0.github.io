---
layout: post
title:  "Web Tooling"
date:   2017-11-25 19:42:20 -0500
categories: notes frontend
---

* TOC
{:toc}

虽然之前用过Yoeman这类的scaffolding tool，但是看着一通`npm install`之后的一堆文件总觉得有点懵。趁着重构[Memory Game](https://github.com/zchan0/MemoryGame)，正好一点点学习为什么要用到Gulp，Webpack，Babel之类的工具，怎么简单使用，以及一些注意事项。

# ES6

大概在完成[这个commit](https://github.com/zchan0/MemoryGame/commit/40e1085d83089ae8f7a94973efca22a7844d3dbf)的时候，我找[学弟](https://timehub.cc/)帮我看了下代码，得到的反馈是：

> js是五六年前的写法，可以查查最近的标准，尽量使用新写法，用babel打包的话就可以写各种酷炫用法了

😂

所以就考虑用ES6重写。不过最后主要也就尝试了ES6的`class`和`module`的写法。原来的 js 文件只有一个 `app.js`，现在把代码分到几个文件：

```
js/
├── app.js
├── controller.js
├── model.js
└── stopwatch.js
```

这样`app.js`就从几百行减少到了几十行。这个时候就是第一次需要工具的时候了。因为现在的浏览器并不能很好的支持ES6的语法，所以首先需要一个 transpiler，能够把ES6的语法转换成浏览器可接受ES6之前的语法。

# Babel

单独使用Babel很简单，可以用`npm`安装后直接用`cli`。[文档在此](https://babeljs.io/docs/usage/cli/#babel)。

```bash
npx babel script.js --out-file script-compiled.js
```

或者不用`npx`，使用`npm run script`甚至直接用 `./node_modules/.bin/babel`。

转换后就要考虑打包的问题了。

之所以要打包，一方面是模块化有利于代码复用，以及增加可读性；另一方面将资源合并到一个文件，就可以在同一个文件请求中发回给客户端，优化用户体验。

在[《ES6 in Depth: Modules》](https://hacks.mozilla.org/2015/08/es6-in-depth-modules/)中介绍了`import`的过程，这个过程其实是略耗费时间的，而且一般来说是可以提前完成这一步。这时候就要祭出Webpack了。

# Webpack

Webpack有很多很好的特性，比如支持CommonJS和AMD之类的，不再赘述。直接看例子。

一个简单的配置文件：

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

## output

这里要注意的是，`output.path` 使用的**绝对路径**，指定的是打包后的文件所存放的文件夹的路径。而另一个`output.publicPath`，则是相对于`output.path`中`html`的路径，默认是`''`。

后面用到BrowserSync时遇到的一个问题，就是因为我将`output.publicPath`设置成了和`output.path`相同的路径，导致本地服务器无法启动。对于我的目录情况（HTML被复制到了`docs`文件夹），使用默认值就可以了。

> Simple rule: The URL of your [`output.path`](https://webpack.js.org/configuration/output/#output-path) from the view of the HTML page.

更多关于`output.publicPath`可以看[这里](https://webpack.js.org/configuration/output/#output-publicpath)。

## loaders

loaders可以让webpack使用预先写好的包，比如Babel，对有关的资源文件进行处理。

这里想在webpack里使用babel，需要额外装一个`babel-loader`。除此之外，仍需要安装babel本身的包，并且指定`preset`（可以通过`.babelrc`来设置，也可以在`webpack.config`的loader里设置`query`）。

```
babel-preset-es2015
babel-core
```

在 config 文件中，`bebel-loader` 的 `-loader`可以不用加。`test`有点像 scanner，将符合条件的文件名传给loader，只是对象是文件路径和文件名，遵循glob。（类似于正则）

如果没有 gulp 或者其它 task runner，就可以在 `package.js` 的 `script`里来定义`"build":"webpack --progress --watch" `，然后 `npm run build` 就可以实现打包了。

这里有两个问题：

1. loaders加载的时机？是在打包之前还是打包之后呢？
2. loaders处理的文件，是所有跟`import`有关的module的文件？还是什么？

# Gulp

现在已经完成了转译和打包，新的问题是，希望能实现 hot-reloading：文件发生改动时，（包括html，css和js）能及时看到效果。

这需要做两件事，一是起一个本地的服务器，二是能持续打包并通知这个服务器更新。

如果只使用webpack的话，有一个`webpack-dev-server` 的工具可以做这两件事。但是，对于css和html，都可能需要preprocessors（比如sass和html模版引擎），而对于静态文件的处理仍然由Gulp来做比较好[^1]。所以这里采取的方式是，将js打包的部分交Webpack，上层的task runner仍由Gulp来做，本地服务器由BrowserSync来完成。参考[这篇博客](http://jsramblings.com/2016/07/16/hot-reloading-gulp-webpack-browsersync.html)和CSS-Trick的[这篇文。](https://css-tricks.com/combine-webpack-gulp-4/)

> 其实这里也有说法是，单纯用 webpack 的速度更快 [^2]。并且其实 webpack 也是可以完成有关静态资源的处理的（通过loaders和plugins）。
> 但是考虑到对更多的（主要是旧的）项目，还是采取上面的方式。不过回头还是要试试直接用 `webapck-dev-server`，感觉简单一些 = =

这次直接用的 Gulp 4，好处是不用装 `runSequence` 来按顺序执行 task 了。可以直接使用两个新的函数：

```javascript
gulp.series()
gulp.parallel()
```

需要注意的是，`gulp.task` 需要对异步任务进行支持，即仍然返回 gulp stream、promise，或者接受一个callback，这样可以chain task，具体可以看[文档](https://github.com/lisposter/gulp-docs-zh-cn/blob/master/API.md#%E5%BC%82%E6%AD%A5%E4%BB%BB%E5%8A%A1%E6%94%AF%E6%8C%81)。比如[`gulpfile.js`](https://github.com/zchan0/MemoryGame/blob/master/gulpfile.js) 里 `clean` 和 `webpack` 的写法。

# BrowserSync

前面提到要在本地起一个服务器，并且能 hot-reloading 文件改动。BrowserSync可以通过 `middleware`（webpack已经提供的 `webpack-dev-middleware`和`webpack-hot-middleware`）来实现。

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

---

[^1]: http://jsramblings.com/2016/07/16/hot-reloading-gulp-webpack-browsersync.html
[^2]: http://www.alloyteam.com/2016/01/webpack-use-optimization/



