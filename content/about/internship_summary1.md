---
title: "Internship Summary in SenseTime (1)"
date: 2021-07-03T19:40:03+08:00
draft: false
---

> 基本上 deliver 了第一个 milestone，踩了不少坑，也积累了不少实践经验，写些东西积累一下吧。

总体来讲，第一个 milestone 还是很赶的，大部分时间都在追赶着完成需求，也有不少地方是快速查资料后写的，并没有完全弄懂其中的原理。这里把一些项目难点记下来，一方面辅助写在以后的简历上，另一方面也加深自己对这方面知识的理解。

## 项目中最难的点

首先是评论组件，一个是组件设计上的考虑，第二个是关于 contenteditable 的踩坑和实现。

**一. 组件设计**  
首先考虑 antd 组件的复用，但是因为要实现全部评论弹窗和多评论折叠，富文本展示，个性化程度高，于是最后决定自己重写这个组件。组件分为三个层次，commentArea -> commentUnit/commentEditor. commentArea 是整个评论的最高级容器，其中由 commentUnit 和 commentEditor 构成。commentEditor 是用户的输入框，而 commentUnit 抽象为一条评论的容器。特别地，commentUnit 作为父评论，其中又包含了子评论，多评论折叠，和评论弹窗，因为子评论样式和功能，弹窗内的评论样式和功能都高度相同，所以这个地方采用了一个递归性的设计：即在 commentUnit 中再次包含 commentUnit。这样就可以高度复用这个组件，减少代码量。缺点是，这个组件变得很重。commentEditor 较为 standalone，功能比较单一，和其他组件较少，但是因为要处理最后 submit 的动作，所有的相关参数要在这里汇合，由于多个参数的原因，显得这个处理有些笨重。

**二、contenteditable**  
功能简介：React 下实现一个带字数统计和限制的输入框，支持表情包，中英文混合输入。
难点：

1.  表情包怎么和中英文混合输入，原理是什么？  
     一般用于文字输入的 html 元素有 input, textarea, 和\<div contenteditable>，前二者只支持文本输入，并没有办法添加表情，所以我们确定使用 contenteditable 的方式来实现。那它的原理又是什么呢？实际上是直接操作 dom，获取光标位置，并在光标位置后用 dom 方法 insertnode 插入对应的 img 元素.  
     那第一个坑就出现了，和光标位置有关的 Range 和 Selection。关于这个的解说参见[这篇文章](https://segmentfault.com/a/1190000039359624).  
    按照我的理解，document 有内置 select 方法和对象，即获取 selection 模式下鼠标的选取对象，而 range 是 selection 的一个实例化体现。即通过 selection 和 range 我们能获得到鼠标光标，光标的状态描述等等。进一步的，我们需要获取到输入框内的 selection 对象，这里用到了 contains 方法。完整的获取、保存一个 valid range 的代码如下。我们用 useref 来记录这个 valid range。并在用户点击要添加的表情时的回调函数里使用这个 range，在 range 后 insertNode(img). 值得注意的是，这里我们实际上是对 div node 进行直接的 dom 操作。

        document.onselectionchange = () => {
           const selection = document.getSelection({);
           if (selection?.getRangeAt && selection.rangeCount) {
               const range = selection.getRangeAt(0);
               if (
                   emojiInput &&
                   emojiInput.current &&
                   emojiInput.current.contains(range.commonAncestorContainer)
               ) {
                   rangeOfInputBox.current = range;
               }
           }}
        };

2.  字数统计怎么做？（字数统计得分辨中英文，进行中文输入的时候，统计不能改变；英文的时候立即改变。）  
    关于字数统计和限制有两种思路。第一，我们用一个新的状态去记录用户的输入，然后在每次用户输入的时候检查这个状态值对应的字数，并设立状态值来改变剩余的字数；第二，我们用和文本无直接关系的状态量 num 来记录剩余字数，每次用户输入/添加表情时，即减掉一定的字数。很容易想到，第一种办法需要每次输入时进行一个新的计算，可能会影响效率和体验；所以我优先采用了第二种方法，但是但是，第二种办法不能 work：对于添加表情还好说，但是最关键的是：删除操作无法捕捉到，即删除的字数很难进行统计，于是作罢。
    第一种方法可以预见的是，必然也存在一个最主要的难点：

    - 怎么保持新状态和实际输入的一致性（有哪些事件监听的方法，又能从中获得什么信息用来维持一致性）对于 contenteditable,有一些常用的方法(React)：onInput, onFocus, onBlur, onCompositionStart, onCompositionEnd, onPaste… onInput 和 onChange 方法在 react 的实现中基本一致，在原生 js 中 onInput 使用也更方便，onChange 只会在 blur 之后才能获取事件。onInput 能捕捉到用户的键盘输入，删除等操作，捕捉事件会以 react 的 synthetic event 的实例形式外显，如图 target 目标能捕捉到事件发生的 dom 信息，里面存在 dom 的 innerhtml，innertext，outerHTML, outerText, childNodes(包含 nodeName, alt, nodeValue, outerHTML), style 等非常有用的信息。nativeEvent 则是原生的事件对象，其中包含像 inputType, data(当前输入的字符）等信息。

    -

3.  placeholder 怎么覆盖？
