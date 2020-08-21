---
layout:     post
title:      安卓基础知识
subtitle:   基于《第一行代码：Android》1～2章
date:       2020-07-09
author:     HYC
header-img: img/android-large.png
catalog: true
tags:
    - Android
    - Java
---

## 1. 日志工具android.util.Log
日志工具，提供了以下5种方法来打印日志：
- `Log.v()`用于打印最为琐碎的、意义最小的信息，对应的级别为verbose，为android日志里等级最低的。
- `Log.d()`用于打印一些调试信息，对应级别debug，比verbose高一级。
- `Log.i()`用于打印一些比较重要的数据，对应级别info，比debug高一级。
- `Log.w()`用于打印一些警告信息，提示会有潜在的风险，对应级别为warn，比info高一级。
- `Log.e()`用于打印一些程序中的错误信息，比如进入了`catch`语句中。对应级别为error，比warn高一级。

这些方法中第一个参数是tag，一般传入当前的类名即可，主要用于对打印信息进行过滤。第二个参数是msg，即想要打印的具体内容。

### 1.1 为什么不用`System.out.println()`而使用Log
缺点太多了：日志打印不可控制，打印时间无法确定、不能添加过滤器、日志级别没有分级。。。
提示：最好在类中添加一个`TAG`常量值：
``` java
public class MainActivity extends AppCompatActivity {
          private static final String TAG = "MainActivity";
          Log.d(TAG, "message");
}
```

## 2. 活动（Activity）
活动是一种可以包含用户界面的组件，主要用于和用户进行交互。一个应用中可以包含零个或多个活动。

### 2.1 活动的基本用法
创建一个活动，在android studio中，右键包->New->Activity->Empty Activity，创建新的空活动。名字设置为`FirstActivity`。
``` java
public class FirstActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
    }
}
```
注意：项目中的任何活动都需要重写`onCreate()`方法，而目前我们的`FirstActivity`中已经重写了这个方法，这是由android studio自动帮我们完成的。

#### 2.1.1 创建和加载布局
android程序的设计讲究逻辑和视图相分离。在选定的layout包中，右键->New->Layout Resource File，弹出一个新建布局资源文件的窗口，我们将这个窗口命名为`first_layout`，根元素选定为`LinearLayout`。android studio为我们生成了以下代码：
``` xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" android:layout_width="match_parent"
    android:layout_height="match_parent">

</LinearLayout>
```
由于我们选择了`LinearLayout`作为根元素，因此现在布局文件中已经有一个LinearLayout元素了。我们对这个布局做一个调整，添加一个按钮：
``` xml
<Button
        android:id="@+id/button_1"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Button 1"
/>
```
添加了一个`Button`元素，并且在其中内部增加了几个属性。`android:id`是给当前元素定义一个唯一的标识符，之后可以在代码中对这些元素进行操作。
- `android:id="@+id/button_1"`其实就是XML中引用资源的方法，+表示这是定义一个id，而不是引用一个id。
- `android:layout_width`指定了当前元素的宽度，`match_parent`表示当前元素和父亲元素一样宽。
- `android:layout_height`指定了当前元素的高度，`wrap_content`表示当前元素的高度只要正好包含里面的内容就行。
- `android:text`指定了元素中显示的内容。

#### 2.1.2 在活动中载入布局
重新回到FirstActivity，在onCreate()下面加入如下代码：
`setContentView(R.layout.first_layout);`  
这里调用了setContentView()的方法来给当前的活动加载一个布局，通常，我们都会传入一个布局文件的id作为参数（项目中添加的任何资源都会在R文件中生成一个相应的资源id）。因此我们刚刚创建的first_layout.xml已经被添加到R文件中去了。通过`R.layout.first_layout`就能引用。

#### 2.1.3 在AndroidManifest文件中注册
所有的活动必须要在AndroidManifest.xml中进行注册才能生效，而实际上`FirstActivity`已经在其中进行注册过了，代码如下：
``` xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.activitytest">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".FirstActivity"></activity>
    </application>

</manifest>
```
我们可以看到，活动的注册都要放在`<application>`标签中，然后通过`<activity>`标签进行注册。
`<activity android:name=".FirstActivity"></activity>`
这其中的`android:name=".FirstActivity"`表示这个活动类，是`com.example.activitytest.FirstActivity`的缩写而已。

上面仅仅是注册了活动，我们的程序仍然不能运行，因为还没有为程序配置主活动，也就是说，不知道要先启动哪个活动。配置主活动的方法就是在`<activity>`中加入如下声明即可：
``` xml
<intent-filter>
    <action android:name="android.intent.action.MAIN"/>
    <category android:name="android.intent.category.LAUNCHER"/>
</intent-filter>
```
#### 2.1.4 在活动中使用Toast
`Toast`是android系统提供的一种非常好的提醒方式，在程序中可以使用它来提供一些短小的信息给用户，这些信息会在一段时间后消失，并且不会占用任何屏幕空间。

首先，我们需要定义一个`Toast`的触发点，正好我们刚刚设计了一个按钮，那就让我们在按下按钮的时候弹出一个`Toast`吧。在`onCreate()`中添加如下方法：
``` java
// 找到Button的引用
Button button1 = (Button)findViewById(R.id.button_1);
button1.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        Toast.makeText(FirstActivity.this, "You clicked Button 1",
            Toast.LENGTH_SHORT).show();
    }
});
```
在活动中，可以通过`findViewById()`的方法获取在布局文件中定义的元素，在这里我们传入`R.id.button_1`，来得到按钮的实例。

`Toast`的用法比较简单：
- `makeText()`创建出一个Toast对象，然后调用`show()`将Toast显示出来就可以了。其中我们需要传入3个参数：
  - `context`：是Toast要求的上下文，由于活动本身是一个Context对象，因此这里我们直接传入`FirstActivity.this`即可。
  - `text`：是Toast显示的文本内容。
  - `duration`：是Toast显示的时长，有两个内置常量可以选择`Toast.LENGTH_SHORT`和`Toast.LENGTH_LONG`。

#### 2.1.5 在活动中使用菜单
在res下面的menu包中，右键新建一个叫main的菜单文件：
``` xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:id="@+id/add_item"
        android:title="Add" />
    <item
        android:id="@+id/remove_item"
        android:title="Remove"/>
</menu>
```
其中`android:title`给这个菜单项指定一个名称。

接着我们来到`FirstActivity`中重写`onCreateOptionsMenu()`方法：
``` java
@Override
public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.main, menu);
    return true;
}
```
这里的`getMenuInflater()`方法能够得到`MenuInflater`对象，再调用`inflate()`方法就可以给当前活动创建菜单了。`inflate()`方法接受两个参数：
- 第一个参数指定我们通过哪一个资源文件创建菜单，这里传入`R.menu.main`。
- 第二个参数指定我们的菜单项将添加到哪个`Menu`对象中，这里直接使用传入的`menu`参数。
- 最后返回`true`，表示允许创建的菜单显示出来，如果返回了`false`，那么创建的菜单将无法显示。

让菜单显示出来是不够的，我们还有定义菜单的响应事件，在`FirstActivity`中重写`onOptionsItemSelected()`方法：
``` java
@Override
public boolean onOptionsItemSelected(@NonNull MenuItem item) {
    // 查看当前指向的菜单的条目是哪个
    switch (item.getItemId()) {
        case R.id.add_item:
            Toast.makeText(this, "You clicked Add", Toast.LENGTH_SHORT).show();
            break;
        case R.id.remove_item:
            Toast.makeText(this, "You clicked Remove", Toast.LENGTH_SHORT).show();
            break;
        default:
    }
    return true;
}
```
#### 2.1.6 销毁一个活动
只要按一下Back键就可以销毁当前的活动了。当然也可以通过代码来销毁活动，Activity类中提供了一个`finish()`方法，我们在活动中调用这个方法就可以销毁当前活动了。
``` java
button1.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        finish();
    }
});
```

### 2.2 使用Intent在活动之间穿梭
其实，在启动器中点击应用的图标只会进入该应用的主活动，那么怎样才能由主活动跳转到其他活动呢？
`Intent`是android程序中各组件进行交互的一种重要的方式，不仅可以指明想要执行的动作，也可以在不同的组件之间传输数据。

#### 2.2.1 使用显式的`Intent`
仍然选择新建一个`Activity`，名字为`SecondActivity`，创建布局文件second_layout.xml。布局仍然是`<LinearLayout>`。

Android studio也同样帮我们在manifest里面注册过了，由于这个活动不是主活动，因此不需要配置`<intent-filter>`中的内容。好了，剩下的事情就是要通过`Intent`来启动这个活动了。

而`Intent`大致可以分为两类：
- 显式`Intent`
- 隐式`Intent`

我们首先学习的是显式Intent，Intent有多个构造函数的重载，其中一个是：
``` java
Intent (Context packageContext, Class<?> cls)
```
- 第一个参数要求提供当前的启动活动的上下文。
- 第二个参数指定想要启动的目标活动，最后，通过这个`constructor`就可以构造出`Intent`的意图。

之后，我们想使用这个Intent的时候，就需要Activity提供的`startActivity()`方法，这个方法专门用于启动活动，它接受的参数是一个Intent，我们将构建好的Intent传入就可以启动目标活动了。

修改点击`Button 1`的代码，让它被点击时启动`SecondActivity`：
``` java
button1.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        Intent intent = new Intent(FirstActivity.this, SecondActivity.class);
        startActivity(intent);
    }
});
```

#### 2.2.2 使用隐式的Intent
隐式的Intent不明确指出我们想要启动哪一个活动，而是指定了一系列更为抽象的`action`和`category`等信息。

通过在activity下面配置`intent-filter`的内容，可以指定当前活动能够响应的`action`和`category`：
``` xml
<activity android:name=".SecondActivity">
    <intent-filter>
        <action android:name="com.example.activitytest.ACTION_START" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```
然后在`ClickListener`中调用Intent的另一个`constructor`：
``` java
Intent intent = new Intent("com.example.activitytest.ACTION_START");
```
就可以使用隐式调用了。

每个Intent只能指定一个`action`，但是却能指定多个`category`。目前我们的Intent中只有一个默认的`category`，我们可以通过如下的代码进行增加：
``` java
intent.addCategory("com.example.activitytest.MY_CATEGORY");
// 同样需要在manifest中进行配置，否则会报错：
```
``` xml
<category android:name="com.example.activitytest.MY_CATEGORY" />
```
这样也可以达到使用隐式Intent的目的。

#### 2.2.3 更多隐式Intent的用法
使用隐式Intent不仅可以启动自己应用程序内的活动，也可以启动其他应用程序中的活动。比如调用调用系统中的浏览器来展示一个网页：
``` java
button1.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setData(Uri.parse("http://baidu.com"));
        startActivity(intent);
    }
});
```
其中我们指定了Intent的动作是`Intent.ACTION_VIEW`，这是一个Android系统里的内置动作，其常量值为`android.intent.action.VIEW`。然后通过`Uri.parse()`方法，将这个网址解析为一个`Uri`对象，再调用Intent的`setData()`方法将这个`Uri`传递进去。

- 使用`setData()`方法时，我们还可以在`<intent-filter>`配置`<data>`标签，用于更精确地指定当前活动能够响应什么类型的数据。主要有一下几个内容可以配置：
  - `android:scheme` - 指定数据协议部分
  - `android:host` - 主机名
  - `android:port` - 主机端口号
  - `android:path` - 主机名和端口号之后的部分
  - `android:mineType` - 指定可以处理的数据类型
- 只有在`<data>`里面指定的内容和Intent中携带的Data一致时，当前活动才能响应Intent。


#### 2.2.4 向下一个活动传送数据
Intent除了可以用来启动一个活动，它也可以用来在启动活动的时候传递数据。

传递数据的思想比较简单，Intent提供了一系列的`putExtra()`方法的重载，可以把我们想要传输的数据暂存在Intent中，启动了之后，就把这个数据取出就行了。比如从FirstActivity中传递一个字符串到SecondActivity中：
``` java
button1.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        String data = "Hello Second Activity";
        Intent intent = new Intent(FirstActivity.this, SecondActivity.class);
        intent.putExtra("extra_data", data);
        startActivity(intent);
    }
});
```
然后在SecondActivity的`onCreate()`方法中将数据取出来：
``` java
Intent intent = getIntent();
String data = intent.getStringExtra("extra_data");
Log.d("SecondActivity", data);
```

#### 2.2.5 返回数据给上一个活动
我们可以使用Activity提供的`startActivityForResult()`方法用于启动活动然后在活动销毁的时候能够返回一个结果给上一个活动。

- `startActivityForResult()`方法有两个参数
  - 第一个参数是还是Intent。
  - 第二个参数是请求码，用于在之后的回调中判断数据的来源，在我们的demo里，这个只要是一个唯一值就可以了。
``` java
Intent intent = new Intent(FirstActivity.this, SecondActivity.class); 
startActivityForResult(intent, 1);
```

接下来，我们在`SecondActivity`中给按钮注册点击事件，并在点击事件中添加返回数据的逻辑：
``` java
Button button2 = (Button)findViewById(R.id.button_2);
button2.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        Intent intent = new Intent();
        intent.putExtra("data_return", "Hello FirstActivity");
        setResult(RESULT_OK, intent);
        finish();
    }
});
```
我们仍然构建了一个Intent，只不过这个Intent仅仅是用于传递数据而已，没有指定别的“意图”。之后调用`setResult()`方法。这个方法非常重要，是专门向上一个活动返回数据的。

- `setResult()`方法接受两个参数
  - 第一个参数用于向上一个活动返回处理结果，一般是`RESULT_OK和RESULT_CANCELED`。
  - 第二个参数则把带有数据的Intent传递回去，然后调用了`finish()`方法来销毁当前活动。

因为在SecondActivity被销毁之后会回调上一个活动的`onActivityResult()`方法，因此我们需要在FirstActivity中重写`onActivityResult()`方法来获得返回的数据：
``` java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    switch (requestCode) {
        case 1:
            if (resultCode == RESULT_OK) {
                String returnedData = data.getStringExtra("data_return");
                Log.d("FirstActivity", returnedData);
            }
            break;
        default:
    }
}
```
- `onActivityResult(int requestCode, int resultCode, Intent data)`方法接受3个参数：
  - 第一个参数`requestCode`，即我们在启动活动时传入的请求码。
  - 第二个参数`resultCode`，是返回数据时传入的结果。
  - 第三个参数`data`，即带着返回数据的Intent。

如果在上面的例子中，用户不是通过点击`Button 2`，而是通过点击Backward按键返回，那么我们这时候就应该重写SecondActivity中的`onBackPressed()`方法：
``` java
@Override
public void onBackPressed() {
    Intent intent = new Intent();
    intent.putExtra("data_return", "Hello First");
    setResult(RESULT_OK, intent);
    finish();
}
```

### 2.3 活动的生命周期

#### 2.3.1 返回栈
android是用任务（Task）来管理活动的，一个任务就是一组存放在栈里的活动的集合，这个栈也被称为返回栈（Back Stack）。
#### 2.3.2 活动状态
1. 运行状态
活动位于返回栈的栈顶，就处于运行状态，系统最不愿意回收处于这个状态的活动。

2. 暂停状态
活动不再处于桟顶位置，但仍然可见时，这时活动就进入了暂停状态。

3. 停止状态
活动不再处于栈顶位置，并且完全不可见的时候，就进入了停止状态。系统仍然会为这种状态保存相应的状态和成员变量，处于停止状态的活动有可能被系统回收。

4. 销毁状态
活动从返回栈中移除后就变成了销毁状态。系统最倾向回收处于这种状态的活动，从而保证手机的内存充足。
#### 2.3.3 活动的生存期
从活动的创建到活动的销毁，中间会经历许多的环节，下面按顺序列出这些环节：
1. `onCreate()`
2. `onStart()`
3. `onResume()`
4. `onPause()`
5. `onStop()`
6. `onDestroy()`
7. `onRestart()`

- 完整生存期 —— `onCreate`方法和`onDestroy`方法之间所经历的。分别管理初始化操作和释放内存的操作。
- 可见生存期 —— `onStart`方法和`onStop`方法之间所经历的。通过这两个方法，我们可以合理地管理那些对用户可见的资源。
- 前台生存期 —— `onResume`方法和`onPause`方法之间所经历的。此时的活动可以和用户进行交互。

#### 2.3.4 活动被回收了怎么办
之前说过，当一个活动进入停止状态，那么它是有可能被系统回收的，而且它其中还可能保存有一些状态和数据，如果就随意地回收了，那么是很影响用户的体验的。

所以，我还提供了一个`onSaveInstanceState()`的回调方法，这个方法保证在活动被回收之前一定会被调用，因此我们可以使用这个方法来解决活动被回收时数据得不到保存的问题。
- `onSaveInstanceState()`这个方法的参数：
  - 一个`Bundle`类型的参数，这个类提供了一系列的方法用于保存数据。

``` java
@Override
protected void onSaveInstanceState(@NonNull Bundle outState) {
    super.onSaveInstanceState(outState);
    String tempData = "Something you just typed";
    outState.putString("data_key", tempData);
}

// 接下来在MainActivity的onCreate方法中使用保存的数据

// 恢复数据
if (savedInstanceState != null) {
    String tempData = savedInstanceState.getString("data_key");
    Log.d(TAG, "onCreate: " + tempData);
}
```

### 2.4 活动的启动模式

#### 2.4.1 standard
`standard`模式是活动的默认启动模式，所有的模式都会自动使用这个启动模式。这个模式下，每启动一个新模式，它就会在返回栈中入栈，并处于栈顶的位置。而且不管这个活动是否已经存在于栈中，都会新建一个新的实例。
#### 2.4.2 singleTop
在启动活动时如果发现桟顶的已经是该活动，则认为可以直接使用它，不会再创建新的活动实例。但是它只会检测桟顶是否相同，而不会检测桟中的其他活动是否和当前要启动的元素相同。

在manifest的activity标签中配置即可：
`android:launchMode="singleTop"`

#### 2.4.3 singleTask
使用`singleTop`可以很好地解决重复创建相同的桟顶活动的问题，但是不能解决整个栈中存在重复的活动的问题。当活动的启动模式为singleTask的时候，每次启动该活动系统都会在返回栈中检测是否存在该活动的实例，如果存在则直接使用，并且把这个活动之上的其他活动都出栈。如果没有则创建新的实例。

#### 2.4.4 singleInstance
不同于以上3种模式，`singleInstance`会指定一个新的返回栈来管理这个活动。想象一种场景，假如我们的程序中有一个活动是运行其他程序调用的，如果我们希望实现其他程序和我们的程序可以共享这个活动的实例，应该如何实现呢？使用singleInstance就可以解决这个问题，因为在这种模式下，会有一个单独的返回栈来管理这个活动，不管是哪个应用程序来访问这个活动，都共用的同一个返回栈，也就解决来共享活动实例的问题。

### 2.5 活动的最佳实践

#### 2.5.1 知晓当前是哪一个活动
新建一个类`BaseActivity`继承`AppCompatActivity`类，在其中的`onCreate()`中加入代码：
``` java
Log.d("BaseActivity", getClass().getSimpleName());
```
然后让其他的活动不要继承`AppCompatActivity`，转而继承`BaseActivity`。

#### 2.5.2 随时随地推出程序
使用一个专门的集合类对所有的活动进行管理就可以了，新建一个`ActivityCollector`类作为活动管理器。
``` java
public class ActivityController {
    public static List<Activity> activities = new ArrayList<>();
    
    public static void addActivity(Activity activity) {
        activities.add(activity);
    }
    
    public static void removeActivity(Activity activity) {
        activities.remove(activity);
    }
    
    public static void finishAll() {
        for (Activity activity : activities) {
            if (!activity.isFinishing()) {
                activity.finish();
            }
        }
        activities.clear();
    }
}
```
最重要的是其中的`finishAll()`方法用于将`List`中存储的活动全部销毁掉。然后我们也需要在上面的`BaseActivity`中创建和销毁时分别加入以下代码：
``` java
// onCreate()
ActivityController.addActivity(this);

// onDestroy()
ActivityController.removeActivity(this);
```

除此之外，还可以在`finishAll()`后面加入以下代码，来保证程序完全退出：
``` java
android.os.Process.killProcess(android.os.Process.myPid());
```

#### 2.5.3 启动活动的最佳写法
启动活动时传参数时，我们最好加入一个`actionStart()`方法，在这个方法中完成来Intent的构建，将需要的参数通过这个方法传递过来，然后再储存到Intent之中，最后调用`startActivity()`方法启动活动：
``` java
public static void actionStart(Context context, String data1, String data2) {
    Intent intent = new Intent(context, SecondActivity.class);
    Intent.putExtra("param1", data1);
    Intent.putExtra("param2", data2);
    startActivity(intent);
}
```
通过编写这个方法可以很清楚地知道启动`SecondActivity`这个活动需要哪些参数。
