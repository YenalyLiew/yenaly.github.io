---
title: 小白如何快速实现简单的可保存状态断点续传后台下载？一个 Jetpack 库搞定一切！
description: 部分图片显示不出来，需要的可以点击链接新开一个页面查看，因为有防盗链
---
# 小白如何快速实现简单的可保存状态断点续传后台下载？一个 Jetpack 库搞定一切！

整篇文章一个 interface 回调都没有，还告诉我能传进度？你别说还真能！

## 前言

对小白来说，实现下载功能确实令人头大，别说什么能断点续传的下载，断点续传能保存状态的后台下载。初来乍到的我们首先想到的就是去各大编程交流网站去查询怎么实现，比如 C 站，一看发稿日期 2013 年、2015 年，一看用的技术，AsyncTask、Thread 等等，全是过时的，没有一个用了较新的技术，而且各种 interface，BroadcastReceiver 回调，一想到再和 RecyclerView 搭配，更无所适从了。来掘金看看，也基本上是推荐自己框架的，其实也不能很系统、简单的教你从零写一个马上能用的。很多人也是因此被劝退了。

首先非常感谢这篇文章给了我清晰的思路 [Android原生下载（上篇）基本逻辑+断点续传 - 掘金](https://juejin.cn/post/6844903712440205325#heading-11)，但还是有点晦涩难懂，再加上作者丰富的图更会让人无所适从。所以我以此摸索出了一套适合小白快速用 **Room 数据库** 搭建的方法。

本文用 Flow 比较多，当然你用 RxJava，LiveData 也可以，只是 Flow 我用惯了，思路一致。

## 目的

这篇文章会手搓一个我自己摸索出来的简单又好用的**单线程 | 断点续传 | 可后台 | 可保存状态**的**下载功能**，而不是专业的下载器。效率不一定高，但是绝对容易理解、上手，而且有基本完善的功能，适合快速整合进个人开发的 APP 中。

## 需求

写这个之前，我们需要理清我们要干什么。

1. 文件添加后可以断点续传下载，可以暂停、继续、取消。
2. 分为“已下载”和“正在下载”两个区域，下载完成之后马上进入“已下载”区域。
3. “正在下载”区域显示下载进度、速度。
4. “已下载”区域可以删除文件。
5. 可以后台下载。从另外一个 Activity 下载东西后，开始在后台下载，即使不在下载界面。

## 使用的 Jetpack 库

我们这里就用两个 Jetpack 库，甚至不用都可以。为了快速搭建，至少还是用个 Room 数据库，所以标题才写了**一个** Jetpack 库搞定一切。

1. Room（数据库，可选，但建议选择）
2. WorkManager（类似 Service，提供后台功能，可选）

为什么说不用都可以，首先 WorkManager 就是大号 Service，完全可以替换，只不过 WorkManager 支持协程，所以写起来方便。然后 Room 就是普通数据库，你非得用自带的也可以，不过那会麻烦很多，因为我们的很多操作需要 Room 的“魔法”操作。

## 使用的普通库

1. RecyclerView（负责列表显示）
2. Flow（负责协程，可选）

其中 RecyclerView 最好用 DiffUtil，实现更好更高效的动画效果。

## 开始前必了解

### 为什么要 RandomAccessFile

让 ChatGPT 写了下关于 `RandomAccessFile` 的一些重要特性和它为什么适合断点续传的原因：

`RandomAccessFile` 是 Java 中用于读写文件的类，它支持随机访问文件的读取和写入操作。与传统的输入流和输出流不同，`RandomAccessFile` 允许你在文件中移动到任意位置并读取或写入数据，这使它非常适合实现断点续传功能。

1. 随机访问：`RandomAccessFile` 支持随机访问文件，这意味着你可以根据需要定位到文件的任意位置，而不必顺序读取或写入整个文件。
2. 读写分离：`RandomAccessFile` 具有独立的读和写方法，你可以单独执行文件的读取和写入操作。这使得在续传时可以轻松实现文件的部分读取和写入。
3. seek 方法：`RandomAccessFile` 提供了 `seek(long position)` 方法，该方法允许你将文件指针移动到指定的位置。通过结合 `seek` 方法和文件的当前大小，你可以轻松地确定从哪里开始续传。
4. 文件截断：`RandomAccessFile` 还提供了 `setLength(long newLength)` 方法，可以用于截断文件的大小。这在续传时非常有用，因为你可以根据已下载的部分来更新文件的大小，然后继续写入剩余的数据。

### 怎么向网站请求断点续传

先看看支不支持断点续传，响应头要是包含`Accept-Ranges`字段且值为`bytes`，说明支持范围请求。

然后给请求中加个 header，字段为`Range`，值为`bytes=%1d-%2d`，这表示请求文件的字节范围为 %1d ~ %2d，其中 %2d 可选，省略表示最后一字节。

举例：`bytes=100-`代表从第100个字节开始传输，一直到结尾。

还有如果范围请求成功，响应代码为 **206**。

在这篇文章中，假定下载的东西都是支持范围请求的。

### Room 到底有什么“魔法”

Room 数据库能返回 Flow，是因为内部使用了 SQLite 的内容更新通知功能，当数据库中的任何表有变化时，会触发一个回调，然后重新执行查询语句，并通过 Flow 发射最新的数据。这样 Flow 的观察者就可以收到最新数据，并得到相应更新。

我们就用的这个特性，将压力给到数据库上，从而简化我们人力写 interface 等等回调的东西。

### RecyclerView 为何搭配 DiffUtil

DiffUtil 是用于计算两个列表之间差异的工具类，可以帮助我们减少不必要的更新操作。

因为我们 Flow 每次返回的是一整个全新列表，手动去处理增加删除是很麻烦的，所以搭配 DiffUtil，可以在后台线程中自动帮你处理每个 item 的变化。

## 主要文件结构梳理

> 正常为普通文件，粗体为重要文件。

- **database** - 数据库
  - **DownloadDao.kt** - 负责 Dao 层，抽象方法集合
  - **DownloadDatabase.kt** - 数据库层，负责实例化数据库
  - **DownloadEntity.kt** - 数据实体类
- **ui** - 界面
  - **DownloadedFragment.kt** - 下载中 Fragment
  - **DownloadedRvAdapter.kt** - 下载中 Fragment 中 RecyclerView 的 Adapter
  - **DownloadingFragment.kt** - 已下载 Fragment
  - **DownloadingRvAdapter.kt** - 已下载 Fragment 中 RecyclerView 的 Adapter
  - MainActivity.kt - 下载相关所处的 Activity
- **worker** - WorkManager
  - **DownloadWorker.kt** - 负责下载的 Worker
- App.kt - 负责提供全局 Context
- Utils.kt - 工具函数

## 开搞！

> 注意：我为了操作简便，使用了封装好的 RecyclerView 库 [BaseRecyclerViewAdapterHelper](https://github.com/CymChad/BaseRecyclerViewAdapterHelper)。
>
> 跟普通 RecyclerView 使用也没太大区别，网上查查基本就会了。

### 显而易见的

别忘了在 manifest 里添加网络权限！

先把工具函数给大家看看，防止之后出现的函数可能看不懂。

```kotlin
// Utils.kt

package com.example.easydownload

import android.app.Activity
import android.content.Context
import android.os.Environment
import android.view.View
import androidx.annotation.IdRes
import androidx.appcompat.app.AppCompatActivity
import androidx.fragment.app.Fragment
import androidx.lifecycle.lifecycleScope
import kotlinx.coroutines.launch

fun <T : View> Activity.id(@IdRes id: Int): Lazy<T> {
    return lazy(LazyThreadSafetyMode.NONE) { findViewById(id) }
}

fun <T : View> Fragment.id(@IdRes id: Int): Lazy<T> {
    return lazy(LazyThreadSafetyMode.NONE) { requireView().findViewById(id) }
}

val downloadDir get() = App.context.getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS)

fun Context.suspendLaunch(block: suspend () -> Unit) =
    (this as? AppCompatActivity)?.lifecycleScope?.launch {
        block()
    }
```

还有 App 类，就是单纯获取一个全局 context 的，没啥别的用，图一个方便。

```kotlin
package com.example.easydownload

import android.annotation.SuppressLint
import android.app.Application
import android.content.Context

/**
 * @project EasyDownload
 * @author Yenaly Liew
 * @time 2023/09/12 012 11:51
 */
class App : Application() {
    companion object {
        @SuppressLint("StaticFieldLeak")
        lateinit var context: Context
    }

    override fun onCreate() {
        super.onCreate()
        context = applicationContext
    }
}
```

### 数据库构建

#### DownloadEntity

首先肯定是创建实体类：

```kotlin
package com.example.easydownload.database

import androidx.annotation.IntRange
import androidx.room.Entity
import androidx.room.PrimaryKey

/**
 * @project EasyDownload
 * @author Yenaly Liew
 * @time 2023/09/12 012 11:22
 */
@Entity
data class DownloadEntity(
    /**
     * 文件名称
     */
    var name: String,
    /**
     * 添加日期
     */
    var addDate: Long,
    /**
     * 存储在本地的位置
     */
    var uri: String,

    /**
     * 下载地址
     */
    var url: String,
    /**
     * 长度
     */
    var length: Long,
    /**
     * 已下载长度
     */
    var downloadedLength: Long,
    /**
     * 速度
     */
    // 速度[kb/s] = (当前下载的长度[b] / 1000)[kb] / (时间间隔[ms] / 1000)[s]
    var speed: Float,
    /**
     * 是否正在下载
     */
    var isDownloading: Boolean = false,

    @PrimaryKey(autoGenerate = true)
    var id: Int = 0,
) {
    /**
     * 下载进度
     */
    @get:IntRange(from = 0, to = 100)
    val progress get() = (downloadedLength * 100 / length).toInt()

    /**
     * 是否已下载完成
     */
    val isDownloaded get() = downloadedLength == length
}
```

实现**可保存状态**的字段就是其中的`length`和`downloadedLength`，`length`是固定值，而每次下载都会用`downloadedLength`记录文件下载的长度，下次下载可以通过`downloadedLength`来获取起始下载位置。

#### DownloadDao

然后就是数据库抽象方法类：

```kotlin
package com.example.easydownload.database

import androidx.room.Dao
import androidx.room.Delete
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query
import androidx.room.Transaction
import androidx.room.Update
import kotlinx.coroutines.flow.Flow

/**
 * @project EasyDownload
 * @author Yenaly Liew
 * @time 2023/09/12 012 11:31
 */
@Dao
abstract class DownloadDao {
    /**
     * 查询所有下载中的任务
     */
    @Query("SELECT * FROM DownloadEntity WHERE downloadedLength <> length ORDER BY id DESC")
    abstract fun loadAllDownloading(): Flow<MutableList<DownloadEntity>>

    /**
     * 查询所有已下载完成的任务
     */
    @Query("SELECT * FROM DownloadEntity WHERE downloadedLength == length ORDER BY id DESC")
    abstract fun loadAllDownloaded(): Flow<MutableList<DownloadEntity>>

    @Query("UPDATE DownloadEntity SET `isDownloading` = 0")
    abstract suspend fun pauseAll()

    @Delete
    abstract suspend fun delete(entity: DownloadEntity)

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    abstract suspend fun insert(entity: DownloadEntity)

    @Update(onConflict = OnConflictStrategy.REPLACE)
    abstract suspend fun update(entity: DownloadEntity): Int

    /**
     * 根据url查询
     */
    @Query("SELECT * FROM DownloadEntity WHERE (`url` = :url) LIMIT 1")
    abstract suspend fun findBy(url: String): DownloadEntity?

    /**
     * 根据url查询是否存在
     */
    @Transaction
    open suspend fun isExist(url: String): Boolean {
        return findBy(url) != null
    }
}
```

关键的关键就是前两个方法，那前两个方法分别是什么意思呢？

第一个：

```sql
SELECT * FROM DownloadEntity WHERE downloadedLength <> length ORDER BY id DESC
```

从 DownloadEntity 表中选择 已下载长度 **不等于** 总长度 的数据，代表没下完，然后根据 id 降序。

第二个：

```sql
SELECT * FROM DownloadEntity WHERE downloadedLength == length ORDER BY id DESC
```

从 DownloadEntity 表中选择 已下载长度 **等于** 总长度 的数据，代表下完了，然后根据 id 降序。

而且注意，他们俩返回的是`Flow<MutableList<DownloadEntity>>`，而不是别的。这个`Flow`可不一般，表修改后，可以实时返回整个表的内容，非常适合用来回调。类似能提供这种功能的也有 `LiveData`、RxJava 的`Observable`等等，想用哪个都可以。

还有注意如果是非抽象方法千万别忘了加 open 关键字！

本文暂时用下载链接 URL 作为查询依据，其实不是很恰当，但是作为演示够了。如果你有需要可以修改。

#### DownloadDatabase

```kotlin
package com.example.easydownload.database

import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase
import com.example.easydownload.App

/**
 * @project EasyDownload
 * @author Yenaly Liew
 * @time 2023/09/12 012 11:45
 */
@Database(entities = [DownloadEntity::class], version = 1)
abstract class DownloadDatabase : RoomDatabase() {

    abstract val downloadDao: DownloadDao

    companion object {
        val instance by lazy {
            Room.databaseBuilder(
                App.context,
                DownloadDatabase::class.java,
                "download.db"
            ).build()
        }
    }
}
```

Room 数据库基本功，这里图省事直接传的全局 context，这个类看不懂的得去看看 Room 手册了。

这就已经完成三个类了，休息一下，马上进入 WorkManager 相关类的编写！

### 下载任务(服务)构建

这里可以说很关键。

#### DownloadWorker

##### 思路

在编写之前需要理清一下思路，应该怎么做？

首先我们需要接收参数，分别为`name`（文件名）、`downloadUrl`（下载地址）和`immediate`（是否立即下载）。

然后是下载初始操作。在下载之前，我们先判断数据库中是否有存在该 url 的实体。若没有，则对下载地址进行请求，得到它的`contentLength`，并创建等大小的，以`name`为名的文件，将其加入到数据库中保存，若`immediate`为`true`，则字段`isDownloading`设置为`true`，立即进行下载；反之为`false`，延迟下载。若存在该数据，则根据其中的字段`downloadedLength`进行 range 请求下载进行。

其次是下载进行操作。在下载中，每隔特定的一段时间，计算出当前的`downloadedLength`和`speed`，并对数据库进行更新。如果中途中断，在中断前再次更新数据库，并把实体的字段`isDownloading`设置为`false`。

##### 代码

接下来是代码展示，我会在一些重要部分写一些注释：

```kotlin
package com.example.easydownload.worker

import android.content.Context
import android.util.Log
import androidx.core.net.toUri
import androidx.work.Constraints
import androidx.work.CoroutineWorker
import androidx.work.ExistingWorkPolicy
import androidx.work.NetworkType
import androidx.work.OneTimeWorkRequestBuilder
import androidx.work.WorkManager
import androidx.work.WorkerParameters
import androidx.work.workDataOf
import com.example.easydownload.App
import com.example.easydownload.database.DownloadDatabase
import com.example.easydownload.database.DownloadEntity
import com.example.easydownload.downloadDir
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.Response
import okhttp3.ResponseBody
import java.io.File
import java.io.RandomAccessFile
import java.util.concurrent.CancellationException

fun enqueueDownloadWorker(name: String, url: String, immediate: Boolean) {
    val constraints = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED) // 只有联网才行，否则 cancel
        .build()
    val data = workDataOf(
        "name" to name,
        "download_url" to url,
        "download_imm" to immediate
    )
    val downloadRequest = OneTimeWorkRequestBuilder<DownloadWorker>()
        .addTag(DownloadWorker.TAG) // 创建的下载 Worker 都有这个 TAG，方便管理 
        .setConstraints(constraints)
        .setInputData(data)
        .build()
    WorkManager.getInstance(App.context)
    	// 以 url 作为唯一凭据，选择别的也可以，最好是 id 之类的，这里就图一个方便
    	// 这里选择的是，若重复会被覆盖
        .beginUniqueWork(url, ExistingWorkPolicy.REPLACE, downloadRequest)
        .enqueue()
}

/**
 * @project EasyDownload
 * @author Yenaly Liew
 * @time 2023/09/12 012 12:00
 */
class DownloadWorker(context: Context, params: WorkerParameters) :
    CoroutineWorker(context, params) {

    companion object {
        const val TAG = "download_worker"
        // 间隔设置为 500ms
        const val RESPONSE_INTERVAL = 500L
    }

    // 创建一个 OkHttpClient 负责网络请求
    // 也可以换成你自己的，这里也是图方便
    private val okHttpClient = OkHttpClient()

    private val name by lazy(LazyThreadSafetyMode.NONE) {
        inputData.getString("name") ?: ""
    }
    private val downloadUrl by lazy(LazyThreadSafetyMode.NONE) {
        inputData.getString("download_url") ?: ""
    }
    private val immediate by lazy(LazyThreadSafetyMode.NONE) {
        inputData.getBoolean("download_imm", true)
    }

    override suspend fun doWork(): Result {
        // 调用 Result.retry() 重试之后就会来这里
        // 若重试次数大于2，则宣告失败
        if (runAttemptCount > 2) {
            return Result.failure(workDataOf("failed_reason" to "下载失败三次"))
        }
        // 下载逻辑
        return download()
    }

    private suspend fun createNewRandomAccessFile(): Boolean = withContext(Dispatchers.IO) {
        val file = File(downloadDir, name)
        var raf: RandomAccessFile? = null
        var resp: Response? = null
        var body: ResponseBody? = null
        try {
            // 根据 file 创建 RAF，"rwd" 差不多是可读写的意思。
            raf = RandomAccessFile(file, "rwd")
            val req = Request.Builder().url(downloadUrl).get().build()
            resp = okHttpClient.newCall(req).await()
            if (resp.isSuccessful) {
                body = resp.body
                body?.let {
                    // 获取 content length
                    val len = body.contentLength()
                    if (len > 0) {
                        // 将 RAF 长度设置为 content length
                        raf.setLength(len)
                        // 实体初始化
                        val entity = DownloadEntity(
                            name = name,
                            addDate = System.currentTimeMillis(),
                            uri = file.toUri().toString(),
                            url = downloadUrl,
                            length = len,
                            downloadedLength = 0,
                            speed = 0F,
                            isDownloading = false
                        )
                        // 保存到数据库中。
                        DownloadDatabase.instance.downloadDao.insert(entity)
                        return@withContext true
                    }
                }
            }
            // 创建失败
            return@withContext false
        } catch (e: Exception) {
            e.printStackTrace()
            // 若文件创建了但是没读取上 content length，则删除
            if (file.length() == 0L) file.delete()
            // 创建失败
            return@withContext false
        } finally {
            raf?.close()
            resp?.close()
            body?.close()
        }
    }

    private suspend fun download() = withContext(Dispatchers.IO) {
        val file = File(downloadDir, name)
        val isExist = DownloadDatabase.instance.downloadDao.isExist(downloadUrl)
        // 不存在则创建，创建失败重试，创建成功但是延迟下载直接返回成功，不进行下一步
        if (!isExist) {
            val isCreated = createNewRandomAccessFile()
            if (!isCreated) {
                return@withContext Result.retry()
            } else if (!immediate) {
                return@withContext Result.success()
            }
        }
        // 重新搜索，看一看有没有添加上，若没有继续重试
        val entity = DownloadDatabase.instance.downloadDao.findBy(downloadUrl)
            ?: return@withContext Result.retry()

        // 判断是否需要加 Range 头
        val needRange = entity.downloadedLength > 0
        var raf: RandomAccessFile? = null
        var response: Response? = null
        var body: ResponseBody? = null
        try {
            raf = RandomAccessFile(file, "rwd")  
            val request = Request.Builder().url(downloadUrl)
                .also { if (needRange) it.header("Range", "bytes=${entity.downloadedLength}-") }
                .get().build()
            response = okHttpClient.newCall(request).await()
            entity.isDownloading = true
            raf.seek(entity.downloadedLength)
            // range 成功后 response code 为 206
            if ((needRange && response.code == 206) || (!needRange && response.isSuccessful)) {
                // 用于计算间隔差
                var delayTime = 0L
                // 用于计算间隔前后的长度差
                var prevLength = entity.downloadedLength
                body = response.body
                body?.let {
                    // 这里就是经典的流传输，随便找个下载功能的实例基本都这么写
                    val bs = body.byteStream()
                    val buffer = ByteArray(DEFAULT_BUFFER_SIZE)
                    var len: Int = bs.read(buffer)
                    while (len != -1) {
                        raf.write(buffer, 0, len)
                        // 给 entity 的 downloadedLength 赋新值
                        entity.downloadedLength += len
                        if (System.currentTimeMillis() - delayTime > RESPONSE_INTERVAL) {
                            val diffLength = entity.downloadedLength - prevLength
                            // 速度[kb/s] = (长度差[b] / 1000)[kb] / (时间间隔[ms] / 1000)[s]
                            entity.speed = (diffLength / 1000) / (RESPONSE_INTERVAL / 1000F)
                            Log.d(TAG, "download: ${entity.speed}")
                            DownloadDatabase.instance.downloadDao.update(entity)
                            delayTime = System.currentTimeMillis()
                            prevLength = entity.downloadedLength
                        }
                        len = bs.read(buffer)
                    }
                }
                return@withContext Result.success()
            } else {

                return@withContext Result.failure(workDataOf("failed_reason" to response.message))
            }
        } catch (e: Exception) {
            // cancellation exception block 一般是代表用户暂停，或者网络受改变
            if (e is CancellationException) {
                return@withContext Result.success()
            }
            return@withContext Result.failure(workDataOf("failed_reason" to e.message))
        } finally {
            raf?.close()
            response?.close()
            body?.close()
            // 无论进行如何，最后都再次更新，保证是最新数据。
            DownloadDatabase.instance.downloadDao.update(
                entity.copy(isDownloading = false)
            )
        }
    }
}
```

我们在开头写了个顶层函数，是为了更方便快捷的开启该 Worker。

### 界面构建

#### XML

##### activity_main.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ui.MainActivity">

    <com.google.android.material.appbar.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <androidx.appcompat.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:title="@string/app_name" />

        <com.google.android.material.tabs.TabLayout
            android:id="@+id/tab_layout"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

    </com.google.android.material.appbar.AppBarLayout>

    <androidx.viewpager2.widget.ViewPager2
        android:id="@+id/view_pager"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior" />

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

##### fragment_download.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/rv"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:clipToPadding="false"
        android:padding="8dp" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

##### item_downloaded.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

    </data>

    <com.google.android.material.card.MaterialCardView
        style="@style/Widget.Material3.CardView.Elevated"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginBottom="8dp">

        <androidx.constraintlayout.widget.ConstraintLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:padding="8dp">

            <TextView
                android:id="@+id/tv_title"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:ellipsize="end"
                android:maxLines="2"
                android:minLines="2"
                android:textAppearance="@style/TextAppearance.Material3.TitleMedium"
                android:textStyle="bold"
                app:layout_constrainedHeight="true"
                app:layout_constrainedWidth="true"
                app:layout_constraintEnd_toEndOf="parent"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toTopOf="parent"
                tools:text="1233333333333333" />

            <TextView
                android:id="@+id/tv_size"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="4dp"
                android:textSize="16sp"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toBottomOf="@id/tv_title"
                tools:text="123 MB" />

            <TextView
                android:id="@+id/tv_added_time"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="4dp"
                android:drawablePadding="2dp"
                android:gravity="center"
                android:textSize="14sp"
                android:textStyle="bold"
                app:layout_constraintBottom_toBottomOf="@id/_barrier"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toBottomOf="@id/tv_size"
                app:layout_constraintVertical_bias="0.0"
                tools:text="2021/02/03 15:12" />

            <TextView
                android:id="@+id/tv_release_date"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:ellipsize="end"
                android:gravity="center"
                android:maxLines="1"
                android:visibility="gone"
                app:layout_constrainedWidth="true"
                app:layout_constraintBaseline_toBaselineOf="@id/tv_added_time"
                app:layout_constraintBottom_toBottomOf="@id/_barrier"
                app:layout_constraintEnd_toEndOf="parent"
                tools:text="2021-02-02" />

            <androidx.constraintlayout.widget.Barrier
                android:id="@+id/_barrier"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                app:barrierDirection="bottom"
                app:constraint_referenced_ids="tv_added_time" />

            <com.google.android.material.button.MaterialButton
                android:id="@+id/btn_delete"
                style="@style/Widget.Material3.Button.TextButton"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="删除"
                app:iconGravity="end"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toBottomOf="@id/_barrier" />

            <com.google.android.material.button.MaterialButton
                android:id="@+id/btn_open"
                style="@style/Widget.Material3.Button.TextButton"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="打开"
                app:iconGravity="start"
                app:layout_constraintEnd_toEndOf="parent"
                app:layout_constraintTop_toBottomOf="@id/_barrier" />

        </androidx.constraintlayout.widget.ConstraintLayout>


    </com.google.android.material.card.MaterialCardView>
</layout>
```

##### item_downloading.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

    </data>

    <com.google.android.material.card.MaterialCardView
        style="@style/Widget.Material3.CardView.Elevated"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginBottom="8dp">

        <androidx.constraintlayout.widget.ConstraintLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:padding="8dp">

            <TextView
                android:id="@+id/tv_title"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"

                android:ellipsize="end"
                android:maxLines="2"
                android:minLines="2"
                android:textAppearance="@style/TextAppearance.Material3.TitleMedium"
                app:layout_constrainedHeight="true"
                app:layout_constrainedWidth="true"
                app:layout_constraintEnd_toEndOf="parent"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toTopOf="parent"
                tools:text="1233333333333333" />

            <LinearLayout
                android:id="@+id/ll_progress"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginTop="4dp"
                android:gravity="center"
                android:orientation="horizontal"
                app:layout_constrainedWidth="true"
                app:layout_constraintEnd_toEndOf="parent"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toBottomOf="@id/tv_title">

                <ProgressBar
                    android:id="@+id/pb_progress"
                    style="?android:attr/progressBarStyleHorizontal"
                    android:layout_width="0dp"
                    android:layout_height="wrap_content"
                    android:layout_weight="1"
                    android:max="100" />

                <TextView
                    android:id="@+id/tv_progress"
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:layout_marginStart="8dp"
                    tools:text="88%" />

            </LinearLayout>

            <TextView
                android:id="@+id/tv_size"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="4dp"
                android:gravity="center"
                app:layout_constraintBottom_toBottomOf="@id/_barrier"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toBottomOf="@id/ll_progress"
                tools:text="1 MB / 7 MB" />

            <TextView
                android:id="@+id/tv_speed"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:ellipsize="end"
                android:gravity="center"
                android:maxLines="1"
                app:layout_constrainedWidth="true"
                app:layout_constraintBaseline_toBaselineOf="@id/tv_size"
                app:layout_constraintBottom_toBottomOf="@id/_barrier"
                app:layout_constraintEnd_toEndOf="parent"
                tools:text="12 MB/s" />

            <androidx.constraintlayout.widget.Barrier
                android:id="@+id/_barrier"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                app:barrierDirection="bottom"
                app:constraint_referenced_ids="tv_speed" />

            <com.google.android.material.button.MaterialButton
                android:id="@+id/btn_cancel"
                style="@style/Widget.Material3.Button.TextButton"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="取消下载"
                app:iconGravity="end"
                app:layout_constraintStart_toStartOf="parent"
                app:layout_constraintTop_toBottomOf="@id/_barrier" />

            <com.google.android.material.button.MaterialButton
                android:id="@+id/btn_start"
                style="@style/Widget.Material3.Button.TextButton"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="开始下载"
                app:iconGravity="start"
                app:layout_constraintEnd_toEndOf="parent"
                app:layout_constraintTop_toBottomOf="@id/_barrier" />

        </androidx.constraintlayout.widget.ConstraintLayout>


    </com.google.android.material.card.MaterialCardView>
</layout>
```

##### layout_new_download.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <EditText
        android:id="@+id/et_name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="名称"
        android:maxLines="2" />

    <EditText
        android:id="@+id/et_url"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="地址"
        android:maxLines="2" />

    <androidx.appcompat.widget.SwitchCompat
        android:id="@+id/switch_imm"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="是否立即下载" />

</LinearLayout>
```

------

先简单介绍一下 [BaseRecyclerViewAdapterHelper](https://github.com/CymChad/BaseRecyclerViewAdapterHelper)：

- `convert`方法约等于原生的`onBindViewHolder`。

- `onItemViewHolderCreated`方法约等于原生的`onCreateViewHolder`。

- `COMPARATOR`相当于用于数据比较刷新的一个 Callback，

  `setDiffCallback`方法用来设置这个 Callback，

  `setDiffNewData`用来通过这个 Callback 异步加载并比较刷新数据。

#### DownloadedRvAdapter

```kotlin
package com.example.easydownload.ui

import android.text.format.DateFormat
import android.text.format.Formatter
import android.view.View
import androidx.core.net.toFile
import androidx.core.net.toUri
import androidx.recyclerview.widget.DiffUtil
import com.chad.library.adapter.base.BaseQuickAdapter
import com.chad.library.adapter.base.viewholder.BaseDataBindingHolder
import com.example.easydownload.R
import com.example.easydownload.database.DownloadDatabase
import com.example.easydownload.database.DownloadEntity
import com.example.easydownload.databinding.ItemDownloadedBinding
import com.example.easydownload.suspendLaunch

/**
 * @project EasyDownload
 * @author Yenaly Liew
 * @time 2023/09/10 010 17:10
 */
class DownloadedRvAdapter :
    BaseQuickAdapter<DownloadEntity, DownloadedRvAdapter.ViewHolder>(R.layout.item_downloaded) {

    inner class ViewHolder(view: View) : BaseDataBindingHolder<ItemDownloadedBinding>(view)

    object COMPARATOR : DiffUtil.ItemCallback<DownloadEntity>() {
        override fun areContentsTheSame(oldItem: DownloadEntity, newItem: DownloadEntity): Boolean {
            return oldItem == newItem
        }

        override fun areItemsTheSame(oldItem: DownloadEntity, newItem: DownloadEntity): Boolean {
            return oldItem.id == newItem.id
        }
    }

    override fun convert(holder: ViewHolder, item: DownloadEntity) {
        holder.dataBinding!!.let { binding ->
            binding.tvTitle.text = item.name
            binding.tvSize.text = Formatter.formatShortFileSize(context, item.downloadedLength)
            binding.tvAddedTime.text = DateFormat.format("yyyy-MM-dd HH:mm", item.addDate)
        }
    }

    override fun onItemViewHolderCreated(viewHolder: ViewHolder, viewType: Int) {
        viewHolder.dataBinding!!.btnDelete.setOnClickListener {
            val pos = viewHolder.bindingAdapterPosition
            val item = getItem(pos)
            val file = item.uri.toUri().toFile()
            if (file.exists()) file.delete()
            context.suspendLaunch {
                DownloadDatabase.instance.downloadDao.delete(item)
            }
        }
        viewHolder.dataBinding!!.btnOpen.setOnClickListener {
            val pos = viewHolder.bindingAdapterPosition
            val item = getItem(pos)
        }
    }
}
```

这个比较好理解，不多说了。

#### DownloadingRvAdapter

```kotlin
package com.example.easydownload.ui

import android.annotation.SuppressLint
import android.text.format.Formatter
import android.view.View
import androidx.core.net.toFile
import androidx.core.net.toUri
import androidx.recyclerview.widget.DiffUtil
import androidx.work.WorkManager
import com.chad.library.adapter.base.BaseQuickAdapter
import com.chad.library.adapter.base.viewholder.BaseDataBindingHolder
import com.example.easydownload.R
import com.example.easydownload.database.DownloadDatabase
import com.example.easydownload.database.DownloadEntity
import com.example.easydownload.databinding.ItemDownloadingBinding
import com.example.easydownload.suspendLaunch
import com.example.easydownload.worker.enqueueDownloadWorker
import com.google.android.material.button.MaterialButton

/**
 * @project EasyDownload
 * @author Yenaly Liew
 * @time 2023/09/10 010 16:22
 */
class DownloadingRvAdapter :
    BaseQuickAdapter<DownloadEntity, DownloadingRvAdapter.ViewHolder>(R.layout.item_downloading) {

    inner class ViewHolder(view: View) : BaseDataBindingHolder<ItemDownloadingBinding>(view)

    object COMPARATOR : DiffUtil.ItemCallback<DownloadEntity>() {
        override fun areContentsTheSame(oldItem: DownloadEntity, newItem: DownloadEntity): Boolean {
            return oldItem == newItem
        }

        override fun areItemsTheSame(oldItem: DownloadEntity, newItem: DownloadEntity): Boolean {
            return oldItem.id == newItem.id
        }
    }

    @SuppressLint("SetTextI18n")
    override fun convert(holder: ViewHolder, item: DownloadEntity) {
        holder.dataBinding!!.let { binding ->
            binding.tvTitle.text = item.name
            binding.tvSize.text =
                "${
                    Formatter.formatShortFileSize(context, item.downloadedLength)
                }/${
                    Formatter.formatShortFileSize(context, item.length)
                }"
            binding.tvSpeed.text = "${item.speed} kb/s"
            binding.tvProgress.text = "${item.progress}%"
            binding.pbProgress.setProgress(item.progress, false)
            binding.btnStart.handleStartButton(item.isDownloading)
        }
    }

    override fun onItemViewHolderCreated(viewHolder: ViewHolder, viewType: Int) {
        viewHolder.dataBinding!!.btnStart.setOnClickListener {
            val pos = viewHolder.bindingAdapterPosition
            val item = getItem(pos)
            // 点击之后，如果当前在下载，就暂停，并取消
            // 如果当前暂停，则启动
            if (item.isDownloading) {
                item.isDownloading = false
                WorkManager.getInstance(context.applicationContext)
                    .cancelUniqueWorkAndPause(item)
            } else {
                item.isDownloading = true
                enqueueDownloadWorker(item.name, item.url, true)
            }
            // 重新对该 Button 进行绘制
            viewHolder.dataBinding!!.btnStart.handleStartButton(item.isDownloading)
        }
        viewHolder.dataBinding!!.btnCancel.setOnClickListener {
            val pos = viewHolder.bindingAdapterPosition
            val item = getItem(pos)
            WorkManager.getInstance(context.applicationContext)
                .cancelUniqueWorkAndDelete(item)
        }
    }

    private fun MaterialButton.handleStartButton(isDownloading: Boolean) {
        text = if (isDownloading) "暂停" else "继续"
    }

	// 取消并删除（文件+数据库内）
    private fun WorkManager.cancelUniqueWorkAndDelete(
        entity: DownloadEntity,
        workName: String = entity.url,
    ) {
        cancelUniqueWork(workName)
        val file = entity.uri.toUri().toFile()
        if (file.exists()) file.delete()
        context.suspendLaunch {
            DownloadDatabase.instance.downloadDao.delete(entity)
        }
    }

	// 取消（暂停）
	// item.isDownloading 在上面的逻辑写了，所以这里直接更新就可以了
    private fun WorkManager.cancelUniqueWorkAndPause(
        entity: DownloadEntity,
        workName: String = entity.url,
    ) {
        cancelUniqueWork(workName)
        context.suspendLaunch {
            DownloadDatabase.instance.downloadDao.update(entity)
        }
    }
}
```

这个相对来说复杂一点，因为我们要控制开始和暂停还有背后的逻辑。

#### DownloadedFragment

```kotlin
package com.example.easydownload.ui

import android.os.Bundle
import android.view.View
import androidx.fragment.app.Fragment
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import com.example.easydownload.R
import com.example.easydownload.database.DownloadDatabase
import com.example.easydownload.id
import kotlinx.coroutines.launch

/**
 * @project EasyDownload
 * @author Yenaly Liew
 * @time 2023/09/10 010 16:21
 */
class DownloadedFragment : Fragment(R.layout.fragment_download) {
    private val rv by id<RecyclerView>(R.id.rv)

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        rv.layoutManager = LinearLayoutManager(context)
        // 创建 adapter 并设置 Callback
        rv.adapter = DownloadedRvAdapter().apply {
            setDiffCallback(DownloadedRvAdapter.COMPARATOR)
        }
        // 加载数据库中 已下载 的数据，并让 adapter 异步刷新
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                DownloadDatabase.instance.downloadDao.loadAllDownloaded().collect {
                    (rv.adapter as DownloadedRvAdapter).setDiffNewData(it)
                }
            }
        }
    }
}
```

#### DownloadingFragment

```kotlin
package com.example.easydownload.ui

import android.os.Bundle
import android.view.View
import androidx.fragment.app.Fragment
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import com.example.easydownload.R
import com.example.easydownload.database.DownloadDatabase
import com.example.easydownload.id
import kotlinx.coroutines.launch

/**
 * @project EasyDownload
 * @author Yenaly Liew
 * @time 2023/09/10 010 16:21
 */
class DownloadingFragment : Fragment(R.layout.fragment_download) {
    private val rv by id<RecyclerView>(R.id.rv)

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        rv.layoutManager = LinearLayoutManager(context)
        rv.adapter = DownloadingRvAdapter().apply {
            setDiffCallback(DownloadingRvAdapter.COMPARATOR)
        }
        // 让 item 的刷新动画时长为 0
        rv.itemAnimator?.changeDuration = 0

        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                DownloadDatabase.instance.downloadDao.loadAllDownloading().collect {
                    (rv.adapter as DownloadingRvAdapter).setDiffNewData(it)
                }
            }
        }
    }
}
```

和 DownloadedFragment 并无太大区别，主要其中有一句

```kotlin
rv.itemAnimator?.changeDuration = 0
```

这一句是让 item 的刷新动画时长为 0。为什么要这样呢？

`loadAllDownloading().collect`是很频繁的，因为每隔 500 ms（假设只下载一个文件，下载多个可能比这还快），都会触发数据库的更新。既然更新，再加上我们调用了`setDiffNewData`，所以每个 item 都会比较频繁的闪烁。将`changeDuration`设为 0，可有效避免。

顺便一提，在 `itemAnimator`里面还有`moveDuration`，`addDuration`，`removeDuration`三个属性。保留这三个动画能使视觉观感更好，并且不会造成闪烁问题。

#### MainActivity

```kotlin
package com.example.easydownload.ui

import android.os.Bundle
import android.view.Menu
import android.view.MenuInflater
import android.view.MenuItem
import android.view.View
import android.widget.EditText
import androidx.appcompat.app.AppCompatActivity
import androidx.appcompat.widget.SwitchCompat
import androidx.core.view.MenuProvider
import androidx.fragment.app.Fragment
import androidx.viewpager2.adapter.FragmentStateAdapter
import androidx.viewpager2.widget.ViewPager2
import com.example.easydownload.R
import com.example.easydownload.id
import com.example.easydownload.worker.enqueueDownloadWorker
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import com.google.android.material.tabs.TabLayout
import com.google.android.material.tabs.TabLayoutMediator

class MainActivity : AppCompatActivity(R.layout.activity_main) {

    companion object {
        val tabArray = arrayOf("正在下载", "已下载")
    }

    private val tabLayout by id<TabLayout>(R.id.tab_layout)
    private val viewPager by id<ViewPager2>(R.id.view_pager)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setSupportActionBar(findViewById(R.id.toolbar))
        init()
    }

    private fun init() {
        // 设置 Menu
        addMenuProvider(object : MenuProvider {
            override fun onCreateMenu(menu: Menu, menuInflater: MenuInflater) {
                menu.add(/* groupId = */ 0, /* itemId = */ 123,
                    /* order = */ 0, /* title = */ "添加"
                )
            }

            override fun onMenuItemSelected(menuItem: MenuItem): Boolean {
                if (menuItem.itemId == 123) {
                    val view =
                        View.inflate(this@MainActivity, R.layout.layout_new_download, null)
                    val etUrl = view.findViewById<EditText>(R.id.et_url)
                    val etName = view.findViewById<EditText>(R.id.et_name)
                    val switchImm = view.findViewById<SwitchCompat>(R.id.switch_imm)

                    // 新建任务
                    MaterialAlertDialogBuilder(this@MainActivity)
                        .setTitle("新建下载")
                        .setView(view)
                        .setPositiveButton("添加") { _, _ ->
                            val url = etUrl.text.toString()
                            val name = etName.text.toString()
                            val imm = switchImm.isChecked
                            if (url.isNotEmpty() && name.isNotEmpty()) {
                                enqueueDownloadWorker(name, url, imm)
                            }
                        }
                        .setNegativeButton("取消", null)
                        .show()
                }
                return true
            }
        })
        viewPager.adapter = object : FragmentStateAdapter(this) {
            override fun getItemCount() = 2

            override fun createFragment(position: Int): Fragment {
                return when (position) {
                    0 -> DownloadingFragment()
                    else -> DownloadedFragment()
                }
            }

        }
        TabLayoutMediator(tabLayout, viewPager) { tab, position ->
            tab.text = tabArray[position]
        }.attach()
    }
}
```

这里实现也不难。主要就是 TabLayout 和 ViewPager2 的结合和 Menu 的创建。

注意这个`addMenuProvider`是新的创建 Menu 的 API，个人感觉用起来比原来需要重写的那个舒服一点。

## 启动！

我们以两个文件进行测试，一个是百度 APP 安装包，一个是幻塔游戏安装包，发现能正常创建并获取到文件长度。

![创建新下载任务](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/da3b69d5cb374bacb658e93bb05fad82~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=600&h=1378&s=12830656&e=gif&f=293&b=fcf9fd)

下载也正常，多个文件可以同时下载，且可随时暂停继续。

将后台 kill 后再次进入程序，发现历史记录全部存在未消失。（这里忘录屏了。但所有信息都储存到数据库了，下载信息肯定不会丢失）

![下载中](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7caf12a834f4ed1996d2a70a66d0a14~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=600&h=1378&s=1263780&e=gif&f=233&b=faf6fb)

其中一个下载完成后，很顺滑地转移到另一界面。

![下载临结束](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f73fb8ddb55441baab58b6cd8e1c2eb6~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=600&h=1378&s=2200827&e=gif&f=183&b=faf6fb)

测试 APP 是否完整，可看到可以正常安装及使用。

![测试](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/df5e433408d84756ab6fa9ab663f6d54~tplv-k3u1fbpfcp-jj-mark:3024:0:0:0:q75.awebp#?w=312&h=718&s=1958210&e=gif&f=326&b=202020)

## 总结

### 优点

优点不必多说，开头也讲了，主要就是简单实用，不用费多大脑筋就能完成一个比较完善的单线程下载功能。

### 缺点

1. 压力集中在数据库上，性能可能不高，很多功能其实可以拆分。
2. 灵活性不高，增添一些字段显示需要对数据库进行更改。
3. ...

------

我的教程更适合当一个跳板，作为学习使用，之后再用 interface 等等做到更复杂、效率又高、又完善的下载功能。

本软件做的有点简陋，但是作为演示很足够了！所有代码全部都贴到上面了，没有夹带私货的函数或者类，复制即可用。

教程到这里就结束了，希望能帮到大家！
