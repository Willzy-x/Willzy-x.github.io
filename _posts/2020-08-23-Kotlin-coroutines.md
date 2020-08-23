---
layout:     post
title:      Kotlin 学习
subtitle:   🎯 Kotlin协程笔记
date:       2020-08-23
author:     HYC
header-img: img/kotlin-banner.png
catalog: true
tags:
    - Kotlin
    - Android
    - Coroutine
---

协程的关键在于`suspend`关键字，`Continuation`，协程的上下文。`suspend`决定了协程将在哪里挂起，而执行语句的状态存储在`Continuation`的对象中，协程的上下文可以决定协程在哪个线程上执行。举个例子：
- `Dispatcher.Main`可以让协程在主线程上执行
- `Dispatcher.Default`生成一个线程数量和CPU内核数量相等的线程池，协程在线程池中执行

## 1. Kotlin的挂起函数
### 1.1 suspend和Continuation
假设我们代码中有这么一个函数：
``` kotlin
suspend fun getUserSuspend(userId: String): User {
    delay(1000)
    return User(userId, "Filip")
}
```

也是一个非常简单的挂起函数，在其中我们调用了`delay`的方法，它会将协程挂起一段时间。它的等价Java代码是这样的：
``` java
@Nullable
public static final Object getUserSuspended(@NotNull String userId, @NotNull Continuation $completion) {
   Object $continuation;
   label20: {
      if ($completion instanceof <undefinedtype>) {
         $continuation = (<undefinedtype>)$completion;
         if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
            ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
            break label20;
         }
      }

      $continuation = new ContinuationImpl($completion) {
         // $FF: synthetic field
         Object result;
         int label;
         Object L$0;

         @Nullable
         public final Object invokeSuspend(@NotNull Object $result) {
            this.result = $result;
            this.label |= Integer.MIN_VALUE;
            return MainApplicationKt.getUserSuspended((String)null, this);
         }
      };
   }

   Object $result = ((<undefinedtype>)$continuation).result;
   Object var4 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
   switch(((<undefinedtype>)$continuation).label) {
   case 0:
      ResultKt.throwOnFailure($result);
      ((<undefinedtype>)$continuation).L$0 = userId;
      ((<undefinedtype>)$continuation).label = 1;
      if (DelayKt.delay(1000L, (Continuation)$continuation) == var4) {
         return var4;
      }
      break;
   case 1:
      userId = (String)((<undefinedtype>)$continuation).L$0;
      ResultKt.throwOnFailure($result);
      break;
   default:
      throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
   }

   return new User(userId, "Filip");
}
```

在上面，我们可以注意到它函数参数中的`Continuation`参数，它是协程的全部的基础设施。它可以被理解为是系统的回调函数或是正在执行的程序。除此之外，它还有着状态机的作用，是对程序控制状态的封装，它在上面的代码中的属性`label`有0，1两种不同的状态，利用这个状态，它可以知道什么时候应该执行函数中哪一些代码，在哪挂起、什么时候继续等等。最后当`label`为1时，如果没有异常产生，那么这个挂起函数就会正常终止，返回一个`User`对象。

究竟什么是`continuation`？每当我们程序中调用一个函数，它就会被加在程序的调用栈（callstack）上。`Continuation`其实就是类似这个call-stack，但是是在非常低的系统层面上实现的。其实更准确的解释还是上面讲的对状态的封装。在每个函数执行之后，它知道在哪返回，而且也会保存一些和上下文相关的信息。比如局部变量，传入的参数，在哪个线程调入等等。

### 1.2 处理continuation
Continuation有一些函数和上下文可以使用和访问。我们可以使用`continuation.context`来获取当前上下文。
- `resume()`让协程继续执行，并且传入一个类型为T的参数作为从上一个挂起点的返回值。
- `resumeWith()`也是让协程继续执行，并且传入一个类型为T的参数表示成功或失败值作为从上一个挂起点的返回值。
- `resumeWithException()`接受一个Throwable类型的参数，应对程序执行出错的情况。

## 2. 协程基本用法
### 2.1 启动协程
通过下面这段代码，我们能很快地去生成一个协程：
``` kotlin
fun main() {
    GlobalScope.launch {
        // code here...
    }
}
```

其中接受协程体的`launch()`函数叫做协程构造器。其次，当启动一个协程时，你需要指定它的协程范围，因为我们需要管控协程的生命周期，万一程序在协程完成前执行完毕了怎么办？因此，在这里我们使用了`GlobalScope`，也就是说这个协程的生命周期是和整个应用的生命周期是绑定的。最后我们看看一个协程构造器的函数签名是什么样子的：
``` kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

它接受3个参数，分别是协程的上下文`context`，协程的启动方式，和执行的协程体：一个lambda函数。前两个参数都含有默认值。
- `context`一般是使用`EmptyCoroutineContext`，可以指代当前协程范围所声明的任何上下文。当然也可以使用：`Dispatcher.Main`, `Dispatcher.Default`, `Dispatcher.Unconfined`, `Dispatcher.IO`。来让协程执行在不同的线程中。
- `start`让我们指定协程开始执行的方式，我们可以使用：
  - `DEFAULT`：根据当前的上下文立刻安排协程执行
  - `LAZY`：使用懒惰的方法启动协程
  - `ATOMIC`：和DEFAULT一样，但是在它开始之前不可以被取消
  - `UNDISPATCHED`：执行协程直到它的第一个挂起点

### 2.2 Job
其实协程中的大部分都和你创建并运行的job相关，`Job`也是`launch()`函数返回的对象。其实本质上是队列中协程的一个句柄（handle）。虽然它只有一些字段和函数，但是它提供了不错的可扩展性。比如，我们可以使用`join`在不同的`Job`之间添加依赖关系。

当我们启动协程并创建一个Job的时候，许多情况可能发生，比如一个异常的产生，而通常一个未被捕获的异常可能会导致整个程序的崩溃。我们可以通过取消Job来应对这种情况，即在Job上调用`cancel()`方法。而且如果你取消了一个Job，系统会自动帮你一起取消它的所有子Job。如果取消的Job有一个父Job，那么它的父Job也会被取消。

### 2.3 Android 中 CoroutineScope
- `ViewModelScope`，请使用 `androidx.lifecycle:lifecycle-viewmodel-ktx:2.1.0-beta01` 或更高版本。
为应用中的每个 `ViewModel`定义了 `ViewModelScope`。如果 `ViewModel` 已清除，则在此范围内启动的协程都会自动取消。如果您具有仅在 `ViewModel` 处于活动状态时才需要完成的工作，此时协程非常有用。例如，如果要为布局计算某些数据，则应将工作范围限定至 `ViewModel`，以便在 `ViewModel` 清除后，系统会自动取消工作以避免消耗资源。
您可以通过 `ViewModel` 的 `viewModelScope` 属性访问 `ViewModel` 的 `CoroutineScope`，如以下示例所示：
``` kotlin
class MyViewModel: ViewModel() {
        init {
            viewModelScope.launch {
                // Coroutine that will be canceled when the ViewModel is cleared.
            }
        }
    }
```

- `LifecycleScope`，请使用 `androidx.lifecycle:lifecycle-runtime-ktx:2.2.0-alpha01` 或更高版本。
为每个 `Lifecycle` 对象定义了 `LifecycleScope`。在此范围内启动的协程会在 `Lifecycle` 被销毁时取消。您可以通过 `lifecycle.coroutineScope` 或 `lifecycleOwner.lifecycleScope` 属性访问 `Lifecycle` 的 `CoroutineScope`。
以下示例演示了如何使用 `lifecycleOwner.lifecycleScope` 异步创建预计算文本：
``` kotlin
class MyFragment: Fragment() {
        override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
            super.onViewCreated(view, savedInstanceState)
            viewLifecycleOwner.lifecycleScope.launch {
                val params = TextViewCompat.getTextMetricsParams(textView)
                val precomputedText = withContext(Dispatchers.Default) {
                    PrecomputedTextCompat.create(longTextContent, params)
                }
                TextViewCompat.setPrecomputedText(textView, precomputedText)
            }
        }
    }
```

## 3. Async/Await
### 3.1 Promises
async/await一直是并发编程中常用的概念，在Javascript中也有其实现。`async`函数返回的是一个`Promise`对象，是对最后返回值的封装。我们可以使用`then`和`catch`分别对其返回的结果和抛出的异常进行处理。最终会形成一个调用链：
``` js
database.findOne({email: request.email})
    .then(user => {
        if (!user) {
            // the user doesn't exist
            return service.registerUser(request.data)
        } else {
            return null
        }
    })
    .then(registerStatus => {
        // do something after registration
    })
    .catch(error => {
        // handle error
    })
```

### 3.2 Future
除此之外，async/await概念和Java中的`Future`也有些类似的地方。通过`Future`，我们可以先运行并发的工作线程，然后在未来某一刻去获取到线程返回的值：
``` java
private static ExecutorService executor = Executors.newSingleThreadExecutor();

public static Future<Integer> parse(final String input) {
    return executor.submit(() -> {
        Thread.sleep(1000);
        return Integer.parseInt(input);
    });
}

public static void main(String[] args) throws ExecutionException, InterruptedException {
    Future<Integer> parser = parse("310");
    
    while (!parser.isDone()) {
        // waiting to parse
    }
    
    int parsedValue = parser.get();
}
```

但是它的缺点在于当我们想去拿返回的值的时候，我们通常都是以一种阻塞的方式去拿，这种方式的开销是非常大的，尤其当我们有用户界面需要处理的时候。可能会导致屏幕上面的界面卡死或者甚至产生死锁。

### 3.3 async/await
和上面不同的是async/await模式依赖的是挂起函数，比如Kotlin中的协程API，它不需要去阻塞代码。我们这里展示一个模拟从网络拿取用户数据的例子：
``` kotlin
fun main() {
    var userId = 100
    userId = 200

    GlobalScope.launch {
        val userData = getUserByIdFromNetworkAsync(userId)

        println(userData.await())
    }

    Thread.sleep(5000)
}

private fun getUserByIdFromNetworkAsync(userId: Int): Deferred<User> =
        GlobalScope.async {
            kotlinx.coroutines.delay(3000)

            User(userId, "Filip", "Babic")
        }

data class User(val id: Int, val name: String, val lastName:
String)
```

当我们使用`GlobalScope.async`函数时，返回的是一个类型为`Deferred<T>`的值。通过这个接口，我们可以用`await`方法拿到我们最终想要的数据。

## 4. Producer & Actor
生产者消费者问题，比如：线程池是个非常好的例子，其中的生产者就是worker，thread就是消费者。
### 4.1 创建生产者
我们可以使用Kotlin提供的`GlobalScope`创建一个生产者。
``` kotlin
val producer = GlobalScope.produce<Int>(capacity = 10) {  }
```

我们来看看这个函数的签名：
``` kotlin
public fun <E> CoroutineScope.produce(
    context: CoroutineContext = EmptyCoroutineContext,
    capacity: Int = 0,
    @BuilderInference block: suspend ProducerScope<E>.() -> Unit
): ReceiveChannel<E>
```

我们可以在这里传入一个`context`，一个`capacity`代表这个生产队列的大小，最后是一个lambda函数，在里面我们可以定义生产者的行为。除此之外，我们还注意到一个注解叫`BuilderInference`，它使得编译器能推断外层泛型函数的类型。比如说如果运行一个`ProducerScope`能一起运行的像`String`一样的函数，那么`produce`函数就知道要创建的是一个`Producer<String>`。

### 4.2 产生数值
在这里我们使用了`offer`函数来将元素入队，如果队列中没有空闲位置来放入新的元素，那么就会返回一`false`，如果成功放入就返回`true`。还有一个值得注意的地方就是检查`isClosedForSend`的值，如果显式调用`close`函数后或者出现异常后，这个值会变为`false`。
``` kotlin
val producer = GlobalScope.produce<Int>(capacity = 10) { 
    while (isActive) {
        if (!isClosedForSend) {
            val number = Random.nextInt()
            if (offer(number)) {
                println("$number sent")
            } else {
                println("$number discarded")
            }
        }
    }
}
```

除此之外，我们也可以使用`isFull`这个值来检测这个生产队列是否已满，如果满了就停止生产。
``` kotlin
if (!isFull) { 
    // code
}
```

如果不用`offer`，我们可以使用`send`方法：
``` kotlin
val producer = GlobalScope.produce<Int>(capacity = 10) {
    while (isActive) {
        val number = Random.nextInt(0, 20)
        send(number)
        println("$number sent")
    }
}
```

在这种情况下，当队列满的时候，我们不会浪费任何一个值，因为`send`函数是可挂起的。它不会继续执行，直到队列中有空的位置腾出来。

### 4.3 消费数值
`produce`函数很方便地返回了一个`ReceiveChannel`。这就意味着你可以对里面的数值进行迭代。一个最简单的例子，一个`while`循环：
``` kotlin
while (producer.isClosedForReceive) {
    val number = producer.poll()
    
    if (number != null) {
        println("#number received")
    }
}
```

在这里，我们使用了`poll`方法来拿出生产队列 中的数值，直到最后`isClosedForReceive`是一个`false`值。而且在使用`poll`方法时，我们也需要检测空值，这是因为这个方法是非阻塞的，而且如果这个通道是空的话就会返回`null`。如果不想处理空值，我们可以使用这个方法：
``` kotlin
GlobalScope.launch { 
    producer.consumeEach { 
        println("$it received")
    }
}
```

除了`poll`，我们也可以使用`receive`方法，这和`send`一样，也是可挂起的：
``` kotlin
GlobalScope.launch {
    while (isActive) {
        val value = producer.receive()
        println("$value received")
    }
}
```
