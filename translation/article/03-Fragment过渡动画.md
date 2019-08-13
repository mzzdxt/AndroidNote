### 03-Fragment过渡动画

这是一篇短篇系列的第一篇文章，在这篇文章中，我将探讨如何使fragments直接的过渡更加完美。这篇文章是关于如何来实现的。

几个月前，在我构建的名字叫Tivi的应用上，我展示过一个网格到网格的变换效果。

在此之前，我对过渡动画的使用非常困惑。在应用程序中，我使用fragment来进行无边界过渡，就像展开数据和底部导航切换。当数据的种类发生变化时（比如，打开单个item的详细信息时），我开始使用Activity来代替。

你在上面看到的就是fragment变换。Fragment A是概述界面，当用户点击了“更多”按钮时，它将会使用包含完整网格数据的Fragment B来代替显示。

我现有的Fragment事务代码如下所示：

```kotlin
supportFragmentManager.beginTransaction()  
    .replace(R.id.home_content, fragment)
    .addToBackStack(null)
    .commit()
```

并没有什么值得兴奋的地方。

所以我查找了转换API并添加了一些shared elements。在这种情况下，shared elements是包含海报的方形图片：

```kotlin
supportFragmentManager.beginTransaction()  
    .replace(R.id.home_content, fragment)
    .addToBackStack(null)
    .apply {
      for (view in sharedElements) {
        addSharedElement(view)
      }
    }
    .commit()
```

然后我在要进入的Fragment（也就是Fragment B）设置了shared element转换。在这种情况下，我使用了ChangedBounds()来开始，因为这些View刚刚从A点移动到了B点，并且改变了大小。

```ko
class GridFragment : Fragment() {
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    sharedElementEnterTransition = ChangeBounds()
  }
}
```

在这一点上，我非常乐观的认为它会生效，但是我实际上看到的是这样：

[]()

所以它在进入的时候根本没有生效，但是在返回的时候过渡效果很好。这只是一个开始。

#### 推迟

在这一点上，我想起来我的同事Nick Butcher过去为Plaid实现过渡效果时层像我提到的一些事情。特别是必须延迟转换，知道你的视图准备就绪（布局、数据加载等等）。

所以，我按照他说的添加了延迟以后，在开始我的Fragments：

```kot
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
  // View is created so postpone the transition
  postponeEnterTransition()

  viewModel.liveList.observe(this) {
    controller.setList(it)

    // Data is loaded so we're ready to start our transition
    startPostponedEnterTransition()
  }
}
```

我们为进入和离开的fragments执行了此操作，这样进入和退出就会按照我们预期的效果来执行。

但是，依旧没有进入动画。🤦‍♂️

#### 重新排序

我有点懵，然后联系了Android团队负责Activity-Fragment-Transitions开发的George Mount。他指出我需要做的是：重新排序。

事实证明，你必须为延迟Fragment转换开启重新排序事务，才能正常工作。幸运的是，这很简单（但是容易忘记）：

```kot
supportFragmentManager.beginTransaction()
    .setReorderingAllowed(true)
    .replace(R.id.home_content, fragment)
    .addToBackStack(null)
    .apply {
      for (view in sharedElements) {
        addSharedElement(view)
      }
    }
    .commit()
```

在此之后，进入转换动画偶尔会起作用，但是大多数时候我看到的时候交叉淡入淡出效果。至少现在有些时候，会出现过渡效果了。

#### 等待父布局

由于我的转换效果偶尔生效了，我开始调试。我推断（deduced）在执行转换效果的时候，已经布局了视图（isLaidOut==true），并且已经绘制。

所以，最后一步就是...等待。

如果你回顾我们调用延迟代码的地方，我们实际上是在数据加载后立即开始了推迟转换。在我们开始转换之前，需要给视图一些时间用来更新，布局以及更重要的绘制操作。

所以我们的延迟转换调用就变成了：

```kotl
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
  // View is created so postpone the transition
  postponeEnterTransition()

  viewModel.liveList.observe(this) {
    controller.setList(it)

    // Data is loaded so lets wait for our parent to be drawn
    (view?.parent as? ViewGroup)?.doOnPreDraw {
      // Parent has been drawn. Start transitioning!
      startPostponedEnterTransition()
    }
  }
}
```

你可能会想，为什么我们要在父布局设置OnPreDrawListener，而不是视图本身。那是应为你的视图可能并没有真正的被绘制出来，因此监听可能永远不会生效，事务也可能永远被终止在那里。为了解决这个问题，我们在父布局设置监听，它（可能）会被绘制。

瞧！我们有了过渡效果：

你可能会想为什么这个效果看起来和tweet上的效果不一样。我会在后面的文章中进行介绍。下一篇文章将介绍如何获取WindowInsets和Fragment转换一起生效。