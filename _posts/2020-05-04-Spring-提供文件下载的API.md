---
layout:     post
title:      Spring学习(1)
subtitle:   利用Spring MVC 提供能够进行文件上传下载的REST API
date:       2020-05-04
author:     HYC
header-img: img/post-2020-05-04-header.jpg
catalog: true
tags:
    - Java
    - Spring
    - Web
    - HTTP
---

最近正好在做软件项目管理大作业的项目开发，在实现大作业中文件上传下载的同时也可以学习一些Spring MVC的相关操作。

## 1. 文件下载

首先来看一下代码：

``` java
    /**
     * @param fileName
     * @param req
     * @param resp
     * @throws Exception
     */
    @GetMapping("/download/{fileName}")
    public void download(@PathVariable String fileName,
                         HttpServletRequest req, HttpServletResponse resp) throws Exception {
        try (InputStream inputStream = new FileInputStream(new File(filePath, fileName));
             OutputStream outputStream = resp.getOutputStream();){
            resp.setContentType("application/x-download");
            resp.addHeader("Content-Disposition", "attachment;filename=" + fileName);
            IOUtils.copy(inputStream, outputStream);
            outputStream.flush();
        }
    }
```

下面来对代码一行一行地进行讲解：

首先来看看这个注解：

### 1.1 `@GetMapping`注解传参

``` java  
    @GetMapping("/download/{fileName}")
```

其中的字符串自然指的就是当前所对应的URL。但是其中的fileName是什么意思呢？其实就是从URL中获取参数，正好对应函数`download`中的`@PathVariable String fileName`, 注解`@PathVariable`表示它是一个URL参数注解，而且参数名和URL中的参数要一致，这样我们就可以通过URL将参数传递过来了。

其实也是有其他的方式可以进行传递参数的方法，比如`@RequestParam`，它是一个请求参数的注解，value属性匹配前台传递的参数(一个参数时默认)，required属性此字段是否必须传值(boolean，默认为true)，defaultValue此参数的默认值(存在此参数时，说明前台不必需传递参数，required为false) 。

### 1.2 `HttpServletRequest`和`HttpServletResponse`

接下来我们来看看函数的参数列表中的`HttpServeletRequest`和`HttpServletResponse`。我们可以看到这两个接口其实都是继承自上层的`ServeletRequest`和`ServletResponse`接口。

而`HttpServletRequest`对象代表客户端的请求，当客户端通过HTTP协议访问服务器时，HTTP请求头中的所有信息都封装在这个对象中，通过这个对象提供的方法，可以获得客户端请求的所有信息。下面看看一个场景，通过客户端请求返回客户端发送请求时的完整URL。

``` java
    String requestUrl = request.getRequestURL().toString();// 得到请求的URL地址
    resp.setCharacterEncoding("UTF-8");// 设置将字符以"UTF-8"编码输出到客户端浏览器
    // 通过设置响应头控制浏览器以UTF-8的编码显示数据，如果不加这句话，那么浏览器显示的将是乱码
    resp.setHeader("content-type", "text/html;charset=UTF-8");
    PrintWriter out = resp.getWriter();
    out.write("请求的URL地址：" + requestUrl);
```

同时这个request也是一个域对象（Map容器），开发人员可以通过request对象在实现转发时，把数据通过request对象带给其它web资源处理。request对象作为一个域对象(Map容器)使用时，主要是通过以下的四个方法来操作：

- `Object getAttribute(String name)`
  - 获取request对象的name属性的属性值，例如：`req.getAttribute("data");`

- `void setAttribute(String name, Object o)`
  - 将数据作为request对象的一个属性存放到request对象中，例如：`req.setAttribute(“data”, data);`

- `void removeAttribute(String name)`
  - 移除request对象的name属性，例如：`req.removeAttribute(“data”);`


- `Enumeration<String> getAttributeNames()`
  - 获取request对象的所有属性名，返回的是一个`Enumeration`，例如：`Enumeration attrNames = req.getAttributeNames();`

`HttpServletResponse`它们中封装了向客户端发送数据、发送响应头，发送响应状态码的方法。常用的使用场景就是使用OutputStream流向客户端浏览器输出中文数据。在服务器端，数据是以哪个码表输出的，那么就要控制客户端浏览器以相应的码表打开。使用OutputStream流向客户端浏览器输出中文，以UTF-8的编码进行输出，此时就要控制客户端浏览器以UTF-8的编码打开，否则显示的时候就会出现中文乱码。那么在服务器端如何控制客户端浏览器以以UTF-8的编码显示数据呢？可以通过设置响应头控制浏览器的行为，例如：`response.setHeader("content-type", "text/html;charset=UTF-8");`通过设置响应头控制浏览器以UTF-8的编码显示数据。

``` java
    /**
     * 使用OutputStream流输出中文
     * @param request
     * @param response
     * @throws IOException
     */
    public void outputChineseByOutputStream(HttpServletResponse response) throws IOException{
        String data = "使用OutputStream流向浏览器输出中文";
        //获取OutputStream输出流
        OutputStream outputStream = response.getOutputStream();
        //通过设置响应头控制浏览器以UTF-8的编码显示数据，如果不加这句话，那么浏览器显示的将是乱码
        response.setHeader("content-type", "text/html;charset=UTF-8");
        //将字符转换成字节数组，指定以UTF-8编码进行转换
        byte[] dataByteArr = data.getBytes("UTF-8");
        //使用OutputStream流向客户端输出字节数组
        outputStream.write(dataByteArr);
    }
```

在代码中，我们使用了：

- `resp.setContentType("application/x-download)`
  - 设置响应头的`Content-Type`字段，比如如果是以utf-8编码的html文件，那么可以设置为`"text/html;charset=UTF-8"`，而我们设置的是`"application/x-download"`，表示的是这是告诉浏览器这是一个要保存到本地的下载的文件。但是具体浏览器是怎么处理的我们是不知道的，所以对这个首部内容的处理不同的浏览器有不同的处理方式，所以这种办法并不太可靠。鉴于以上的情况，我们可以使用一个Http首部来完成下载工作，这就是`Content-Disposition`，使用这个首部是告诉浏览器不要参与处理，而是要客户自己来处理该文件。具体在下面详细说明。但是其实未必一定要使用`x-download`，`x-msdownload`和`octet-stream`也可以解决，甚至只使用下面所说的`Content-Disposition`指名文件即可。

- `resp.addHeader("Content-Disposition", "attachment; fileName=" + fileName);`
  - 主要是在响应头部加一个字段，并且支持多值。`Content-Disposition` 是 MIME 协议的扩展，MIME 协议指示 MIME 用户代理如何显示附加的文件。`Content-Disposition`其实可以控制用户请求所得的内容存为一个文件的时候提供一个默认的文件名，文件直接在浏览器上显示或者在访问时弹出文件下载对话框。其中我们提供的文件名就是从URL中提取出的`fileName`。

综上所述我们给出比较好的使用办法，必须使用`Content-Disposition`来说明文件名和类型；另外最好使用`Content-Type:application/octet-stream`来说明`Content-Type`，这样也避免了我们对MIME类型陌生的问题，但是如果你熟悉MIME类型的话，最好还是指明该类型为好。

### 1.3 利用IOUtils.copy()

首先我们要使用`IOUtils`，需要在maven中添加依赖：

``` xml
    <dependency>
        <groupId>commons-io</groupId>
        <artifactId>commons-io</artifactId>
        <version>2.6</version>
    </dependency>
```

然后在需要的地方导入包就可以啦：

``` java
    import org.apache.commons.io.IOUtils;
```

`copy`方法的使用:

- `copy`能拷贝`Integer.MAX_VALUE`的字节数据，即$ 2^{31}-1 $。 
- 如果是很大的数据，那么可以选择用`copyLarge`方法，适合拷贝较大的数据流，比如2G以上 

在这里我们主要做的就是将从本地读取文件的`InputStream`拷贝到响应报文的`OutputStream`中去。

## 2. 文件上传

代码：

``` java
    static class FileInfo {
        private String filePath;

        public String getFilePath() {
            return filePath;
        }

        public void setFilePath(String filePath) {
            this.filePath = filePath;
        }

        public FileInfo(String _filePath) {
            this.filePath = _filePath;
        }
    }

    /**
     * @param fileName
     * @param file
     * @return
     * @throws Exception
     */
    @CrossOrigin
    @PostMapping("/upload/{fileName}")
    public FileInfo upload(@PathVariable String fileName,  MultipartFile file) throws Exception {
        File localFile = new File(filePath, fileName);
        file.transferTo(localFile);
        return new FileInfo(localFile.getAbsolutePath());
    }
```

其中定义了一个`FileInfo`的内部类来封装文件的一些信息，将其作为响应体中的一部分发送给客户端。这部分不加赘述。

### 2.1 `@CrossOrigin`注解

这部分其实未必需要，但是在测试的时候发现如果不加上这个注解，有时居然无法访问API。我去查阅了一下Spring的文档，发现这样一段话：

> Annotation for permitting cross-origin requests on specific handler classes and/or handler methods. Processed if an appropriate `HandlerMapping` is configured.

其实就是用来处理跨域（Cross-Origin）请求的注解，那么什么是跨域呢？

跨域，指的是浏览器不能执行其他网站的脚本。它是由浏览器的同源策略造成的，是浏览器施加的安全限制。

而所谓同源是指，域名，协议，端口均相同。

下面举一些🌰：

- http://ychu.top/index.html 调用 http://ychu.top/servers.php 是非跨域的。
- http://ychu.top/index.html 调用 https://ychu.top/servers.php 是跨域的，因为协议不同
- http://ychu.top:8080/ 调用 http://ychu.top:8081/index.html 是跨域的，因为端口不同。（我碰到的就是这种情况）
- 注意：localhost 和 127.0.0.1 虽然都指向本机，但是也属于跨域。

### 2.2 `MultipartFile`类

我也去看了一下Spring文档中的描述：

> A representation of an uploaded file received in a multipart request.  
> 
> The file contents are either stored in memory or temporarily on disk. In either case, the user is responsible for copying file contents to a session-level or persistent store as and if desired. The temporary storage will be cleared at the end of request processing.

说的很清楚，其实就是一般是用来接受前台传过来的文件的。我们在功能里主要使用了`transferTo`这个功能，来把它转成`File`类的对象。而这个`MultipartFile`对象一般要么在内存中要么会放在磁盘上。在这两种情况下，用户应该要把它存在一个类似session的结构中或者其他永久存储的地方，因为这个文件在请求结束时会被清除。

### 2.3 Postman 测试文件上传

之前用过Postman进行过一些简单的POST和GET请求操作，这次碰到文件上传不知道咋整了，顺便学习一下Postman对文件上传进行测试。

![postman](http://ychu.top/img/postman-fie-upload-test.png)

其实也挺简单的，就是在Body里面点选form-data，然后选择需要上传的文件即可。
