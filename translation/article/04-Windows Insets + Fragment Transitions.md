### 04-Windows Insets + Fragment Transitions

这篇文章是我正在撰写的关于Fragment过渡效果短篇系列的第二篇。可以在下面找到第一篇文章的链接，里面介绍了如何是fragment转换生效。[**Fragment Transitions** Getting them working](https://chris.banes.dev/2018/02/18/fragmented-transitions/)

> 在我进一步讨论之前，我将假设你知道什么是WindowInsets，并且知道它们是如何传递的。如果你不知道，我建议你看一下这个视频（当然，也是我讲的🙋‍♂️）。[**Becoming a master window fitter 🔧**](https://chris.banes.dev/talks/2017/becoming-a-master-window-fitter-lon)

我承认，但我在写这个系列的第一篇文章时，我用视频隐瞒了大家一些问题。我实际上遇到了WindowInsets的问题，这意味着我实际上最终的到的内容如下：

[](https://raw.githubusercontent.com/mzzdxt/AndroidNote/master/translation/imgs/pic_04_01.gif)

转换会破坏状态栏的处理。

这并不是我在第一篇文章中展示出来的效果。我不想让第一篇文章过于复杂，所以我觉得单独写这一篇文章。无论如何，你可以看到在添加转换时我们突然失去了对状态栏的处理，并且所有的视图被推到了状态栏的后面。

#### 问题所在

这两个fragment都大量使用WindowInsets在系统栏后面进行绘制。Fragment

A使用CoordinatorLayout和AppBarLayout，而Fragment B使用了自定义WindowInsets处理（通过OnApplyWindowInsetsListener）。无论如何实现，转换都会变得混乱。

为什么会出现这种情况？当你使用fragment转换时，退出（Fragment A）和进入（Fragment B）内容视图实际发生的情况如下：

1. 转换被提交。
2. 因为我们在fragment A使用了退出转换，视图A将会被保持，转换动画会在它上面执行。
3. 视图B被添加到容器View中，被立即设置为不可见。
4. Fragment B的进入和Shared element enter转换动画开始。
5. 视图B被设置为可见状态。
6. 当Fragment A的退出转换完成后，视图A被从容器View中移除。

这一切看起来不错，为什么它会突然影响WindowInsets处理呢？这是因为在过渡期间，两个Fragment的视图都会存在于容器中。

这听起来完全没有问题不是吗？在我的场景中，两个Fragment的视图都想操作和使用WindowInsets，因为他们都想成为屏幕上的“主”视图。但是只能有一个视图会接收到WindowInsets：第一个子View。这是由ViewGroup如何分发WindowInsets决定的，它是通过遍历子节点，知道其中的一个子View消费了insets。如果第一个孩子（这里是fragment A）消费了这个insets，那么后续的孩子节点（这里是fragment B）不会得到它们，我们最终会遇到这种情况。

我们回到之前的步骤，但是这次新增了WindowInsets分发的步骤：

1. 转换被提交。
2. 因为我们正在进行退出过渡，视图A会被保持在这里，过渡动画在它上面执行。
3. 视图B被添加到容器View中，立即被设置为不可见状态。
4. **WindowInsets被分发。我们想让视图B（第1个子View）来获取它，但是视图A（第0个子View）先得到了。**
5. Fragment B的进入和Shared element enter转换动画开始。
6. 视图B被设置为可见状态。
7. 当Fragment A的退出转换完成后，视图A被从容器View中移除。

#### 解决方案

解决方案其实非常简单：我们只需要确保两个视图都可以接收到WindowInsets。

我是这样做的，在容器View中添加OnApplyWindowInsetsListener监听（这个例子中就是宿主Activity），它会手动分发任何insets到所有的子View，而不仅仅只是消耗insets。

```kotlin
fragment_container.setOnApplyWindowInsetsListener { view, insets ->
  var consumed = false

  (view as ViewGroup).forEach { child ->
    // Dispatch the insets to the child*
    val childResult = child.dispatchApplyWindowInsets(insets)
    // If the child consumed the insets, record it
    if (childResult.isConsumed) {
      consumed = true
    }
  }

  // If any of the children consumed the insets, return
  // an appropriate value*
  if (consumed) insets.consumeSystemWindowInsets() else insets
}
```

当我们照做时，两个fragment都会接收到WindowInsets，这才是我们在第一篇文章中看到的真实效果：

[](https://raw.githubusercontent.com/mzzdxt/AndroidNote/master/translation/imgs/pic_04_02.gif)

#### 重要部分：确保去request

我差点忘了写有关的一件小事。如果你在fragment中处理WindowInsets，隐式（通过使用AppBarLayout等等）或显式处理，你需要确保request一些insets。使用requestApplyInsets()方法很容易做到。

```kotlin
override fun onViewCreated(view: View, icicle: Bundle) {
  super.onViewCreated(view, savedInstanceState)
  // yadda, yadda

  ViewCompat.requestApplyInsets(view)
}
```

你必须要这么做，因为window会自动传递insets，如果整个视图层次结构的聚合提供ui可见性发生变化的时候。由于有时你提供的两个fragment提供了完全相同的值，因此聚合值不会改变，所以系统会忽略“更改”。

原文链接戳 [这里](https://chris.banes.dev/2018/03/01/window-insets-fragment-transitions/)。