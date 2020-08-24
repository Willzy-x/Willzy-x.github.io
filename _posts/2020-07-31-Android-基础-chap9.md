---
layout:     post
title:      安卓基础知识
subtitle:   基于《第一行代码：Android》第9章（网络）
date:       2020-07-31
author:     HYC
header-img: img/android-large.png
catalog: true
tags:
    - Android
    - Java
    - OkHttp
    - JSON
    - XML
    - Network
    - Retrofit
    - Kotlin
---

## 9.1 WebView的用法
在布局文件中添加如下代码，添加WebView控件：
``` xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <WebView
        android:id="@+id/web_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</LinearLayout>
```

然后修改MainActivity中的代码：
``` java
public class MainActivity extends AppCompatActivity {

    @SuppressLint("SetJavaScriptEnabled")
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        WebView webView = (WebView) findViewById(R.id.web_view);
        webView.getSettings().setJavaScriptEnabled(true);
        webView.setWebViewClient(new WebViewClient());
        webView.loadUrl("http://wwww.baidu.com");
    }
}
```

我们可以调用webView的`getSettings()`方法去设置浏览器的熟悉，在这里我们使用`setJavaScriptEnabled(true)`打开浏览器对JavaScript脚本的支持。

然后调用了WebView的`setWebViewClient()`方法，并传入了一个WebViewClient的实例。作用是当需要从一个网页跳转到另一个网页的时候，我们希望目标网页仍然在当前的WebView中显示，而不是打开系统的浏览器。

最后一步就是调用WebView的`loadUrl()`方法将网址传入。

除此之外，因为本程序使用到了网络的功能，访问的网络的权限需要在manifest文件中声明：
``` xml
<uses-permission android:name="android.permission.INTERNET" />
```

## 9.2 使用HTTP协议访问网络

### 9.2.1 使用HttpURLConnection

主要功能是在android发送HTTP请求。
  1. new一个URL对象，传入目标的网络地址
  2. 调用url对象的`openConnection`方法，获取实例

``` java
URL url = new URL("https://baidu.com");
HttpURLConnection connection = (HttpURLConnection) url.openConnection();
```

拿到了这个connection之后，我们也可以设置一下HTTP请求所使用的方法。常用的方法主要有两个：GET和POST：
``` java
connection.setRequestMethod("GET"); // 表示从服务器端拿数据
```

然后也可以根据自己的情况编写设置：
``` java
connection.setConnectTimeout(8000);
connection.setReadTimeout(8000);
```

然后调用`getInputStream()`方法可以获取到服务器返回的输入流了，剩下的任务就是对输入流进行读取，如下所示：
``` java
InputStream in = connection.getInputStream();
```

最后调用disconnect将链接关闭：
``` java
connection.disconnect();
```

如果想要提交数据给服务器呢，将请求方法改成POST，将数据输入到一个输出流中去，其中不同的参数用&隔开：
``` java
connection.setRequestMethod("POST");
DataOutputStream out = new DataOutputStream(connection.getOutputStream());
out.writeBytes("username=admin&password=123456");
```

### 9.2.2 使用OkHttp
第一件事还是添加依赖：
``` groovy
implementation("com.squareup.okhttp3:okhttp:4.8.0")
```

然后android studio会自动下载两个库，一个是OkHttp库，一个是Okio库，后者是通信的基础。

然后是具体用法：
  1. 首先创建一个`OkHttpClient`的实例
  2. 创建一个`Request`对象，准备发起请求
  3. 对请求对象进行设置，比如通过`url()`方法来设置目标的网络地址
  4. 之后调用`OkHttpClient`的`newCall()`方法来创建一个`Call`对象，并调用其`execute()`方法来发送请求并获取服务器返回的数据
  5. 从返回的`Response`对象数据中获取具体内容

``` java
private void sendRequestWithOkHttp() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                OkHttpClient client = new OkHttpClient();
                Request request = new Request.Builder().url("https://baidu.com").build();
                Response response = client.newCall(request).execute();
                String responseData = response.body().string();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }).start();
}
```

如果是发起一条POST请求，那么会比GET请求稍微复杂一点，我们需要构建出一个`Request Body`对象来存放待提交的参数，然后再放入`request`中，如下：
``` java
RequestBody requestBody = new FormBody.Builder()
        .add("username", "admin")
        .build();
Request request = new Request.Builder()
        .url("https://baidu.com")
        .post(requestBody)
        .build();
```

## 9.3 解析XML格式数据

在网络上传递数据时最常使用的格式有两种：XML和JSON格式，下面首先学习XML格式数据。

我们首先需要一个web服务器，具体配置Mac自带的Apache Server参考了：
配置Apache服务器

放在服务器上的XML数据：
``` xml
<apps>
<app>
<id>1</id>
<name>Google Maps</name>
<version>1.0</version>
</app>
<app>
<id>2</id>
<name>Chrome</name>
<version>2.1</version>
</app>
<app>
<id>3</id>
<name>Google Play</name>
<version>2.3</version>
</app>
</apps>
```

### 9.3.1 Pull解析方式
解析XML数据的主要流程：
``` java
private void parseXMLWithPull(String xmlData) {
    try {
        XmlPullParserFactory factory = XmlPullParserFactory.newInstance();
        XmlPullParser xmlPullParser = factory.newPullParser();
        xmlPullParser.setInput(new StringReader(xmlData));
        int eventType = xmlPullParser.getEventType();
        String id = "";
        String name = "";
        String version = "";
        while (eventType != XmlPullParser.END_DOCUMENT) {
            String nodeName = xmlPullParser.getName();
            switch (eventType) {
                case XmlPullParser.START_TAG : {
                    if ("id".equals(nodeName)) {
                        id = xmlPullParser.nextText();
                    } else if ("name".equals(nodeName)) {
                        name = xmlPullParser.nextText();
                    } else if ("version".equals(nodeName)) {
                        version = xmlPullParser.nextText();
                    }
                    break;
                }
                case XmlPullParser.END_TAG : {
                    if ("app".equals(nodeName)) {
                        Log.d(TAG, "id is " + id);
                        Log.d(TAG, "name is " + name);
                        Log.d(TAG, "version is " + version);
                    }
                    break;
                }
                default:
                    break;
            }
            eventType = xmlPullParser.next();
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

1. 借助`XmlPullParserFactory`的实例拿到一个`XmlPullParser`的对象
2. 调用`setInput()`方法将服务器返回的XML数据设置进去就可以开始解析了
3. 解析过程：
  1. `getEventType()`可以得到当前的解析事件
  2. 通过`getName()`方法获取当前节点的名字，如果节点名为当前要读取的那个数据，就调用`nextText()`方法获取节点中的内容
  3. 然后在`while`循环中不停迭代解析，使用`next`方法获取下一个解析事件
  4. 直到解析事件为`XmlPullParser.END_DOCUMENT`

### 9.3.2 SAX解析方式
SAX比Pull稍微复杂一点，但是在语义方面会更加清楚。

我们新建一个类继承自`DefaultHandler`，并重写父类的5个方法，如下所示：
``` java
public class MyHandler extends DefaultHandler {

    private String nodeName;
    private StringBuilder id;
    private StringBuilder name;
    private StringBuilder version;

    public static final String TAG = "ContentHandler";

    @Override
    public void startDocument() throws SAXException {
        this.id = new StringBuilder();
        this.name = new StringBuilder();
        this.version = new StringBuilder();
    }

    @Override
    public void endDocument() throws SAXException {
        super.endDocument();
    }

    @Override
    public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
        this.nodeName = localName;
    }

    @Override
    public void endElement(String uri, String localName, String qName) throws SAXException {
        if ("app".equals(nodeName)) {
            Log.d(TAG, "endElement: id is " + id.toString().trim());
            Log.d(TAG, "endElement: name is " + name.toString().trim());
            Log.d(TAG, "endElement: version is " + version.toString().trim());
            id.setLength(0);
            name.setLength(0);
            version.setLength(0);
        }
    }

    @Override
    public void characters(char[] ch, int start, int length) throws SAXException {
        if ("id".equals(nodeName)) {
            id.append(ch, start, length);
        } else if ("name".equals(nodeName)) {
            name.append(ch, start, length);
        } else if ("version".equals(nodeName)) {
            version.append(ch, start, length);
        }
    }
}
```

- `startDocument()`会在开始解析XML的时候调用
- `startElement()`会在开始解析某个节点的时候调用
- `characters()`方法会在获取节点中内容的时候调用
- `endElement()`会在完成解析某个节点的时候调用
- `endDocument()`会在完成整个XML解析的时候调用

然后在MainActivity中修改代码：
``` java
private void parseXMLWithSAX(String xmlData) {
    try {
        SAXParserFactory factory = SAXParserFactory.newInstance();
        XMLReader xmlReader = factory.newSAXParser().getXMLReader();
        ContentHandler handler = new ContentHandler();
        xmlReader.setContentHandler(handler);
        xmlReader.parse(new InputSource(new StringReader(xmlData)));
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

## 9.4 解析JSON格式数据

在Web服务器上放的JSON数据：
``` json
[
    {"id":"5","version":"5.5","name":"Clash of Clans"},
    {"id":"6","version":"7.0","name":"Boom Beach"},
    {"id":"7","version":"3.5","name":"Clash Royale"}
]
```

比起XML，JSON的优势主要就在它的体积更小，在网络上传输可以减少更多的流量。缺点在于，它的语义较差，看起来不如XML直观。

解析JSON既可以用JSONObject，也可以用谷歌的开源库GSON。第三方开源库如Jackson、FastJSON等也非常不错。

### 9.4.1 使用JSONObject
修改MainActivity的代码：
``` java
private void parseJSONWithJSONObject(String jsonData) {
    try {
        JSONArray jsonArray = new JSONArray(jsonData);
        for (int i = 0; i < jsonArray.length(); ++i) {
            JSONObject jsonObject = jsonArray.getJSONObject(i);
            String id = jsonObject.getString("id");
            String name = jsonObject.getString("name");
            String version = jsonObject.getString("version");
            Log.d(TAG, "parseJSONWithJSONObject: id is " + id);
            Log.d(TAG, "parseJSONWithJSONObject: name is " + name);
            Log.d(TAG, "parseJSONWithJSONObject: version is " + version);
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

由于我们在服务器中定义的是一个JSON数组，首先将其传入一个JSON数组，因此这里首先是将服务器返回的数据传入到了一个`JSONArray`对象中。然后循环遍历这个`JSONArray`，从中取出的每一个元素都是一个`JSONObject`对象，每个`JSONObject`对象中又会包含id、name和version这些数据。接下来只需要调用`getString()`方法将这些数据取出。

### 9.4.2 使用GSON
首先还是得添加依赖：
``` groovy
implementation 'com.google.code.gson:gson:2.8.6'
```

GSON主要就是可以将一段JSON格式的字符串自动映射成一个对象，从而不需要我们再动手去编写代码进行解析了。

比如有一个JSON格式的数据如下：
``` json
{"name": "Tom", "age":20}
```

我们可以定义一个`Person`类，并加入name和age两个字段，然后只需要简单地调用如下代码就可以将JSON数据自动解析成一个`Person`对象了：
``` java
Gson gson = new Gson();
Person person = gson.fromJson(jsonData, Person.class);
```

如果需要解析的是一段JSON数组会变得稍微麻烦一点，我们需要借助`TypeToken`将期望解析成的数据类型传入到`fromJson()`方法中，如下所示：
``` java
List<Person> people = gson.fromJson(jsonData, new TypeToken<List<Person>>() {}.getType());
```

基本用法就是这样，修改MainActivity中的代码，我们也可以解析JSON数据：
``` java
private void parseJSONWithGSON(String jsonData) {
    Gson gson = new Gson();
    List<App> appList = gson.fromJson(jsonData, new TypeToken<List<App>>(){}.getType());
    for (App app : appList) {
        Log.d(TAG, "parseJSONWithGSON: id is " + app.getId());
        Log.d(TAG, "parseJSONWithGSON: name is " + app.getName());
        Log.d(TAG, "parseJSONWithGSON: version is " + app.getVersion());
    }
}
```

### 9.5 Retrofit网络库补充
首先添加gradle依赖：
``` groovy
implementation 'com.squareup.retrofit2:retrofit:2.5.0'
```

定义一个网络请求接口：
``` kotlin
interface FlickrApi {
    /**
     *  By default, all retrofit web requests return a Call object,
     *  which represent a single web request that you can execute.
     *  Executing a Call produces one corresponding web response.
     *  Specifying Call<String> means that you want the response deserialized
     *  into a String object instead.
     *
     */
    @GET("/")
    fun fetchContents() : Call<String>
}
```

初始化Retrofit：
``` kotlin
// create Retrofit instance
val retrofit: Retrofit =
    Retrofit.Builder()
        .baseUrl("https://www.flickr.com/")
        .build()
// instantiate a anonymous class that implements the interface
val flickrApi: FlickrApi = retrofit.create(FlickrApi::class.java)
```

这里`retrofit`使用了动态代理的设计模式来创建实现FlickrApi接口的匿名实现类。我阅读了一下Retrofit的源码，Retrofit会将动态代理的方法放到一个cache里面，以增加使用的效率，如果cache里面没有相应的方法，Retrofit会让`HttpServiceMethod`去实现这个方法。根据接口上的注解判断Api实现的不同的请求：
``` java
private void parseMethodAnnotation(Annotation annotation) {
  if (annotation instanceof DELETE) {
    parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
  } else if (annotation instanceof GET) {
    parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
  } else if (annotation instanceof HEAD) {
    parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
  } else if (annotation instanceof PATCH) {
    parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
  } else if (annotation instanceof POST) {
    parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
  } else if (annotation instanceof PUT) {
    parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
  } else if (annotation instanceof OPTIONS) {
    parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
  } else if (annotation instanceof HTTP) {
    HTTP http = (HTTP) annotation;
    parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
  } else if (annotation instanceof retrofit2.http.Headers) {
    String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
    if (headersToParse.length == 0) {
      throw methodError(method, "@Headers annotation is empty.");
    }
    headers = parseHeaders(headersToParse);
  } else if (annotation instanceof Multipart) {
    if (isFormEncoded) {
      throw methodError(method, "Only one encoding annotation is allowed.");
    }
    isMultipart = true;
  } else if (annotation instanceof FormUrlEncoded) {
    if (isMultipart) {
      throw methodError(method, "Only one encoding annotation is allowed.");
    }
    isFormEncoded = true;
  }
}
```

然后我们通过创建的接口实现类去创建一个web请求，生成的请求是`Call`类型，你可以在未来任何的时候执行它。除此之外，还有有一个好处就是把耗时的HTTP请求-响应放到后台的线程中去执行了。
``` kotlin
/* Call represents a web request, and you can execute it at any time in the future */
val flickrHomePageRequest: Call<String> = flickrApi.fetchContents()
```

如果我们想要执行这个web请求，我们需要调用它的`enqueue()`方法，并且将除了响应成功和失败的回调函数对象`Callback`作为参数传递给它：

``` kotlin
/* To execute the web request represented by the Call object, call the enqueue() function and pass an instance of Callback to handle success and failure situations */
flickrHomePageRequest.enqueue(object : Callback<String> {
    override fun onFailure(call: Call<String>, t: Throwable) {
        Log.e(TAG, "Failed to fetch photos ", t )
    }

    override fun onResponse(call: Call<String>, response: Response<String>) {
        Log.d(TAG, "Response received: ${response.body()}")
    }
})
```