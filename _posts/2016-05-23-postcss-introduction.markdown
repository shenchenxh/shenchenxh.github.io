---
layout: post
title:  "postcss简介"
date:   2016-05-23 11:04:54 +0800
categories: css
---
第二届[css conf](http://www.w3ctech.com/topic/1463)，勾三股四的演讲：[手机淘宝CSS实践启示录](http://jiongks.name/slides/css-memos/)，给我们介绍了他们是如何处理css相关事宜的。

![css memos](/assets/postcss-introduction/css-memos.png)

我们了解了所有css工具是集成到postcss中的。那么postcss是什么？

要理解这件事，首先要了解我们是怎么最终得到最终使用的css的。

#### 一、一开始，开始使用css的时候，我们所写的就是最终要用的css。

当人们意识到文档的结构和样式要分离，于是有了css。一开始大家按照css的规则，很朴素的写着css。

#### 二、后来，有人开始造轮子，写了各种css预处理器。

目的是为了让css更好编写、更好维护，有人发明了sass,less,stylus。

他们实现了变量、函数、嵌套等功能，同时也有一些还有一些第三方的工具可以扩展他们的功能。

什么是css预处理器，就是目标格式为 CSS 的预处理器都是css预处理器。一般来说，这些预处理器都是有自己的特定的语法。我们按照他们要求的语法去写，然后他们编译生成最终的css文件。

但是我们发现，我们要使用任何一种功能的代价都是很大的。

首先，我们要选择一种。一般来说，所有的东西都是有利有弊，所以你的每一个选择都是有得有失。相应的，选择某种之后，就要学习对应的语法知识。

其次，我们要安装他。众所周知，sass ＋ compass 不是很好安装。

接着，改变是一件很难的事。因为每一个预处理器的语法都不一样，所以你很难迁移。

最后，你的项目是完全依赖于 你选择的预处理器的， 如果后来他停止维护了，但是css仍然在不断发展，你该怎么办。

#### 三、 又后来，人们又发明了一种东西，叫做css后处理器。

这是什么意思，就是对CSS进行处理，并最终生成CSS的处理器。我们可以从概念上看出来，其实这也是属于css预处理器的。

例如clean-css，一种css的文件压缩工具。又比如说[Autoprefixer](https://github.com/postcss/autoprefixer)，以CanIUse上的**浏览器支持数据**为基础，自动处理兼容性问题。 他不需要你每次都去检查 是否需要加上**-webki-**等前缀，你只要按照最标准的写法，他会自动帮你生成。

例如： 你想写

```
a {
    display: flex;
}
```
但是呢，仅仅这样写是不够的。w3中是这样说的

> To avoid clashes with future CSS features, the CSS2.1 specification reserves a prefixed syntax for proprietary and experimental extensions to CSS.

> Prior to a specification reaching the Candidate Recommendation stage in the W3C process, all implementations of a CSS feature are considered experimental. The CSS Working Group recommends that implementations use a vendor-prefixed syntax for such features, including those in W3C Working Drafts. This avoids incompatibilities with future changes in the draft.

所以，因为这个属性还没有定稿，所以我们必须这样写

```
a {
    display: -webkit-box;
    display: -moz-box;
    display: -ms-flexbox;
    display: flex;
}
```

大家觉得，这个是太麻烦了，我们应该按照标准的css语法来写，但是为了浏览器的兼容性，为什么要多打这么多行。于是就有人写了各种各样的方法来解决这个问题，例如处理方案有：

* 如果你项目里有用到sass之类的预处理器，@include可以帮你解决
* 编辑器里的emmet,在属性前加-帮你自动生成
* 打开[prefixr](http://prefixr.com/)来格式化一下

后来，有人提出了autoprefixer，他可以分析你的css文件，并根据设定好的规则添加前缀。

然后，人们发现，结合自身的业务需求，可能要对css做一些变动，于是就写了一个个的小工具，一次次的去执行。

#### 四、最后，人们发现css预处理器灵活性不够，css后处理器不便于管理。于是有了我们的postcss。

postcss是什么？

> PostCSS itself is very small. It includes only a CSS parser, a CSS node tree API, a source map generator, and a node tree stringifier.
> All of the style transformations are performed by plugins, which are plain JS functions. Each plugin receives a CSS node tree, transforms it & then returns the modified tree.

他是一个css处理器的框架，可以用来分析css规则，并给出AST(抽象语法树)。所有的转换都是通过插件来完成的。

具体怎么用呢？

Start using PostCSS in just two steps:

1. Add PostCSS to your build tool.
2. Select plugins from the list below and add them to your PostCSS process

一般的你的自动化工具有Grunt, Gulp, webpack, Broccoli, Brunch, ENB, Stylus and Connect/Express等。以gulp举例， 

```
var postcss = require('gulp-postcss');
var gulp = require('gulp');
var autoprefixer = require('autoprefixer');
var mqpacker = require('css-mqpacker');
var csswring = require('csswring');

gulp.task('css', function () {
    var processors = [
        autoprefixer({browsers: ['last 1 version']}),
        mqpacker,
        csswring
    ];
    return gulp.src('./src/*.css')
        .pipe(postcss(processors))
        .pipe(gulp.dest('./dest'));
});

```

首先， 安装gulp-postcss以及你要使用的插件cssnext和cssnano，接着指定你要执行的源文件和生成的目标文件的路径，然后执行一下这个gulp文件就可以了。

大家贡献了很多很多解决特定问题的小工具， 例如postcss-nested可以用来写嵌套等等，然后是Based on NPM ecosystem，所以找插件和开发插件都很容易。

然后大家关心的是项目的迁移，如果你之前使用sass风格的语法，你可以整体使用<https://github.com/jonathantneal/precss>，或者你也可以选择你喜欢的特性。

例如：**Partials and imports**可以使用[postcss-import](https://github.com/postcss/postcss-import)， **变量**可以使用[postcss-simple-vars](https://github.com/postcss/postcss-simple-vars)等等。

postcss实际上是这样一种想法，你写css的时候，觉得我不想每次都写这个颜色的值，我想用变量啊，然后你就可以用 postcss-simple-vars；然后你觉得我不想每次都把这个相似功能重复写，你可以用函数，比如 postcss-mixins。

下面举一些比较通用的插件

* Autoprefixer

Autoprefixer是来添加vendor prefix，来处理兼容性问题。同样他也会移除一些旧的、不需要的前缀。
编译前

```
a {
    display: flex;
}
 
a {
    -webkit-border-radius: 5px;
            border-radius: 5px;
}
```

编译后

```
a {
    display: -webkit-box;
    display: -webkit-flex;
    display: -ms-flexbox;
    display: flex
}
a {
    border-radius: 5px;
}
```

* postcss-nested

可以嵌套写css，使结构更清晰直观，更易维护。

编译前

```
.phone {
    &_title {
        width: 500px;
        @media (max-width: 500px) {
            width: auto;
        }
        body.is_dark & {
            color: white;
        }
    }
    img {
        display: block;
    }
}
```

编译后

```
.phone_title {
    width: 500px;
}
@media (max-width: 500px) {
    .phone_title {
        width: auto;
    }
}
body.is_dark .phone_title {
    color: white;
}
.phone img {
    display: block;
}
```

* postcss-import

引入其他文件。

编译前

```
//例如，有个 body.css 文件
//body.css
body {
  background: black;
}

//再main.css中引入 body.css
//main.css
@import "main.css"
```

编译后

```
//main.css
body {
  background: black;
}
```

* cssnano

压缩css文件	

编译前

``` 
h1::before, h1:before {
	margin: 10px 20px 10px 20px;
	color: #ff0000;
    -webkit-border-radius: 16px;
	border-radius: 16px;
	font-weight: normal;
	font-weight: normal;
} 
/* invalid placement */ 
@charset "utf-8";
```

编译后

```
@charset "utf-8";h1:before{margin:10px 20px;color:red;border-radius:1pc;font-weight:400}
```

* cssnext

按照最新的css语法来写，而不需要考虑任何兼容性问题。跟js的babel有点像。

* postcss-mixins

函数	

编译前

```
@define-mixin icon $network, $color: blue {
    .icon.is-$(network) {
        color: $color;
        @mixin-content;
    }
    .icon.is-$(network):hover {
        color: white;
        background: $color;
    }
}

@mixin icon twitter {
    background: url(twt.png);
}
@mixin icon youtube, red {
    background: url(youtube.png);
}
```

编译后

```
.icon.is-twitter {
    color: blue;
    background: url(twt.png);
}
.icon.is-twitter:hover {
    color: white;
    background: blue;
}
.icon.is-youtube {
    color: red;
    background: url(youtube.png);
}
.icon.is-youtube:hover {
    color: white;
    background: red;
}
```

**总结**：现在大家的思路好像都是倾向于小而美，而不是大而全。当每个模块都专注于特定的问题时，那他多数情况下要比一个大而全的集中式框架更靠谱。

postcss使用中的注意点：

* 使用 变量 postcss-simple-vars 的时候， 如果变量放在一个文件中 import postcss-import 进来的时候，postcss-simple-vars 要放在 postcss-import 之后。
* 写注释 必须用 /* */ 而不能用 //。

## 相关参考
* <https://css-tricks.com/the-trouble-with-preprocessing-based-on-future-specs/>
* <http://zhaolei.info/2014/01/04/css-preprocessor-and-postprocessor/>
* <http://studiorgb.uk/from-sass-to-postcss/>
* <https://github.com/postcss/postcss>
