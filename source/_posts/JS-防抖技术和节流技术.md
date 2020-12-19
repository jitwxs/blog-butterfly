---
title: JS 防抖技术和节流技术
categories: 前端
tags: Javascript
abbrlink: a265bb29
date: 2019-08-18 11:30:49
copyright_author: liuliu
---

在监听窗口进行 resize、scroll 等调用函数频率很高的操作时，如果每次都做相应的处理，则会加重浏览器的负担，导致渲染延迟，甚至是假死，这样会给用户带来非常糟糕的体验。为此我们必须在特定场景下限制调用频率，但是又不影响效果。

## 一、防抖

`防抖技术`：使得事件被触发 N 秒之后再执行回调，如果再 N 秒内再次触发，则重新倒计时。

```js
var btn = document.getElementById('btn');

var submit = function(value) {
    console.log(arguments[0]);
}

//监听按钮的点击事件
btn.addEventListener('click', debounce(submit, 'hello'));

function debounce(fn, value) {
    var timer = null;
    return function() {
    //如果timer还存在，则清空timer，并重新计时
        if(timer) {
            clearTimeout(timer);
        }
        timer = setTimeout(function(){
            fn.call(this, value);
        }, 3000);
    }
}
```

结果如下:

![防抖技术](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20190818113249.gif)

多次触发按钮的点击事件，但是这些都在3S之内触发的，所以每一次点击都会清空当前的计时器，然后重新生成新的计时器。所以等到最后一次超过 3S 才会回调 Submit。

防抖技术函数只会在频繁调用时只执行一次。比如用户在提交表单时，防止多次提交表单。或者搜索联想词功能类似时，在用户连续输入完成之后再向服务器发送。

## 二、节流

`节流技术`：当某一事件连续被触发时，保证一定时间内只做一次处理。

```js
var btn = document.getElementById('btn');

var submit = function (value) {
    console.log(value);
}

function throttle(fn, value) {
    var flag = true;
    return function() {
        //如果本次定时器还没执行，则直接返回
        if(!flag) {
            return ;
        }
        flag = false; 
        setTimeout(function(){
            fn.call(this, value);
            flag = true;
        }, 1000);
    }
}

btn.addEventListener('click',throttle(submit, "hellos"));
```

结果如下：

![节流技术](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20190818113250.gif)

多次点击按钮，但是回调函数只会以1S每次的频率进行执行。本次回调执行完成之后会将flag置为true，这样就可以生成下一次执行回调的定时器。

- **拖拽场景**：固定时间内只执行一次，防止超高频次触发位置变动
- **缩放场景**：监控浏览器 resize
- **动画场景**：避免短时间内多次触发动画引起性能问题

## 三、总结

防抖和节流的区别在于连续触发时执行的次数。如果只想在最后执行一次的场景，需要用防抖技术。如果想在期间以固定频率触发，则使用节流。
