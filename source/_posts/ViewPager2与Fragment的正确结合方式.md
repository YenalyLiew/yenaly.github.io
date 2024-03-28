---
title: ViewPager2 与 Fragment 的正确结合方式
---
# ViewPager2 与 Fragment 的正确结合方式

## 错误用法

`TabLayoutMediator`和`FragmentStateAdapter`，只要经常使用 TabLayout 和 ViewPager2 ，这两个东西你一定不会陌生。前者用来使 TabLayout 和 ViewPager2 进行结合，后者用来设置 ViewPager2 的适配器。

前者大家应该用的都没什么问题，只要不要忘记在最后调用`TabLayoutMediator#attach()`方法。

后者问题就比较多了，甚至在 StackOverflow 上也有不少像这样用错的。

举一个非常简单的例子，你需要给 ViewPager2 绑定几个以下需求的 Fragment ：

1. FirstFragment，需要给它传一个名为`name`的参数，值为`Ace Taffy`。
2. SecondFragment，需要给它传一个名为`position`的参数，值为`1`。
3. ThirdFragment，不需要参数。
4. ...

> 传值我为了方便，使用一个拓展函数
>
> `Fragment.makeBundle(vararg Pair<String, Any>): Fragment`
>
> 类似于自己在 Fragment 内部写的 static 函数 `XXFragment.newInstance(vararg Any)`

灵机一动，这不简单吗？先把它们放在 List 里

```kotlin
val fragmentList = listOf(
    FirstFragment().makeBundle("name" to "Ace Taffy"),
    SecondFragment().makeBundle("position" to 1),
    ThirdFragment()
    ...
)
```

然后构建一个 FragmentStateAdapter，设置 ViewPager2 的 adapter。这里就不单独给它弄一个 class 了

``` kotlin
viewPager2.adapter = object : FragmentStateAdapter(this) {
    override fun getItemCount() = fragmentList.size
    override fun createFragment(position: Int) = fragmentList[position]
}
```

这么寥寥几行，就完成了构建，看起来非常的简单，要是再想加几个 Fragment 也会很方便。但这样真的对吗？

## 错误分析

咱们先来分析一下 `FragmentStateAdapter#createFragment(int)`这个函数，看一看官方是怎么定义的

```java
/**
 * Provide a new Fragment associated with the specified position.
 * <p>
 * The adapter will be responsible for the Fragment lifecycle:
 * <ul>
 *     <li>The Fragment will be used to display an item.</li>
 *     <li>The Fragment will be destroyed when it gets too far from the viewport, and its state
 *     will be saved. When the item is close to the viewport again, a new Fragment will be
 *     requested, and a previously saved state will be used to initialize it.
 * </ul>
 * @see ViewPager2#setOffscreenPageLimit
 */
public abstract @NonNull Fragment createFragment(int position);
```

翻译一下，就是**在与其关联的位置上提供一个新的 Fragment**。这里这个 new 非常的显眼，它需要一个 Fragment，而且得是全新的。

再看一下这个抽象函数被哪里引用，可以发现全局只用在`ensureFragment(int)`这里

```java
private void ensureFragment(int position) {
    long itemId = getItemId(position);
    if (!mFragments.containsKey(itemId)) {
        // TODO(133419201): check if a Fragment provided here is a new Fragment
        Fragment newFragment = createFragment(position);
        newFragment.setInitialSavedState(mSavedStates.get(itemId));
        mFragments.put(itemId, newFragment);
    }
}
```

先不用看别的，这个 TODO 就很显眼，甚至官方都还没完美解决如何检测提供的 Fragment 是否为新的 Fragment 这个问题。

这个函数也很简单，如果`mFragment`（itemId 与 Fragment 关联起来的 LongSparseArray）中没有该页应该有的 Fragment，则通过之前说的 `createFragment(int)`回调获取 Fragment，然后通过 `Fragment#setInitialSavedState(Fragment.SavedState)`方法，从 `mSavedStates` （itemId 与 SavedState 关联起来的 LongSparseArray）中获取**该页**储存的 SavedState（没有就返回 null），最后把这个加工过的 Fragment 与 itemId 配对加入到`mFragment`里。

再去翻翻代码，还可以看到`gcFragments()`的函数，函数如其名，就是回收不必要的 Fragment，所以说 Fragment 是一个回收重建的过程，但你不一定感受的到，因为把 Fragment 的 SavedState 保存了，重建的时候恢复一下就可以了。

可以看到，它需要一个新的 Fragment，然后加工成一个我们真正所需的 Fragment。而在我们之前的错误用法中（如下），

```kotlin
val fragmentList = listOf(
    FirstFragment().makeBundle("name" to "Ace Taffy"),
    SecondFragment().makeBundle("position" to 1),
    ThirdFragment()
    ...
)
```

通过编写 list，已经提前给他实例化了，你只要获取到 list 中的元素，那就是引用，而不是深拷贝（而且Fragment 没有实现`Cloneable`接口，不支持深拷贝）。所以在这里（如下），

```kotlin
override fun createFragment(position: Int) = fragmentList[position]
```

它需要一个 new Fragment，而我们一直给它一个已经实例化的 Fragment 的引用，这就是错误所在。

举一个例子，假设你考一次试，这是你第一次见到这张试卷。但是过了一段时间，你的老师为了复习让你再一次做这张试卷。你要是把答案记住直接往上誊写，那会有很大的风险，只有抛弃之前的记忆，重新写一遍这张试卷，那才能避免各种不必要的风险。

一直给它一个已经实例化的 Fragment 还有什么问题？那就是资源浪费。假设你需要展示 100 个 Fragment，我放进 list，它一次性就给我实例化完了，我可能进软件都划不到 10 个，那剩余 90 个实例化还有什么意义呢？

## 正确使用

### 多 Fragment

> 这里暂时使用了魔法数字，如果你看着不爽也可以定义几个常量代表位置。

```kotlin
override fun createFragment(position: Int): Fragment {
    return when (position) {
        0 -> FirstFragment().makeBundle("name" to "Ace Taffy")
        1 -> SecondFragment().makeBundle("position" to 1)
        2 -> ThirdFragment()
        3 -> ...
        else -> Fragment()
    }
}

override fun getItemCount() = 4
```

这样就避免了资源浪费的问题，划到哪儿就实例化哪儿，而且每次都是全新的 Fragment。

### 单 Fragment

如果不带参数，那就太简单了，直接`return new XXFragment()`就完事了。

假设你有一个下载界面，需要展示`已下载`、`正在下载`、`失败下载`三个界面，因为它们三个的整体构造类似，所以由一个 Fragment 组成，但是需要分别传`downloaded`、`downloading`、`failed`三个参数才能触发到相应的请求。

根据上面的错误，我们可以写出这样的代码

> `DownloadFragment#newInstance(String): DownloadFragment`是在 DownloadFragment 中定义好的静态函数，方便传参实例化。与上文的`Fragment.makeBundle(vararg Pair<String, Any>): Fragment`效果一致。

```kotlin
override fun createFragment(position: Int): Fragment {
    return when (position) {
        0 -> DownloadFragment.newInstance("downloaded")
        1 -> DownloadFragment.newInstance("downloading")
        2 -> DownloadFragment.newInstance("failed")
        else -> Fragment()
    }
}

override fun getItemCount() = 3
```

这样写确实没问题，但也可以这样，从 DownloadFragment 中添加静态 List

```kotlin
companion object {
    val typeList = listOf("downloaded", "downloading", "failed")
}
```

然后直接

```kotlin
override fun createFragment(position: Int) =
	DownloadFragment.newInstance(DownloadFragment.typeList[position])

override fun getItemCount() = DownloadFragment.typeList.size
```

这样想添加新的界面也很简单，往`typeList`里加新的 type 就完事了，不用再改这改那了。

### 另辟蹊径

利用 Kotlin 高阶函数特性（Java 的单方法接口也可以），返回的都是 new Fragment，可以想到这种方法来构建

```kotlin
typealias HandleFragment = () -> Fragment

open class SimpleViewPagerAdapter(
    fragmentManager: FragmentManager,
    lifecycle: Lifecycle
) : FragmentStateAdapter(fragmentManager, lifecycle) {
    
    private val mFragmentList = mutableListOf<HandleFragment>()

    override fun getItemCount(): Int {
        return mFragmentList.size
    }

    override fun createFragment(position: Int): Fragment {
        return mFragmentList[position].invoke()
    }

    fun add(fragment: HandleFragment): SimpleViewPagerAdapter {
        mFragmentList.add(fragment)
        return this
    }

    fun add(fragmentList: List<HandleFragment>): SimpleViewPagerAdapter {
        mFragmentList.addAll(fragmentList)
        return this
    }
}
```

使用起来也很方便

```kotlin
vp.adapter = SimpleViewPagerAdapter(childFragmentManager, lifecycle).apply {
    add { FirstFragment().makeBundle("name" to "Ace Taffy") }
    add { SecondFragment().makeBundle("position" to 1) }
    add { ThirdFragment() }
}
```

这样更加简化，修改位置不用修改数字了，改动一下 add 的相对位置就可以了。

## 总结

要是想把 position 那个参数和 list 结合起来，最多把传参组成个 list，然后 new Fragment 的时候把传参放进去。**切忌 list 里放实例化的 Fragment 传给 FragmentStateAdapter！**你既不能深拷贝 Fragment，又会造成资源浪费。

聪明的你非要把 Fragment 存个 list，然后说这样不就保证每次都为 new 了吗？（如下）

```kotlin
override fun createFragment(position: Int) = fragmentList[position].javaClass.newInstance()
```

呃呃，实例化完了还要反射实例化，这多重性能开销我就不说了。而且利用反射的`newInstance()`根本不好进行参数传递。

聪明的你又想到，我虽然不能深拷贝 Fragment，但我可以在`createFragment(int)`里深拷贝`List<Fragment>`啊！那你更逆天了，这样的话每调用一次`createFragment(int)`就会把 list 里的所有元素深拷贝一遍，这样确实能保证它时刻是 new 的了，但你有没有觉得这样做很搞笑呢？