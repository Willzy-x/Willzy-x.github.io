---
layout:     post
title:      OpenGL 学习笔记
subtitle:   OpenGL 中的缓冲对象
date:       2020-08-27
author:     HYC
header-img: img/cg_bg_eg.png
catalog: true
tags:
    - OpenGL
    - C++
    - GPU
---

OpenGL中有3种主要的数据对象：

- 顶点数组对象（VAO）
- 顶点缓冲对象（VBO）
- 索引缓冲对象（EBO）



## 1. VBO

在定义顶点数据后，我们需要VBO来管理这个内存，它会在GPU内存中储存大量的顶点。这样的话使用的好处就是我们可以**一次性地发送一大批数据到GPU上**，而不是每个顶点发送一次。如果一个顶点就发送一次会带来大量的开销，因为从CPU发送到GPU相对较慢。



像OpenGL中的其他对象一样，这个缓冲有一个独一无二的ID，因此我们可以调用`glGenBuffers`方法来生成一个VBO对象：

``` cpp
unsigned int VBO = glGenBuffers(1, &VBO);
```

通过打印一下这个VBO的ID，我们会发现一般第一次创建这个VBO时，ID都是从1开始的。



OpenGL有很多缓冲对象类型，VBO对象的**缓冲类型**是`GL_ARRAY_BUFFER`。而且只要这些缓冲对象是不同的类型，OpenGL允许我们同时绑定多个缓冲。我们可以使用`glBindBuffer`把新创建的缓冲绑定在`GL_ARRAY_BUFFER`。

``` cpp
glBindBuffer(GL_ARRAY_BUFFER, VBO);

void glBindBuffer(	GLenum 	target,
 					GLuint 	buffer);
```

而且，在绑定之后，对在GL_ARRAY_BUFFER目标上的调用都会用来配置当前的VBO。之后我们使用`glBufferData`函数，它会把之前定义的顶点数据复制到缓冲的内存中去：

``` cpp
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
```

这个函数是一个专门用来把用户定义的数据复制到当前绑定缓冲的函数。分别有四个参数：

``` cpp
void glBufferData(	GLenum  			target,
                    GLsizeiptr  		size,
                    const GLvoid *  	data,
                    GLenum  			usage);
```

- 第一个参数是target，是目标缓冲的类型：将VBO绑定到`GL_ARRAY_BUFFER`；
- 第二个参数是size，指定传输数据的大小（以字节为单位）；
- 第三个参数是data，也是就我们的顶点数据；
- 第四个参数是usage，告诉我们希望显卡如何管理给定的数据有三种形式：
  - `GL_STATIC_DRAW`：数据不会或几乎不会改变
  - `GL_DYNAMIC_DRAW`：数据会被改变很多
  - `GL_STREAM_DRAW`：数据每次绘制时都会改变



由于我们的位置数据不会改变，所以它的使用类型最好是`GL_STATIC_DRAW`。



在经过了上述的步骤后，我们已经将这些顶点数据存储在显存中了，同时我们使用VBO管理这些数据。



## 2. VAO

VAO可以像VBO那样被绑定，任何随后的顶点属性调用都会储存在这个VAO中。这样的好处就是，当配置顶点属性指针的时候，你只需要将那些调用执行一次。这样使得不同顶点数据和属性配置之间切换变得非常简单，只需要绑定不同的VAO就行了。



> OpenGL的`CORE_PROFILE`要求我们使用VAO。如果我们绑定VAO失败，OpenGL会拒绝绘制任何东西。



一个VAO会储存以下的这些内容：

- `glEnableVertexAttribArray`和`glDisableVertexAttribArray`的调用
- 通过`glVertexAttribPointer`设置的顶点属性配置
- 通过`glVertexAttribPointer`调用与顶点属性关联的VBO



创建一个VAO和创建一个VBO很类似：

``` cpp
unsigned int VAO = glGenVertexArrays(1, &VAO);
```



如果想要使用VAO，要做的是使用`glBindVertexArray`绑定VAO。从绑定之后起，我们应该绑定和配置对应的VBO和属性指针，之后解绑VAO供之后使用。

``` cpp
glBindVertexArray(VAO);
```



### 链接顶点属性

顶点着色器允许我指定任何以顶点属性为形式的输入，我们也需要手动指定输入数据的哪一个部分对应大的着色器的哪一个顶点属性。



我们需要将顶点在VBO中的存储信息告诉OpenGL以方便解析顶点数据：

``` cpp
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void *)0);
glEnableVertexAttribArray(0);
```

`glVertexAttribPointer`这个函数的参数非常多：

``` cpp
void glVertexAttribPointer(	GLuint  			index,
                            GLint  				size,
                            GLenum  			type,
                            GLboolean  			normalized,
                            GLsizei  			stride,
                            const GLvoid *      pointer);

```

- 第一个参数是index，也就是我们在顶点着色器中使用`layout (location = 0)`定义的顶点属性的**位置值**。因此在这里我们传入的是0。
- 第二个参数是size，指定一个**顶点的大小**，由于我们的顶点数据都是由一个大小为3的向量组成（分别代表x，y，z），所以这里顶点大小应该是3。
- 第三个参数是type，指定的**数据类型**，因为我们顶点的坐标都是浮点类型，所以这里我们填入`GL_FLOAT`。
- 第四个参数是normalized，表示我们是否希望我们的**顶点数据被标准化**。如果设置为`GL_TRUE`，那么所有的`signed`数据会被映射到-1到1之间，`unsigned`会被映射到0到1之间。
- 第五个参数是stride，步长，告诉OpenGL**相邻的两个顶点**之间间隔的大小。
- 第六个参数是pointer，表示位置数据在缓冲中起始的**偏移量**。



在告诉了OpenGL应该如何解释顶点数据之后，我们应该使用`glEnableVertexAttribArray`，来启用顶点属性。

``` cpp
glEnableVertexAttribArray(0);

void glEnableVertexAttribArray(GLuint  	index);
```





## 3. EBO

EBO其实就是利用索引的信息将我们传入的顶点进行重新组合，再用按照新的顺序组合的顶点进行绘制。比如我们有这些顶点，并且想用它们画出一个长方形：

``` cpp
float vertices[] = {
    // 第一个三角形
    0.5f, 0.5f, 0.0f,   // 右上角
    0.5f, -0.5f, 0.0f,  // 右下角
    -0.5f, 0.5f, 0.0f,  // 左上角
    // 第二个三角形
    0.5f, -0.5f, 0.0f,  // 右下角
    -0.5f, -0.5f, 0.0f, // 左下角
    -0.5f, 0.5f, 0.0f   // 左上角
};
```

那其实我们可以用右上角、右下角和左上角组成第一个三角形，然后右下角、左下角和左上角组成第二个三角形。这样最终在屏幕上显示的就是一个由两个三角形组成的长方形了。



根据上面提供的方案，我们找到这些顶点的在整个数组中的索引；就是0、1、3和1、2、3，然后我们将其组成一个索引数组：

``` cpp
unsigned int indices[] = { // 注意索引从0开始! 
    0, 1, 3, // 第一个三角形
    1, 2, 3  // 第二个三角形
};
```

这样我们就可以重复地去俩用顶点数据了，我们可以看到上面的索引1和3被使用了2次。



首先我们再来创建一个EBO对象：

``` cpp
unsigned int EBO = glGenBuffers(1, &EBO);
```

这一步和创建VBO相同，但是不同的是我们需要把EBO绑定到`GL_ELEMENT_ARRAY_BUFFER`上。然后用`glBufferData`把索引复制到缓冲里。

``` cpp
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);
```



最后要注意的就是我们要使用`glDrawElements`来替换`glDrawArrays`来绘制。

``` cpp
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);

void glDrawElements(GLenum  		mode,
                    GLsizei  		count,
                    GLenum  		type,
                    const GLvoid *  indices);
```

- 第一个参数是mode，指定了我们的绘制模式。
- 第二个参数是count，是我们打算绘制的顶点的个数，这里填6，因为我们要绘制6个顶点。
- 第三个参数是type，索引的类型，这里是`GL_UNSIGNED_INT`。
- 第四个参数是indices，指定EBO中的偏移量。



`glDrawElements`函数从当前绑定到`GL_ELEMENT_ARRAY_BUFFER`目标的EBO中获取索引。这意味我们必须在每次要用索引渲染一个物体时绑定相应的EBO，这还是有点麻烦。不过VAO同样可以保存EBO的绑定状态。



## Reference

1. https://learnopengl-cn.github.io/



## Appendix

最后附上我绘制三角形的代码：

``` cpp 
#include <glad.h>
#include <GLFW/glfw3.h>
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
#include <glm/gtc/type_ptr.hpp>

#include <iostream>

#include "shader_s.h"

void framebuffer_size_callback(GLFWwindow* window, int width, int height);
void processInput(GLFWwindow *window, unsigned int VAO=-1);

// settings
const unsigned int SCR_WIDTH = 800;
const unsigned int SCR_HEIGHT = 600;
float x_bias = 0.0f, y_bias = 0.0f;

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);

#ifdef __APPLE__
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
#endif

    GLFWwindow *window = glfwCreateWindow(SCR_WIDTH, SCR_HEIGHT, "title", NULL, NULL);

    if (window == NULL) {
        std::cerr << "Init window failed! " << std::endl;
        glfwTerminate();
        return -1;
    }

    glfwMakeContextCurrent(window);
    glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);

    if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress)) {
        std::cerr << "Init glad failed" << std::endl;
        return -1;
    }
    auto * shader = new Shader("./shaders/vertex_shader.txt", "./shaders/fragment_shader.txt");

    float vertices[] = {
            -0.5f, -0.5f, 0.0f, 0.0f, 1.0f, 0.0f,
            0.5f, -0.5f, 0.0f, 1.0f, 0.0f, 0.0f,
            0.0f, 0.5f, 0.0f, 0.0f, 0.0f, 1.0f
    };

    float triangle[] = {
            -0.5f, 0.5f, 0.0f,
            0.5f, 0.5f, 0.0f,
            0.0f, -0.5f, 0.0f
    };

    unsigned int VBO, VAO;
    glGenVertexArrays(1, &VAO);
    glGenBuffers(1, &VBO);

    glBindVertexArray(VAO);

    // 将VBO绑定到GL_ARRAY_BUFFER上
    glBindBuffer(GL_ARRAY_BUFFER, VBO);
    // 对在GL_ARRAY_BUFFER目标上的调用都会用来配置当前的VBO
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

    // 告诉opengl如何解析顶点属性
    // 0 for layout (location = 0)
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float ), (void *)nullptr);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float ), (void *) (3 * sizeof(float )));
    glEnableVertexAttribArray(1);

//    unsigned VBO2, VAO2;
//    glGenVertexArrays(1, &VAO2);
//    glGenBuffers(1, &VBO2);
//
//    glBindVertexArray(VAO2);
//
//    glBindBuffer(GL_ARRAY_BUFFER, VBO2);
//    glBufferData(GL_ARRAY_BUFFER, sizeof(triangle), triangle, GL_STATIC_DRAW);
//
//    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float ), (void *) nullptr);
//    glEnableVertexAttribArray(0);

    while (!glfwWindowShouldClose(window)) {
        processInput(window);
        shader->use();
        glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);
        // no need to bind VAO here since we only have 1 VAO
        glBindVertexArray(VAO);
        shader->setFloat("x_bias", x_bias);
        shader->setFloat("y_bias", y_bias);

        glDrawArrays(GL_TRIANGLES, 0, 3);
        glfwPollEvents();

        glfwSwapBuffers(window);
    }
    glfwTerminate();

    return 0;
}

// process all input: query GLFW whether relevant keys are pressed/released this frame and react accordingly
// ---------------------------------------------------------------------------------------------------------
void processInput(GLFWwindow *window, unsigned int VAO)
{
    if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS)
        glfwSetWindowShouldClose(window, true);

    if (glfwGetKey(window, GLFW_KEY_UP) == GLFW_PRESS) {
        y_bias += 0.01f;
    } else if (glfwGetKey(window, GLFW_KEY_DOWN) == GLFW_PRESS) {
        y_bias -= 0.01f;
    }

    if (glfwGetKey(window, GLFW_KEY_LEFT) == GLFW_PRESS) {
        x_bias -= 0.01f;
    } else if (glfwGetKey(window, GLFW_KEY_RIGHT) == GLFW_PRESS) {
        x_bias += 0.01f;
    }
}

// glfw: whenever the window size changed (by OS or user resize) this callback function executes
// ---------------------------------------------------------------------------------------------
void framebuffer_size_callback(GLFWwindow* window, int width, int height)
{
    // make sure the viewport matches the new window dimensions; note that width and
    // height will be significantly larger than specified on retina displays.
    glViewport(0, 0, width, height);
}
```




