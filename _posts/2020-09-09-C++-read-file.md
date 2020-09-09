---
layout:     post
title:      C++笔记之文件操作
subtitle:   C++文件读取
date:       2020-09-09
author:     HYC
header-img: img/codes.jpeg
catalog: true
tags:
    - C++
    - IO
---

## 1. `std::ifstream`


### 1.1 抛出异常

通过对ifstream对象调用exceptions方法，设置错误的掩码。从而根据抛出的failure类型异常决定错误状态。这个函数最后返回的也是io异常掩码。

``` cpp
vShaderFile.exceptions (std::ifstream::failbit, std::ifstream::badbid);
```

- `std::ifstream::badbit`表示：不可恢复的流错误

- `std::ifstream::failbit`表示：输入/输出操作失败（格式或提取错误）



- `std::ifstream::goodbit`表示：没有错误
- `std::ifstream::eofbit`表示：相关的输入序列到达文件结尾



### 1.2 打开文件

通过调用对象的open方法，我们可以打开相应的文件。这个方法有4个重载，分别是：

```cpp
void open( const char *filename,
           ios_base::openmode mode = ios_base::in );

// (since C++17)
void open( const std::filesystem::path::value_type *filename,
           ios_base::openmode mode = ios_base::in );

// (since C++11)
void open( const std::string &filename,                                  
           ios_base::openmode mode = ios_base::in );

//(since C++17)
void open( const std::filesystem::path &filename,                                  
           ios_base::openmode mode = ios_base::in );
```

我们使用第一个方法就行，参数选择默认提供`ios_base::in`即可，因为我们只需要将文件读入，不需要将其写入。



当打开失败时，会调用`setstate(failbit)`，具体用法：

``` cpp
		// 检测文件流的错误状态
		if (!stream.fail()) {
        std::cout << "stream is not fail\n";
    }
 		// 人为将状态设置为failbit
    stream.setstate(std::ios_base::failbit);
 
    if (stream.fail()) {
        std::cout << "now stream is fail\n";
    }
```



当打开成功时，会调用`clear()`方法：

``` cpp
// 调用后默认将状态设置为goodbit
void clear( std::ios_base::iostate state = std::ios_base::goodbit );
```

如果`rdbuf()`返回的是空指针，那么就会将状态设置为`state | badbit`。也可能抛出异常。



### 1.3 读取数据

在打开文件成功之后，我们调用`rdbuf()`方法，返回一个指向原始文件的指针。我们把这个指针放入`stringstream`中。

`rdbuf`方法：

``` cpp
std::basic_filebuf<CharT, Traits>* rdbuf() const;
```

返回的类型是`std::basic_streambuf<Char T, Traits>`子类的指针，因此我们调用的`<<`是这个重载函数：

``` cpp
basic_ostream& operator<<( std::basic_streambuf<CharT, Traits>* sb);
```

最后的代码是这样的：

``` cpp
vShaderStream << vShaderFile.rdbuf();
```



### 1.4 关闭文件流

当我们完成读入文件中的数据之后，我们需要关闭这个文件。当然，如果过程当中出现了错误，也会调用`setstate(failbit)`将流的状态设置为fail状态。

``` cpp
void close();
```

然而，在我们把文件的数据流放入到`stringstream`之后，我们也需要将其转化为`string`类型。

``` cpp
std::string vertexCode = vShaderStream.str();
```

