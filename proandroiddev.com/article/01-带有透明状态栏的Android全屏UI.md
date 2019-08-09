### 01-带有透明状态栏的Android全屏UI

Activity是构建任何Android应用的基石。有时它用起来很简单，不过有时又很复杂。在这里，我们将要讨论一些和Activity相关的问题，这些问题一开始看上去很简单，但是又很快变得复杂。我们将要构建一个全屏布局并且带有透明状态栏的Activity。我不会在这里讨论为什么你需要一个全屏布局，以及在什么情境下使用它。那个主题需要在另外一个情境下再去讨论。

然而，这里有一个简单的用例。如果你曾经见过任何地图应用（比如一款骑行app），你将会看到地图控件会占据状态栏的下方。除了地图之外的内容布局并不会和状态栏里的图标重叠显示。这样看起来不是很好吗？所以，我们准备重建那样的UI效果。如下图所示：

![pic_01_01](https://raw.githubusercontent.com/mzzdxt/AndroidNote/master/proandroiddev.com/imgs/pic_01_01.jpeg)

![](https://raw.githubusercontent.com/mzzdxt/AndroidNote/master/proandroiddev.com/imgs/pic_01_02.jpeg)

#### 为Activity设置一个主题

我们开始最基本的操作（我假设你已经创建了一个带有map控件和一些内容的Activity），然后为我们的activity设置一个主题：

```xml
<style name="CustomTheme" parent="Theme.AppCompat.Light.NoActionBar"/>
```

然后和平时一样，把这个主题应用到activity上：

```xml
<activity
		android:name="com.my.app.CustomActivity"
    android:theme="@style/CustomTheme">
</activity>
```

非常简单！！让我们看看它是什么样子。

#### 设置UI全屏

现在，让我们看看有趣的部分。如果让布局全屏展示并且设置状态栏的颜色呢？

我们开始编码吧！把下面的代码运行到你的设备上，然后我们再去理解实际上发生了什么：

以下代码针对Android 5.0以上的设备：

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    this.window.apply {
        clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS)
        addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS)
        decorView.systemUiVisibility = View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
        statusBarColor = Color.TRANSPARENT
    }
}
```

#### 什么是Window？

当你打开一个标准的app时，你会看的状态栏、导航栏和真正的activity。它们这些组件都有不同的window。每个组件都会被赋予一个window来绘制它们自己。对于activity来说，它被赋予的window用来绘制我们指定的视图层次结构。对于状态栏来说，它被赋予的window用来绘制系统指定的视图，比如时间、电池状态、所有的通知图标等等。对于导航栏来说，它同样拥有一个window用来绘制返回按钮、home按钮等等。所有的这些window在同一个屏幕上被WindowManager来统一管理。

#### 针对window应用不同的flags有什么区别？

如果**FLAG_TRANSLUCENT_STATUS**被应用，将会显示出半透明的状态栏，这不是我们想要的效果。所以我们需要首先移除这个flag。

使用**FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS**标志意味着我们会告知系统，我们自己的窗口将会负责绘制这些系统bar的背景。

#### 什么是systemUiVisibility？

使用这个方法，我们可以控制那些被系统绘制UI元素的可见性。系统UI元素指的是状态栏、导航栏。在进入和退出全屏模式时，系统栏将会隐藏和显示时，使用**SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN**标志有助于防止内容调整大小。

我想*setStatusBarColor()*就无需解释了吧。然而，这个API只在API 21以上才可用。

#### 在Android 6.0以上的设备上出现的问题：

系统栏的图标都会变成白色，如果你布局的颜色也是浅色，这样看起来并不太好。

##### 如何解决这个问题？

同样，使用*setSystemUiVisibility*来解决吧。

使用**SYSTEM_UI_FLAG_LIGHT_STATUS_BAR**标志，用于浅色主题下，来确保状态栏图标完全可见。

```kotlin
window.apply {
		decorView.systemUiVisibility = View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN or View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR
}
```

Coll...现在，在大多数API以上的设备上，我们的状态栏看起来非常完美。但是当出现Floating Action Button会看起来怎么样呢？这个家伙会和系统栏的图标重叠，这让我看起来非常不爽。接下来，我们解决它吧：

#### 转移内容

你app布局的实际内容应该往下移，那样才不会和状态栏的图标重叠在一起。OK，我们该怎么做才能让它往下移呢？给它设置一个marginTop值可以吗？

但是，这样会出现各种各样的问题。我们怎么知道应该设置多大的marginTop值给内容视图呢？有很多种方式可以做到这一点，但我将采用最明确的方法，这种方式在各种设备上都会生效。

**Window insets to the rescue...**

#### 什么是window insets？

Window insets会提供给我们需要的系统View的尺寸。我们这里讨论的系统View指的就是状态栏。所以，顶部的window insets会给我们提供我们内容到顶部的边距。

##### 如何获取当前可见window的insets尺寸？

```kot
ViewCompat.setOnApplyWindowInsetsListener(
    findViewById(R.id.map)
) { _, insets ->
    Log.d(TAG, "topInset -> ${insets.systemWindowInsetTop}")
    Log.d(TAG, "bottomInset -> ${insets.systemWindowInsetBottom}")
    //and so on for left and right insets
    insets.consumeSystemWindowInsets()
}
```

现在，它只是获取视图并将marginTop设置为等于窗口的topInset：

```ko
ViewCompat.setOnApplyWindowInsetsListener(
    findViewById(R.id.map)
) { _, insets ->
    Log.d(TAG, "topInset -> ${insets.systemWindowInsetTop}")
    Log.d(TAG, "bottomInset -> ${insets.systemWindowInsetBottom}")

    val layoutParams = btn.layoutParams as FrameLayout.LayoutParams
    layoutParams.setMargins(0, insets.systemWindowInsetTop, 0, 0)
    btn.layoutParams = layoutParams
    //and so on for left and right insets
    insets.consumeSystemWindowInsets()
}
```

#### 扩展以便重用

通常在一个大型应用中，像这样的代码经常会被重复使用。所以，创建一个具有帮助性的kotlin扩展，用于任何activity从而避免代码重复，岂不是更好吗？

```kotlin
fun Activity.makeStatusBarTransparent() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        window.apply {
            clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS)
            addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS)
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                decorView.systemUiVisibility =
                    View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN or View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR
            } else {
                decorView.systemUiVisibility = View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
            }
            statusBarColor = Color.TRANSPARENT
        }
    }
}

fun View.setMarginTop(marginTop: Int) {
    val menuLayoutParams = this.layoutParams as ViewGroup.MarginLayoutParams
    menuLayoutParams.setMargins(0, marginTop, 0, 0)
    this.layoutParams = menuLayoutParams
}
```

以后，我们只需要在任何activity中使用如下代码：

```kot
makeStatusBarTransparent()
ViewCompat.setOnApplyWindowInsetsListener(findViewById(R.id.content_container)) { _, insets ->
    findViewById<FloatingActionButton>(R.id.fab1).setMarginTop(insets.systemWindowInsetTop)
    findViewById<FloatingActionButton>(R.id.fab2).setMarginTop(insets.systemWindowInsetTop)
    insets.consumeSystemWindowInsets()
}
```

如果你想了解更多关于Window insets的知识，请访问 [Window insets](https://chris.banes.dev/talks/2017/becoming-a-master-window-fitter-lon/ )。