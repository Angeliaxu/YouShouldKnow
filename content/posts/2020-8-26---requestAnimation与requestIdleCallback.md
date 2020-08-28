---
title: requestAnimationFrame与requestIdleCallback
date: "2020-8-26"
template: "post"
draft: false
slug: "/post/requestAnimationFrameAndRequestIdleCallback/"
category: "JavaScript"
tags:
  - "js"
description: "requestAnimationFrame 用来优化动画的执行机制，让动画达到 16.66（1/60）毫秒的丝滑。在电脑显卡中有一块叫做前缓冲区域的地方，这个地方用来存放显示器需要显示图像的地方。显示器一般是每间隔 1/60 秒去前缓冲区域读取新的图像。"
socialImage: "/media/42-line-bible.jpg"
---

requestAnimationFrame 用来优化动画的执行机制，让动画达到 16.66（1/60）毫秒的丝滑。在电脑显卡中有一块叫做前缓冲区域的地方，这个地方用来存放显示器需要显示图像的地方。显示器一般是每间隔 1/60 秒去前缓冲区域读取新的图像。如果浏览器要更新显示的图片，会生成新的图片提交到显卡的后缓冲区域中。提交完成之后，GPU 会置换前缓冲和后缓冲，后缓冲变成前缓冲，前缓冲变成后缓冲，保证显示器每次都能读取到最新的图片，展示在人眼跟前。

但是怎么保证显示器与浏览器之间的工作关系呢，具体来说当显示器需要图片的时候，你浏览器需要给我准备好，让我有图片可读。其中用到了 VSync（vertical synchronization）垂直同步信号，当显示器读取当前帧图片完成之后，在准备读取下一帧图片之前，显示器会像 GPU 发送 VSync 信号，GPU 收到信号传递给浏览器进程，浏览器进程开始进行下一帧图片的计算。这个计算中如果我们定义了 requestAnimationFrame 的回调，那么这个回调会此时会执行。大致的过程如下：

![vsync](/media/vsync.png)

这也就解释了为什么 requestAnimationFrame 能达到让动画丝滑的效果。在显示器发送 Vsync 信号到浏览器进程，如果浏览器器进程在计算完图像有剩余的时间，上面说过 Vsyn 发送周期是 16.66 毫秒，此时计算图像只花了 8 秒或者更少的时间，那么还有剩余的 8 秒或者更多的时间，这时渲染主线程是空闲的，可能会进行垃圾回收，或者可以使用 requestIdleCallback 在这段空闲时间执行指定的回调函数。原来 requestIdleCallback 是在这种情况下执行，下面一张图了解下：

![frame](/media/frame.jpg)

结合上面的大致描述，可以把 requestAnimationFrame 和 requestIdleCallback 的执行时机大致的了解了一番。
接下来看看 requestIdleCallback。
举个栗子 🌰：当你在滚动页面的时候，你需要上传一些供分析的数据，或者当你点击 button，需要往页面插入元素。这个时候，你肯定不想在滚动或者点击按钮的时候，js 代码的执行阻碍了页面使用效果。所以，开发人员可以使用 requestIdleCallback 来解决上面描述的问题。也就是说，你能在不阻碍用户与页面交互的情况下，去执行 js 代码，这个功能从 chrome47 版本之后就可以使用。在完成 requestAnimationFrame 回调之后，浏览器进程还会进行样式计算，布局，重绘，或者其他消息任务，但是在使用 requestAnimationFrame 的时候，可以精确的知道，当前帧执行完毕之后，还剩余多少时间是可以使用的。

Best Practice
requestIdleCallback((param)=>{})，当回调执行的时候，回调参数是一个 deadline object，可以通过 deadline.timeRemaining()来获取当前存在的可用时间，这个时间从正数一直减少到 0。deadline.didTimeout 表示剩余时间是否用尽

```
function myNonEssentialWork (deadline) {
  while (deadline.timeRemaining() > 0) {
    console.log(deadline.timeRemaining());
    console.log(deadline.didTimeout,'didTimeout')
  }

}
requestIdleCallback(myNonEssentialWork);
```

下图截取的是部分上面代码执行的结果

![remainTime](/media/remainTime.jpg)

加入浏览器中一直有一项任务阻塞主线程，怎么才能保证 requestIdleCallback 必须能执行呢，这时候就需要第二个参数，接受一个毫秒级的数据，
保证回调会在指定的毫秒之后触发，值得注意的是，提供第二个参数之后，deadline.timeRemaining()一直都会是 0，deadline.didTimeout 一直都会返回 true。在需要使用第二个参数的情况下这种操作可能会引起页面阻塞。

以上就是对 requestAnimationFrame 与 requestIdleCallback 的介绍，基于 requestIdleCallback，还可以延伸出在 react Fiber 中的使用，以及为什么 React 需要对其 reconciliation 进行优化。
