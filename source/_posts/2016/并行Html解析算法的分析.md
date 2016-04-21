layout: "post"
title: "并行Html解析算法的分析"
date: "2016-04-22 14:22"
tags: [Html, 解析, 并行, 文法]
---

并行Html解析作为一个研究性课题来说，十分具有挑战性。因为html特有的上下文关联特性，使得它很难被一般的自动机解析，即使是支持大部分编程语言的下推自动机，也很难描述。

本文，就从html的特点分析入手，解释html语法的特点，和将其并行化的一些思考。

<!--more-->

html是标记语言家族中的最为广泛使用的一种，作为网页描述的载体，是如今网络中必不可少的文件格式。html本身是很简单的，利用容易识别的标签，描述整个文档的层次结构。在不出现特殊情况下，是很容易被解析的。

例如：
```html
<!DOCTYPE html>
<html lang="zh_CN">
    <head>
        <meta charset="utf-8">
        <title>Hello World</title>
    </head>
    <body>
        <h2>Hello World</h2>
        <p>你好，世界！</p>
    </body>
</html>
```


## html文法特征

一般解析器解析的都是 [上下文无关文法][] ，简单的html可以是上下文无关的，但有些情况，html很难被用上下文无关文法描述。


然而遗憾的是，很多标签都有互相嵌套之间的关系，例如：`title`只能在`head`标记下。还不仅仅如此，html是一种容错性极强的语法，html解析器总是在尽可能的找出其中正确的部分进行解析，恢复顺序错乱或不匹配的标签。这个特性使得html的解析非常困难。


一般定义HTML文法的方式：[DTD][] (Document Type Definition) 这是传统的HTML定义方式，DTD由如下四种组成：

* 元素（Elements）
* 属性（Attribute）
* 实体（Entities）
* 注释（Comments）

由於DTD限制較多，使用時較不方便，近來已漸被 [XML Schema][] 所取代。


[上下文无关文法]: https://www.wikiwand.com/zh/%E4%B8%8A%E4%B8%8B%E6%96%87%E6%97%A0%E5%85%B3%E6%96%87%E6%B3%95

[DTD]: https://www.wikiwand.com/zh/%E6%96%87%E6%A1%A3%E7%B1%BB%E5%9E%8B%E5%AE%9A%E4%B9%89

[XML Schema]: https://www.wikiwand.com/zh/XML_Schema



## html的标签化

Html文档首先要经过分词器（tokenizer），成为一系列的标签流。
