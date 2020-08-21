---
layout:     post
title:      RxJava3学习笔记
subtitle:   基于《Learn RxJava - Second Edition》published by Packt, Chapter1
date:       2020-07-30
author:     HYC
header-img: img/ReactiveX-FINAL.jpg
catalog: true
tags:
    - Android
    - Reactive Programming
    - Java
---
## 1. `Observable` 和 `Observer`
> The Observable class is a push-based composable iterator. For a given Observable<T>, it pushes items (called emissions) of type T through a series of operators until they finally arrive at a final Observer, which consumes them. 
Observable是一个基于push（emit）操作的可组合的一个迭代器。对于任意一个Observable<T>，它都可以将T类型的的一个数据项经过一系列的操作后push进去，知道它们到达一个最终的消费它的Observer。

我们先添加Maven的依赖：
``` xml
<!-- https://mvnrepository.com/artifact/io.reactivex.rxjava3/rxjava -->
<dependency>
    <groupId>io.reactivex.rxjava3</groupId>
    <artifactId>rxjava</artifactId>
    <version>3.0.4</version>
</dependency>
```
我们首先介绍一下`Observable`的一个基本的操作，也就是`Observable`是如何按照顺序地将数据项通过一个链式的结构传送到`Observer`的：
- `onNext()`：每次传递一个数据项从`Observable`到`Observer`。
- `onComplete()`：将一个完成的事件传递到`Observer`，表明`onNext()`不能再继续调用了。
- `onError()`：将一个错误从链式结构上传递到`Observer`，而且一般这个`Observer`应该会定义如何去处理这个错误。除非调用`retry()`操作来拦截这个错误，否则一般的话，`Observable`链会终止，而且不能再发射（emit）出别的东西。

### 1.1 `Observable`
#### 1.1.1 Observable的`create()`方法
通过这个方法我们可以创建一个源`Observable`，这个`Observable`是所有发射数据项的起始的地方；也是我们的`Observable`链的开始。这个方法是一个工厂方法，它允许我们传递一个接受`Observable`发射器的lambda函数作为参数；也就是说它的参数应该是一个`ObservableOnSubscribe`类型的对象。这个类只有一个方法就是`subsribe(ObservableEmitter emitter)`，它接受一个`ObservableEmitter`的对象作为参数。而这个类呢就是继承我们的`Emitter`接口，其中有三个方法，就是我们上面提到的`onNext()`，`onComplete()`和`onError()`。

下面我们举个🌰讲解一下：
``` java
public class CreateMethodExample {
    public static void main(String[] args) {
        Observable<String> source = Observable.create(emitter -> {
            emitter.onNext("Alpha");
            emitter.onNext("Beta");
            emitter.onNext("Gamma");
            emitter.onComplete();
        });
        source.subscribe(s -> System.out.println("RECEIVED: " + s));
    }
}
```
`onNext()`我们上面提到了，就是将每一个数据项移交给链上的下一站，在这就是我们的`Observer`；它的功能就是拿到移交过来的字符串之后，打印"RECEIVED: " + s这句话。`onComplete()`就是告诉`Observer`没有更多数据项了。

实际上`Observable`是可以不停地去发射数据项：在技术上，就是在停止使用`onNext()`进行发射后，不调用`onComplete()`进行停止。

虽然上面的代码并没有什么错误，但是我们也可以在`create`方法中处理一些错误。在这个方法里我们使用`onError()`将错误发送给`Observer`，让它来处理。`subscribe()`方法有很多重载，上面使用的`subscribe()`重载版本不支持异常的处理，但是我们可以使用另外一个版本：             
``` java                      
// subscribe(Consumer<String> onNext)
// subscribe(Consumer<String> onNext, Consumer<Throwable> onError)
public class ErrorHandling {
    public static void main(String[] args) {
        Observable<String> source = Observable.create(emitter -> {
            try {
                emitter.onNext("Alpha");
                emitter.onNext("Beta");
                emitter.onNext("Gamma");
                emitter.onComplete();
            } catch (Throwable e) {
                emitter.onError(e);
            }
        });
        source.subscribe(s -> System.out.println("RECEIVED: " + s),
                Throwable::printStackTrace);
    }
}
```

#### 1.1.2 map和filter方法
之前说了，在`Observable`和`Observer`之间也可能会有一些站点，而我们一般也不会直接地将数据项从`Observable`传递到`Observer`：我们会在中间的地方对数据项进行一些操作。map()方法和filter()方法就是用来做这个事情的：

为了重复利用代码，我们使用一个`DummyStringEmitter`实现`ObservableOnSubscribe<String>`接口：
``` java
public class DummyStringEmitter implements ObservableOnSubscribe<String> {

    @Override
    public void subscribe(@NonNull ObservableEmitter<String> emitter) throws Throwable {
        try {
            emitter.onNext("Alpha");
            emitter.onNext("Beta");
            emitter.onNext("Gamma");
            emitter.onComplete();
        } catch (Throwable e) {
            emitter.onError(e);
        }
    }
}
```

然后这是我们的🌰：
``` java
public class MapAndFilter {
    public static void main(String[] args) {
        Observable<String> source = Observable.create(new DummyStringEmitter());
        Observable<Integer> lengths = source.map(String::length);
        Observable<Integer> filtered = lengths.filter(i -> i >= 5);
        filtered.subscribe(s -> System.out.println("RECEIVED: " + s));
    }
}

// 运行结果：
// RECEIVED: 5
// RECEIVED: 5
```
`map()`和`filter()`操作被放在了`Observable`和`Observer`之间，任何从`Observable`中发射出来的数据项都会经过`map()`定义的操作。它实际上像是一个中间的`Observer`，并且使用`length()`方法将`Observable`传递过来的字符串转换为一个整数。然后调用`onNext()`在将这些整数传递给后面定义的`filter()`方法，过滤掉那些字符长度不超过5的字符串，最后再继续调用`onNext()`将结果发送给`Observer`。

#### 1.1.3 `Observable`的`just()`方法
在一些情况下，你很可能不需要使用`onCreate()`方法，而是使用`just()`方法就能达到相同的效果。just方法就像Java9中为集合提供的工厂方法一样，如`List.of()`方法，它有多个版本的重载，每个版本比上一个多一个元素，做多支持十个元素作为参数。它对其中的每一个元素调用`onNext()`并在最后调用`onComplete()`方法：
``` java
public class JustExample {
    public static void main(String[] args) {
        Observable<String> source = Observable.just("Alpha", "Beta", "Gamma");
        source.map(String::length).filter(i -> i >= 5).subscribe(s -> System.out.println(s));
    }
}
```

除了`just`，`Observable`也可以使用`fromIterable(`)将任何一个`Iterable`类型的对象转换为一个`Observable`。

1.2 `Observer`接口
``` java
package io.reactivex.rxjava3.core;

import io.reactivex.rxjava3.annotations.NonNull;
import io.reactivex.rxjava3.disposables.Disposable;

public interface `Observer`<T> {
    void onSubscribe(@NonNull Disposable d);
    void onNext(@NonNull T t);
    void onError(@NonNull Throwable e);
    void onComplete();
}
```
每一个被类似`map`操作返回的`Observable`内部其实是一个`Observer`，它接受、转换、中转上流发送的数据项，并传输给下流的`Observer`，它并不知道下流的那个`Observer`是另一个操作还是链的终端。因此，当我们谈到`Observer`时，我们所指的就是那个链末端最终消费发送的数据项的`Observer`。

#### 1.2.1 实现并订阅一个`Observer`
这次，我们不像之前一样在`subscribe()`方法里面传递一个lambda表达式作为参数，我们去实现一个`Observer`，并将其作为参数放入`subscribe()`中：
``` java
public class ImplementingObserverExample {
    public static void main(String[] args) {
        Observable<String> source = Observable.create(new DummyStringEmitter());

        Observer<Integer> myObserver = new Observer<Integer>() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {
                // do nothing with Disposable, disregard for now
            }

            @Override
            public void onNext(@NonNull Integer integer) {
                System.out.println("RECEIVED: " + integer);
            }

            @Override
            public void onError(@NonNull Throwable e) {
                e.printStackTrace();
            }

            @Override
            public void onComplete() {
                System.out.println("Done!");
            }
        };
        source.map(String::length).filter(i -> i >= 5).subscribe(myObserver);
    }
}

// 运行结果
// RECEIVED: 5
// RECEIVED: 5
// Done!
```
我们在上面的例子里创建了一个最终消费处理发送的数据项的`Observer`；也就是说在一些场景中，这些数据可能会被写入数据库、文本文件、服务器的响应报文中、UI的展示抑或是直接输出到终端上面。

每一个字符串都通过上面的操作转换成了一个整数，`Observer`的`onNext()`接受一个整数并且将其打印。同样地，它的`onError()`方法会接受一个`Throwable`异常，并打印这个异常的stackTrace。最终，当源没有别的数据项发送时，调用`onComplete()`方法将完成事件传递给`Observer`，`Observer`也会调用`onComplete()`处理一些完成的逻辑。

### 1.2.2 利用lambda表达式创建简单的`Observer`
> Implementing `Observer` is a bit verbose and cumbersome.
subscribe()对上面的三种事件都进行重载，使之能够接受Lambda表达式。你可以创建3个lambda表达式，并在作为参数传递进subscribe时用逗号隔开： 
``` java
public class ObserverWithLambda {
    public static void main(String[] args) {
        Observable<String> source = Observable.create(new DummyStringEmitter());
        source.map(String::length)
                .filter(integer -> integer >= 5)
                .subscribe(s -> System.out.println("RECEIVED: " + s),
                            Throwable::printStackTrace,
                            () -> System.out.println("Done!"));
    }
}
```

### 1.3 冷热`Observable`
根据`Observable`的不同实现方法，我们可以将其划分为冷和热`Observable`。首先我们需要定义一下，如果一个`Observable`有多个`Observer`时，它的行为是什么样的。

#### 1.3.1 冷`Observable`（Cold Observable）
> A cold Observable is much like a music CD that is provided to each listener, so each person can hear all the tracks any time they start listening to it.  

一个冷`Observable`仅仅是将发射的数据项不停重播给`Observer`，确保`Observer`能拿到全部数据。大多数数据驱动的`Observable`都是冷的`Observable`，包括使用just()和使用fromIterable()创建的`Observable`。
``` java
public class ColdObservableExample {
    public static void main(String[] args) {
        Observable<String> source = Observable.just("Alpha", "Beta", "Gamma");
        // first observer
        source.subscribe(s -> System.out.println("Observer 1: " + s));
        // second observer
        source.subscribe(s -> System.out.println("Observer 2: " + s));
    }
}

// 运行结果：
// Observer 1: Alpha
// Observer 1: Beta
// Observer 1: Gamma
// Observer 2: Alpha
// Observer 2: Beta
// Observer 2: Gamma
```
即使是在第1个`Observer`将接受到的数据进行操作后，第二个还是会拿到自己的从源`Observable`发送过来的数据流，输出的结果：
``` java
public class AnotherColdObservableExample {
    public static void main(String[] args) {
        Observable<String> source = Observable.just("Alpha", "Beta", "Gamma");

        // first observer
        source.map(String::length)
                .filter(i -> i >= 5)
                .subscribe(s -> System.out.println("Observer 1: " + s));
        // second observer
        source.subscribe(s -> System.out.println("Observer 2: " + s));

    }
}

// 运行结果：
// Observer 1: 5
// Observer 1: 5
// Observer 2: Alpha
// Observer 2: Beta
// Observer 2: Gamma
```
注意：一个源源不断发送数据集的`Observable`通常是冷`Observable`。


#### 1.3.2 热`Observable`（Hot Observable）
> A hotObservable is more like a radio station. It broadcasts the same emissions to all observers at the same time. If an Observer subscribes to a hot Observable, receives some emissions, and then another Observer subscribes later, that second Observer will have missed those emissions.

> Just like a radio station, if you tune in too late, you will have missed that song.    

逻辑上，一个热`Observable`代表的是不同的事件而不是无穷的数据集。事件上面带有数据，但是上面会有一个对时间敏感的组件，让后面的订阅者错失之前发送的数据。

举例来说，一个JavaFX或android UI事件就代表了一个热`Observable`。
``` java
public class JavaFXObservableDemo extends Application {
    @Override
    public void start(Stage primaryStage) throws Exception {
        ToggleButton toggleButton = new ToggleButton("Toggle Me");
        Label label = new Label();
        Observable<Boolean> selectedStates = valuesOf(toggleButton.selectedProperty());
        selectedStates.map(selected -> selected ? "DOWN" : "UP").subscribe(label::setText);
        VBox vBox = new VBox(toggleButton, label);
        primaryStage.setScene(new Scene(vBox));
        primaryStage.show();
    }

    private static <T> Observable<T> valuesOf(final ObservableValue<T> fxObservable) {
        return Observable.create(observableEmitter -> {
           // emit the initial state
           observableEmitter.onNext(fxObservable.getValue());
           // emit value changes use a listener
            final ChangeListener<T> listener = ((observable, oldValue, 
                        newValue) -> observableEmitter.onNext(newValue));
            fxObservable.addListener(listener);
        });
    }
}
```
在这个例子中我们把一个使用create方法将ToggleButton的被选中的状态作为一个`Observable`。我们将被按钮被选择是否的`Boolean`类型转换`String`类型，表示按钮是否被选中（DOWN or UP）。然后使用一个`Observer`用Label显示这个按钮是否被选中的信息

每次我们按下按钮。`ToggleButton`被唤醒，`Observable<Boolean>`发送一个`true`或`false`的值表示选中的状态。然后被转换成字符串，最终让一个`Observer`去更改Label上显示的文本。

#### 1.3.3 ConnectableObservable
一个比较好用的热`Observable`形式是`ConnectableObservable`类。它接受一个`Observable`，也不管这个`Observable`是不是一个冷`Observable`，把它变成热的从而所有发射的数据项都被立即发送到所有的`Observer`那里。而要做到这个转换，只需要对`Observable`调用publish()方法即可，它会返回一个ConnectableObservable对象。

但是仅仅是`subscribe`并不代表开始发射，我们仍然需要调用`connect()`方法让它开始发射数据项。这个方法让你在发射数据前先准备好所有的`Observer`。
``` java
public class ConnectableObservableExample {
    public static void main(String[] args) {
        ConnectableObservable<String> source = Observable.just(
                "Alpha", "Beta", "Gamma"
        ).publish();

        // Set up Observer 1
        source.subscribe(s -> System.out.println("Observer 1: " + s));
        // Set up Observer 2
        source.map(String::length)
                .subscribe(integer -> System.out.println("Observer 2: " + integer));
        // Fire
        source.connect();
    }
}

// 运行结果：
// Observer 1: Alpha
// Observer 2: 5
// Observer 1: Beta
// Observer 2: 4
// Observer 1: Gamma
// Observer 2: 5
```
从上面的例子中我们看到，两个`Observer`是同时接受到`Observable`发送的数据，而不是`Observer`1在`Observer2`之前先处理完数据，`Observer2`再开始处理数据。                    
> Using ConnectableObservable to force each emission to go to all observers is known as multicasting.
(multicast: send (data) across a computer network to several users at the same time)也就是多播

而且，如果在`connect()`调用之后的新的订阅会错失之前发射的数据。

### 1.4 其他创建`Observable`源的方法
- `range()`
- `interval()`
- `empty()`
- `never()`

### 1.5 Single，Completable，和Maybe
`Observable`有其他不同版本的实现从而使得它能够只发射一次或是不发射：
- `Single`
- `Completable`
- `Maybe`

### 1.5.1 Single
Single类其实本质上是只发射一次的`Observable`，而且也只适用于对一次发射合理的操作。和`Observable`实现了`ObservableSource`一样，`Single`类也实现了`SingleSource`功能接口，这个接口也只有一个方法，就是`subscribe(SingleObserver observer)`:
``` java
interface SingleObserver<T> {
            void onSubscribe(@NonNull Disposable d);
            void onSuccess(T value);
            void onError(@NonNull Throwable error);                                        
}
```
和`Observer`不同，`SingleObserver`并没有`onNext()`方法而是有一个和`onComplete()`方法对应的`onSuccess()`方法。这是因为它最多只能发射一次数据。这个`onSuccess()`方法实际上是整合了`onNext`和`onComplete`方法到一个事件中去，当你在一个`Single`对象上调用`subscribe()`方法时，你只需要为`onSuccess`提供一个lambda表达式以及一个可选的处理`onError`的lambda表达式。
``` java
public class SingleExample {
    public static void main(String[] args) {
        Single.just("Hello!").map(String::length).subscribe(System.out::println,
                throwable -> System.out.println("Error captured: " + throwable));
    }
}
```
`Single`和`Observable`之间也可以互相转换；你可以使用`toObservable()`方法来将Single转换为`Observable`。相反地，一些`Observable`操作也会返回Single，例如`first()`方法就是这样，因为它只关心单一的数据项。

#### 1.5.2 Maybe
除了`Maybe`完全不允许发射数据外，它和`Single`很像。`MaybeObserver`也和普通的`Observer`很像，但是`onNext()`变成了`onSuccess()`。
``` java
public interface MaybeObserver<T> {
              void onSubscribe(@NonNull Disposable d);
              void onSuccess(T value);
              void onError(@NonNull Throwable e);
              void onComplete();                                        
}
```
一个给定的`Maybe`对象发射0个或1个数据项，它会将可能的发射数据传递给`onSuccess()`，不管发射0个还是1个，最终也会调用`onComplete()`方法。`Maybe.just()`创建一个只发射一个数据项的`Maybe`， 而`Maybe.empty()`创建发射0个数据项的`Maybe`。
``` java
public class MaybeExample1 {
    public static void main(String[] args) {
        // has emission
        Maybe<Integer> source = Maybe.just(100);
        source.subscribe(s -> System.out.println("Process 1: " + s),
                e -> System.out.println("Error captured: " + e),
                () -> System.out.println("Process 1 done!"));

        // no emission
        Maybe<Integer> empty = Maybe.empty();
        empty.subscribe(s -> System.out.println("Process 2: " + s),
                e -> System.out.println("Error captured: " + e),
                () -> System.out.println("Process 2 done!"));
    }
}
// 运行结果：
// Process 1: 100
// Process 2 done!
```
第一个没有打印Process 1 done!是因为`Maybe`这个`Observable`只能发射不超过1个数据项。而且`MaybeObserver`也不会期待其他的东西。我们将其中的Maybe替换为`Observable`看看：
``` java
public class MaybeExample2 {
    public static void main(String[] args) {
        // has emission
        Observable<Integer> source = Observable.just(100);
        source.subscribe(s -> System.out.println("Process 1: " + s),
                e -> System.out.println("Error captured: " + e),
                () -> System.out.println("Process 1 done!"));

        // no emission
        Observable<Integer> empty = Observable.empty();
        empty.subscribe(s -> System.out.println("Process 2: " + s),
                e -> System.out.println("Error captured: " + e),
                () -> System.out.println("Process 2 done!"));
    }
}
// 运行结果：
// Process 1: 100
// Process 1 done!
// Process 2 done!
```
现在`onComplete()`方法被调用了，因为`Observer`仍然期待别的东西过来。

#### 1.5.3 Completable
`Completable`只是简单地关心被执行的一个动作，而且它不会收到任何发射的数据。逻辑上说，它没有`onNext()`和`onSuccess()`来接受发射的数据，但是它有`onError()`和`onComplete()`方法：
``` java
interface CompletableObserver<T> {
            void onSubscribe(@NonNull Disposable d);
            void onComplete();
            void onError(@NonNull Throwable error);
}
```
`Completable`不会是你常用的一个接口。你可以使用`Completable.complete()`或者是`Completable.fromRunnable()`来很快地创建一个对象；第一个创建之后啥也不做立即调用`onComplete`，第二个在调用`onComplete`之前会执行特定的动作：
``` java
public class Ch2_32 {
    public static void main(String[] args) {
        Completable.fromRunnable(() -> runProcess())
                .subscribe(() -> System.out.println("Done!"));
    }
    
    private static void runProcess() {
        System.out.println("Hello!");
    }
}
// 运行结果：
// Hello!
// Done!
```
### 1.6 销毁
当我们在链上传输完数据之后，我们当然想要销毁这些创建的资源，从而使它们能够被垃圾回收。一般地，那些不停发送数据的`Observable`在调用`onComplete()`方法之后就会自动销毁。但是，有些情况下，我们也需要显示地调用销毁的方法使其被回收，来防止可能的内存泄露。

`Disposable`接口是在`Observable`和一个活跃的`Observer`之间的一条链接。我们可以通过调用`dispose()`方法来结束发射，并且销毁回收所有`Observer`用的资源。它也有一个`isDisposed()`方法来检查是否被销毁。 
``` java                          
public interface Disposable {
      void dispose();
      boolean isDisposed();                                     
}
```
其实，`Observable`的接受lambda表达式作为参数的`subscribe()`方法最终返回的也是一个`Disposable`。从而使用者可以调用其中的`dispose()`方法结束数据的发送。
``` java
public class DisposableExample {
    public static void main(String[] args) throws InterruptedException {
        Observable<Long> seconds = Observable.interval(1, TimeUnit.SECONDS);
        Disposable disposable = seconds.subscribe(s -> System.out.println("Received: " + s));

        // sleep after 5 seconds
        sleep(5000);
        // dispose and stop emission
        disposable.dispose();
        // sleep 5 seconds to prove
        // there are no more emissions
        sleep(5000);
    }
}

// 运行结果：
// Received: 0
// Received: 1
// Received: 2
// Received: 3
// Received: 4
```

#### 1.6.1 在`Observer`里处理`Disposable`
之前我们在讲到`Observer`时跳过了它的`onSubscribe()`方法，我们注意到这个方法中的参数正好是一个`Disposable`。这个方法是在RxJava 2.0中才有的，使得`Observer`能够在任何时刻销毁订阅。

比如，你可以在`Observer`的`onNext`、`onComplete`和`onError`中使用这个`disposable`。这样的话，在我们需要的时候就可以中断`Observer`数据的接收：         
``` java                           
Observer<Integer> myObserver = new Observer<Integer>() {
    private Disposable disposable;

    @Override
    public void onSubscribe(Disposable disposable) {
        this.disposable = disposable;
    }                      
    @Override
    public void onNext(Integer value) {
        // has access to Disposable
    }
                                        
    @Override
    public void onError(Throwable e) {      
        //has access to Disposable
    }
                                        
    @Override
    public void onComplete() {                           
        //has access to Disposable
    }                          
};
```
如果你不想显式地去处理`Disposable`并且想让RxJava去帮你处理，也可以使用`ResourceObserver`作为你的`Observer`，这个类包含默认的`Disposable`处理逻辑，将这个类的对象传给`subscribeWith()`方法而不是`subscribe()`方法，之后你可以拿到返回的一个默认的`Disposable`。
``` java
public class ResourceObserverExample {
    public static void main(String[] args) {
        Observable<Long> source = Observable.interval(1, TimeUnit.SECONDS);
        ResourceObserver<Long> myObserver = new ResourceObserver<Long>() {
            @Override
            public void onNext(@NonNull Long aLong) {
                System.out.println(aLong);
            }

            @Override
            public void onError(@NonNull Throwable e) {
                e.printStackTrace();
            }

            @Override
            public void onComplete() {
                System.out.println("Done!");
            }
        };
        // capture Disposable
        Disposable disposable = source.subscribeWith(myObserver);
    }
}
```

#### 1.6.2 使用`CompositeDisposable`
当我们有多个需要被销毁的订阅时，用`CompositeDisposable`会很方便，它实现了`Disposable`接口，但其实内部是`Disposable`对象的集合。通过在集合中增减对象，你就可以立即将它们全部销毁。
``` java
public class CompositeDisposableExample {
    public static final CompositeDisposable disposables = new CompositeDisposable();

    public static void main(String[] args) throws InterruptedException {
        Observable<Long> source = Observable.interval(1, TimeUnit.SECONDS);
        // subscribe and capture disposables
        Disposable disposable1 = source.subscribe(l -> System.out.println("Observer 1: " + l));
        Disposable disposable2 = source.subscribe(l -> System.out.println("Observer 2: " + l));
        disposables.addAll(disposable1, disposable2);
        sleep(5000);
        disposables.dispose();
        sleep(5000);
    }
}
```

#### 1.6.3 使用`Observable.create()`方法来处理销毁
如果你的`Observable.create()`方法返回的是一个长时间运行的或是无穷的`Observable`，那么最好经常检查`ObservableEmitter`的`isDisposed()`方法来看看是否还需要发出新的数据项。这样的话可以在没有活跃的订阅的情况下防止`Observable`去做一些无意义的工作。
``` java
public class HandleDisposalInCreateMethod {
    public static void main(String[] args) {
        Observable<Integer> source = Observable.create(emitter -> {
           try {
               for (int i = 0; i < 1000; i++) {
                   while (!emitter.isDisposed()) {
                       emitter.onNext(i);
                   }
                   if (emitter.isDisposed()) {
                       return;
                   }
               }
               emitter.onComplete();
           } catch (Throwable e) {
               emitter.onError(e);
           }
        });
    }
}
```