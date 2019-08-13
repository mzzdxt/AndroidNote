## 02-WindowInsets - 布局的监听器

原文来自 [链接](https://chris.banes.dev/2019/04/12/insets-listeners-to-layouts/)。官方文档 [WindowInsets](https://developer.android.com/reference/android/view/WindowInsets)。

如果你已经观看过我的[Becoming a Master Window Fitter](https://chris.banes.dev/talks/2017/becoming-a-master-window-fitter-lon/)演讲，你会知道处理window insets是非常复杂的。最近，我一直在改进应用程序中的系统栏处理，使它们能够在状态栏和导航栏后面绘制。我想我已经提出了一些方法，可以使处理insets更加容易一些（希望如此）。

#### 在导航栏后面绘制

对于本文的剩余部分，我们将使用BottomNavigationView进行简单的示例，这个控件布局在屏幕的底部。它的实现非常简单：

```xml
<BottomNavigationView
    android:layout_height="wrap_content"
    android:layout_width="match_parent" />
```

![](https://raw.githubusercontent.com/mzzdxt/AndroidNote/master/proandroiddev.com/imgs/pic_02_01.png)

默认情况下，Activity的内容将在系统提供的UI（导航栏等）中进行布局，因此我们的视图与导航栏齐平。当我们的设计师希望应用程序在导航栏后面绘制。为此，我们将使用适当的标志调用setSystemUiVisibility():

```kotlin
rootView.systemUiVisibility = View.SYSTEM_UI_FLAG_LAYOUT_STABLE or View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
```

最后，我们将更新我们的主题，以便我们有一个带有黑色图标的半透明导航栏：

```xm
<style name="AppTheme" parent="Theme.MaterialComponents.Light">
    <!-- Set the navigation bar to 50% translucent white -->
    <item name="android:navigationBarColor">#80FFFFFF</item>
    <!-- Since the nav bar is white, we will use dark icons -->
    <item name="android:windowLightNavigationBar">true</item>
</style>
```

![](https://raw.githubusercontent.com/mzzdxt/AndroidNote/master/proandroiddev.com/imgs/pic_02_02.png)

如你所见，这只是我们开始要做的事情。由于Activity现在被布局到导航栏后面，我们的BottomNavigationView也是如此。这意味着用户无法点击导航栏上的任何条目。为了解决这个问题，我们需要处理系统调度的任何WindowInsets，并使用这些值对视图应用适当的padding或margin值。

#### 通过设置padding处理insets

处理WindowInsets的常用方式之一就是给view设置padding值，这样它们的内容就不会显示在系统UI的后面。为此，我们可以设置OnApplyWindowInsetsListener监听，它为视图添加必要的底布padding，确保它的内容不被遮挡（obscured）。

```kot
bottomNav.setOnApplyWindowInsetsListener { view, insets ->
    view.updatePadding(bottom = insets.systemWindowInsetBottom)
    insets
}
```

![](https://raw.githubusercontent.com/mzzdxt/AndroidNote/master/proandroiddev.com/imgs/pic_02_03.png)

我们现在已经正确的处理了底部系统WindowInsets。但是后来我们觉得在布局中添加一些填充，可能是出于审美（aesthetic）的原因：

```xm
<BottomNavigationView
    android:layout_height="wrap_content"
    android:layout_width="match_parent"
    android:paddingVertical="24dp" />
```

![](https://raw.githubusercontent.com/mzzdxt/AndroidNote/master/proandroiddev.com/imgs/pic_02_04.png)

不对呀！你能看到问题所在吗？我们从`onApplyWindowInsetsListener`中调用updatePadding()将会从布局中清除预期的底部padding。

让我们一起添加当前padding和inset：

```kot
bottomNav.setOnApplyWindowInsetsListener { view, insets ->
    view.updatePadding(
        bottom = view.paddingBottom + insets.systemWindowInsetsBottom
    )
    insets
}
```

我们现在有一个新问题。WindowInsets可以再视图的生命周期中随时调度。这意味着我们的新逻辑会在第一次运行时没问题，但是对于后续（subsequent）的每一次调度，我们会添加越来越多的底部padding。这并不是我们想要的效果。🤦‍♂️

![](https://raw.githubusercontent.com/mzzdxt/AndroidNote/master/proandroiddev.com/imgs/pic_02_05.png)

我想出的解决方案就是记录视图的初始padding值，然后再参考这些值。例如：

```ko
// Keep a record of the intended bottom padding of the view
val bottomNavBottomPadding = bottomNav.paddingBottom

bottomNav.setOnApplyWindowInsetsListener { view, insets ->
    // We've got some insets, set the bottom padding to be the
    // original value + the inset value
    view.updatePadding(
        bottom = bottomNavBottomPadding + insets.systemWindowInsetBottom
    )
    insets
}
```

![](https://raw.githubusercontent.com/mzzdxt/AndroidNote/master/proandroiddev.com/imgs/pic_02_06.png)

这种方式很好用，意味着我们从布局中保持填充的意图，我们仍然根据需要插入视图。保持每个padding值在对象级属性是非常混乱（messy）的，我们可以做的更好...🤔

#### doOnApplyWindowInsets

进入`doOnApplyWindowInsets`扩展方法。这是`setOnApplyWindowInsetsListener()`的包装器，它概括了上面的模式：

```kotlin
fun View.doOnApplyWindowInsets(f: (View, WindowInsets, InitialPadding) -> Unit) {
    // Create a snapshot of the view's padding state
    val initialPadding = recordInitialPaddingForView(this)
    // Set an actual OnApplyWindowInsetsListener which proxies to the given
    // lambda, also passing in the original padding state
    setOnApplyWindowInsetsListener { v, insets ->
        f(v, insets, initialPadding)
        // Always return the insets, so that children can also use them
        insets
    }
    // request some insets
    requestApplyInsetsWhenAttached()
}

data class InitialPadding(val left: Int, val top: Int, 
    val right: Int, val bottom: Int)

private fun recordInitialPaddingForView(view: View) = InitialPadding(
    view.paddingLeft, view.paddingTop, view.paddingRight, view.paddingBottom)
```

当我们需要一个View去处理insets时，可以参照下面的代码：

```ko
bottomNav.doOnApplyWindowInsets { view, insets, padding ->
    // padding contains the original padding values after inflation
    view.updatePadding(
        bottom = padding.bottom + insets.systemWindowInsetBottom
    )
}
```

#### requestApplyInsetsWhenAttached()

你可能注意到上面的`requestApplyInsetsWhenAttached()`方法。这不是绝对必要的，但确实可以解决WindowInsets的分发方式。如果View在未附加到视图层次时调用`requestApplyInsets()`，则会被自动忽略。

在Fragment的onCreateView()方法中创建视图，是一种常见的情况。修复方法是确保简单的再onStart()中调用方法，或者使用监听器在attached时请求insets。以下扩展函数处理两种情况:

```kot
fun View.requestApplyInsetsWhenAttached() {
    if (isAttachedToWindow) {
        // We're already attached, just request as normal
        requestApplyInsets()
    } else {
        // We're not attached to the hierarchy, add a listener to
        // request when we are
        addOnAttachStateChangeListener(object : OnAttachStateChangeListener {
            override fun onViewAttachedToWindow(v: View) {
                v.removeOnAttachStateChangeListener(this)
                v.requestApplyInsets()
            }

            override fun onViewDetachedFromWindow(v: View) = Unit
        })
    }
}
```

#### 在绑定视图中封装使用

在这一点上，我们已经大大的简化了如何处理WindowInsets。我们实际上在一些即将推出的应用程序中使用此功能，包括即将举行的会议apps。它仍然有一些缺点。首先，逻辑远离我们的布局，这意味着它很容易被遗忘。其次，我们可能会在很多地方使用它，导致许多几乎相同的副本在整个应用程序中传播。我知道我们可以做的更好。

到目前为止，整篇文章只关注代码，并通过设置监听器来处理insets。我们在这里讨论的是视图，所以在理想的世界中我们会声明我们打算在布局文件中处理insets。

输入数据适配器！如果你之前从来没有使用过它，他会让我们将代码映射到布局属性（当你使用data binding时）。因此，我们来创建一个属性：

```kot
@BindingAdapter("paddingBottomSystemWindowInsets")
fun applySystemWindowBottomInset(view: View, applyBottomInset: Boolean) {
    view.doOnApplyWindowInsets { view, insets, padding ->
        val bottom = if (applyBottomInset) insets.systemWindowInsetBottom else 0
        view.updatePadding(bottom = padding.bottom + insets.systemWindowInsetBottom)
    }
}
```

在我们的布局中，我们可以简单的使用新的paddingBottomSystemWindowInsets属性，该属性将使用任何insets自动更新。

```xm
<BottomNavigationView
    android:layout_height="wrap_content"
    android:layout_width="match_parent"
    android:paddingVertical="24dp"
    app:paddingBottomSystemWindowInsets="@{ true }" />
```

希望你能看到与单独使用OnApplyWindowListener相比，它是如何简单易用。

但等等~绑定适配器硬编码只设置底部尺寸。如果我们还需要处理顶部inset呢？或者左边？或者右边？幸运的是，绑定适配器让我们可以很好的概括所有维度的模式：

```ko
@BindingAdapter(
    "paddingLeftSystemWindowInsets",
    "paddingTopSystemWindowInsets",
    "paddingRightSystemWindowInsets",
    "paddingBottomSystemWindowInsets",
    requireAll = false
)
fun applySystemWindows(
    view: View,
    applyLeft: Boolean,
    applyTop: Boolean,
    applyRight: Boolean,
    applyBottom: Boolean
) {
    view.doOnApplyWindowInsets { view, insets, padding ->
        val left = if (applyLeft) insets.systemWindowInsetLeft else 0
        val top = if (applyTop) insets.systemWindowInsetTop else 0
        val right = if (applyRight) insets.systemWindowInsetRight else 0
        val bottom = if (applyBottom) insets.systemWindowInsetBottom else 0

        view.setPadding(
            padding.left + left,
            padding.top + top,
            padding.right + right,
            padding.bottom + bottom
        )
    }
}
```

这里我们已经声明了一个具有多个属性的适配器，每个属性都映射到相关的方法参数上。需要注意的是`requireAll = false`，意味着适配器可以处理所有设置属性的任意组合。这意味着我们可以执行以下操作，例如设置左侧和底部：

```xm
<BottomNavigationView
    android:layout_height="wrap_content"
    android:layout_width="match_parent"
    android:paddingVertical="24dp"
    app:paddingBottomSystemWindowInsets="@{ true }"
    app:paddingLeftSystemWindowInsets="@{ true }" />
```

易用度：100分

#### android:fitSystemWindows

你可能已经阅读过这篇文章，并想到“为什么没有提到fitSystemWindow属性？”。原因是属性带来的功能，通常不是我们想要的。

如果你正在使用AppBarLayout, CoordinatorLayout, DrawerLayout，那么按照指示使用。构建这些视图是为了识别属性，并以与这些视图相关的固定方式应用窗口插入。

android:fitSystemWindows的默认View实现意味着使用insets填充每个维度，但不适用于上面的示例。 有关更多信息，请参阅此博客文章仍然非常相关。

