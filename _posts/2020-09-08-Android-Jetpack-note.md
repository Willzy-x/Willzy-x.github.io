---
layout:     post
title:      安卓基础知识
subtitle:   Android Jetpack 笔记
date:       2020-09-08
author:     HYC
header-img: img/android_jetpack_new.png
catalog: true
tags:
    - Android
    - Java
    - Jetpack
    - Kotlin
---


## `LiveData`和`MutableLiveData`
直接理解`LiveData`的字面意思是前台数据，其实这其实是很准确的表达。下面我们来说说`LiveData`的几个特征：
1. 和数据实体类（POJO）是一样的东西，它负责暂存数据。
2. 也是一个观察者模式的数据实体类，它可以跟它注册的观察者回调数据是否已经更新。
3. LiveData还能知道它绑定的`Activity`或`Fragment`的生命周期，它只会给前台活动回调。这样你可以放心地在它的回调方法里面直接将数据添加到`View`，而不用担心会不会报错。

`LiveData`和`MutableLiveData`的概念几乎是一模一样的，为数不多的几个区别如下：
1. `MutableLiveData`的父类是`LiveData`
2. `LiveData`在实体类里可以通知某个字段的数据更新
3. `MutableLiveData`则是完全是整个实体类或者数据类型变化后才通知，不会细节到某个字段

### API讲解
- `observeForever(@NonNull Observer<? super T> observer)`
  - 设置永远的观察者，永远不会被自动删除。需要手动调用`removeObserver(observer)`来停止观察此`LiveData`。
  - 设置后，此`LiveData`一直处于活动状态，不管是否在前台都会获得回调。
- `removeObserver(@NonNull Observer<? super T>  observer)`
  - 移除指定的观察者。

`LiveData`是jetpack提供的一种响应式编程组件，它可以包含任何类型的数据，并在数据发生变化的时候通知给观察者。`LiveData`特别适合和`ViewModel`结合在一起使用。

在我们原来的例子中，在`Activity`中一直都是用手动的方法去获取`ViewModel`的数据。我们也不能将`Activity`的实例传给`ViewModel`，因为`ViewModel`的生命周期长于`Activity`，可能造成内存泄露。

`LiveData`可以包含任何类型的数据，并在数据发生变化的时候通知给观察者。

``` kotlin
class MainViewModel(countReversed: Int) : ViewModel() {
    var counter = MutableLiveData<Int>()

    init {
        counter.value = countReversed
    }

    fun plusOne() {
        val count = counter.value ?: 0
        counter.value = count + 1
    }

    fun clear() {
        counter.value = 0
    }
}
```

这里，我们将`counter`设置为`MutableLiveData`，也就是一种可变的`LiveData`，它主要有3种可以读写的方法：
- `getValue()`：用于获取`LiveData`中的数据
- `setValue()`：给`LiveData`设置数据，但是只能在主线程中调用
- `postValue()`：用于在非主线程中给`LiveData`设置数据

接下来我们改造`MainActivity`：
``` kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    ...
    plusOneBtn.setOnClickListener {
        viewModel.plusOne()
    }

    clearBtn.setOnClickListener {
        viewModel.clear()
    }
    viewModel.counter.observe(this, Observer { count ->
        infoText.text = count.toString()
    })
}
```

这里最重要的就是最后一行代码。我们的`counter`变量因为是一个`LiveData`，它现在已经可以调用`observe()`方法来观察数据的变化来。这个方法接受两个参数：第一个参数是一个`LifecycleOwner`对象，因此直接传`this`就好；第二个参数是一个`Observer`接口，当`counter`中包含的数据变化时，就会回调这里。

我们应该让不可变的`LiveData`暴露给外面，而不是让可变的`MutableLiveData`暴露给外面。因此我们还需要改动`MainViewModel`，利用多态的性质：

``` kotlin
class MainViewModel(countReversed: Int) : ViewModel() {
    
    val counter: LiveData<Int>
        get() = _counter
    val _counter = MutableLiveData<Int>()

    init {
        _counter.value = countReversed
    }

    fun plusOne() {
        val count = _counter.value ?: 0
        _counter.value = count + 1
    }

    fun clear() {
        _counter.value = 0
    }
}
```

在成员变量前加上下划线，就对外不可见了。保证来`ViewModel`数据的封装性。

## `ViewModel`
`ViewModel`是google推出的一个数据处理框架，`ViewModel`类是被设计用来以可感知生命周期的方式存储和管理 UI 相关数据`ViewModel`中数据会一直存活即使 activity configuration发生变化。另外它生来可能目的就是与Fragment在数据共享上进行配合的。
使用它常与`LiveData`数据前台类(类似观察者模式的数据实体回调类)进行配合以前使用。

`ViewModel`在手机旋转的时候不会被销毁，而`Activity`会被销毁，如果数据是存放在`Activity`上面的话就会很容易丢失。`View`有独立的生命周期，且比`Activity`的生命周期长，因此我们需要独立创建`ViewModel`：

``` kotlin
ViewModelProviders.of(<Activity or Fragment object>).get(<Your ViewModel>::class.java>)
```

有以下的特征：
1. 让数据和UI隔离
2. 数据与生命周期绑定
3. 数据持久化
4. 与其他Activity独立
5. 天生为了配合Fragment

向`ViewModel`传递参数
借助`ViewModelProvider.Factory`实现即可：

``` kotlin
class MainViewModelFactory(private val countReversed: Int) : ViewModelProvider.Factory {
    override fun <T : ViewModel?> create(modelClass: Class<T>): T {
        return MainViewModel(countReversed) as T
    }
}
```

然后只需要实现接口的`create()`方法即可，在这个方法中我们将传递给factory的countReversed传递给了MainViewModel，从而创建了实例。而且这个方法和Activity的生命周期无关。

## `LifeCycles`
我们需要时刻感知`Activity`的生命周期，以便在适当的时候进行相应的逻辑控制，下面展示如何在非`Activity`类中感知`Activity`的生命周期，`LifeCycles`就是为了解决需要在`Activity`中编写大量逻辑而出现的，新建一个`MyObserver`并让它实现`LifecycleObserver`接口，代码如下：

``` kotlin
class MyObserver : LifecycleObserver {

    val TAG : String = "MyObserver"

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun activityStart() {
        Log.d(TAG, "activityStart: ")
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun activityStop() {
        Log.d(TAG, "activityStop: ")
    }
}
```

在上面的代码中，我们使用了注解并传入生命周期的事件，而且这些事件一共有7种：
- `ON_CREATE`
- `ON_START`
- `ON_RESUME`
- `ON_PAUSE`
- `ON_STOP`
- `ON_DESTROY`
分别匹配`Activity`当中的生命周期回调，另外还有一种类型叫`ON_ANY`类型，表示可以匹配`Activity`的任何生命周期。

之后我们通过`LifecycleOwner`添加观察者：  
``` kotlin
lifecycleOwner.lifecycle.addObserver(MyObserver())
```

那如何才能获取`LifecycleOwner`呢？其实`Fragment`和`AppCompatActivity`自身就是一个`LifecycleOwner`的实例，这部分已经由AndroidX库自动帮我们完成了。

``` kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    ...
    lifecycle.addObserver(MyObserver())
    ...
}
```
只需要添加这一行代码，`MyObserver`就能自动感知`Activity`的生命周期了。

我们也可以通过在`MyObserver`的构造函数中将`Lifecycle`对象传递进来获知当前是哪个生命周期状态：

``` kotlin
class MyObserver(val lifecycle: Lifecycle) : LifecycleObserver {
    ...
    lifecycle.currentState
}
```

## `WorkManager`
`WorkManager`组件可以帮助开发者根据不同的操作系统选择不同的底层组件去实现后台的功能，从而降低我们的使用成本。另外，它还支持周期性任务、链式任务处理等功能。

`WorkManager`很适合执行一些定期的服务和服务器进行进行交互的任务，比如周期性地同步数据。

`WorkManager`的基本用法
基本用法其实非常简单主要分为以下3步：
1. 定义一个后台任务，并实现具体的逻辑任务
2. 配置该后台任务的运行条件和约束信息，并构建后台任务请求
3. 将该后台任务请求传入`WorkManager`的`enqueue()`方法中，系统会在合适的时间运行

我们首先创建一个简单的`SimpleWorker`类，重写它的`doWork`方法，在这个方法中编写具体的后台任务的逻辑即可：
``` kotlin
class SimpleWorker (context: Context, params: WorkerParameters) : Worker(context, params) {
    override fun doWork(): Result {
        Log.d("SimpleWorker", "doWork: in SimpleWork")
        return Result.success()
    }

}
```

`doWork`方法不会运行在主线程中，因此你可以放心地在这里执行耗时的逻辑，不过这里只是打印了一行日志而已。`doWork`对象返回的是一个`Result`对象，用于表示任务的运行结果，成功就返回`Result.success()`，失败就返回`Result.failure()`。除了这两个方法之外我们还有一个`Result.retry()`方法，它其实也代表失败，只是可以结合`WorkRequest`.`Builder`的`setBackoffCriteria()`方法来重新执行任务。

然后执行第二步，配置该后台任务的运行条件和约束信息，我们来进行一个简单的配置：  
``` kotlin
val request = OneTimeWorkRequest.Builder(SimpleWorker::class.java).build()
```

我们可以看到只需将刚刚创建的`SimpleWorker的Class`对象传入`OneTimeWorkRequest.Builder`的构造函数中，然后调用`build()`方法即可完成构建。`OneTimeWorkRequest.Builder`是`WorkRequest.Builder`的子类，用于构建单次运行的后台任务请求。它还有另外一个子类：

``` kotlin
val request = PeriodicWorkRequest.Builder(SimpleWorker::class.java, 15, TimeUnit.MINUTES).build()
```

用于构建周期性的运行的后台任务请求，但是为了降低设备的能耗，它传入的运行周期间隔不能短于15分钟。

最后一步，将构建出的后台任务请求传入`WorkManager`的`enqueue()`方法中，系统就会在合适的时间去运行了：
``` kotlin
WorkManager.getInstance(context).enqueue(request)
```

使用`WorkManager`处理复杂任务
- 让后台任务在指定的延迟时间后运行，只需要借助`setInitialDelay()`方法就可以了，代码如下所示：

``` kotlin
val request = OneTimeWorkRequest.Builder(SimpleWorker::class.java)
    .setInitialDelay(5, TimeUnit.MINUTES)
    .build()
```

这就表示我们希望让`SimpleWorker`这个后台任务在5分钟后运行。除了控制时间之外，我们再增加一些别的功能，比如说给后台任务请求添加标签：

``` kotlin
    ...    
    .addTag("Simple")
    ...
```

加了标签之后，最主要的一个功能就是我们可以通过标签来取消后台任务的请求，当然，如果没有标签也可以通过id来取消：

``` kotlin
WorkManager.getInstance(this).cancelAllWorkByTag("Simple")
WorkManager.getInstance(this).cancelWorkById(request.id)
```

`setBackoffCriteria`方法接受3个参数：
- 第一个参数用于指定如果任务再次执行失败，下次重试应该以什么样的方式延迟。有两种选择，一个是`LINEAR`，另一个是`EXPONENTIAL`
- 第二个参数和第三个参数用于指定在多久之后重新执行任务

`.setBackoffCriteria(BackoffPolicy.LINEAR, 10, TimeUnit.SECONDS)`

我们可以使用以下的代码对后台任务的运行结果进行监听：

``` kotlin
WorkManager.getInstance(this).getWorkInfoByIdLiveData(request.id)
    .observe(this, Observer  { t: WorkInfo? ->  
        if (t?.state == WorkInfo.State.SUCCEEDED) {
            Log.d("MainActivity", "onCreate: ")
        } else if (t?.state == WorkInfo.State.FAILED) {
            Log.d("MainActivity", "onCreate: ")
        }
    })
```

`WorkManager`中的链式任务
假设我们定义了3个独立的后台任务：同步数据、压缩数据和上传数据。现在我们想要实现先同步、再压缩、最后上传的功能，就可以借助链式任务来实现：

``` kotlin
val sync = ...
val compress = ...
val upload = ...
WorkManager.getInstance(this)
    .beginWith(sync)
    .then(compress)
    .then(upload)
    .enqueue()
```

`beginWith`方法用于开启一个链式任务，至于后面接什么样的后台任务，只需要使用`then()`方法来连接即可。另外`WorkManager`还要求，必须在前一个后台任务运行成功之后，下一个任务才会执行。

## Reference
1. https://www.cnblogs.com/guanxinjing/category/1550385.html
2. 《第一行代码：Android》
