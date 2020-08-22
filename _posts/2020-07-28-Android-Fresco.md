---
layout:     post
title:      Android 图像加载
subtitle:   图像加载之Fresco
date:       2020-07-28
author:     HYC
header-img: img/android-large.png
catalog: true
tags:
    - Android
    - Fresco
---

## 1. Fresco中相关类的介绍
### 1.1 Image Pipeline
Fresco 中设计有一个叫做 Image Pipeline 的模块。它负责从网络，从本地文件系统，本地资源加载图片。为了最大限度节省空间和CPU时间，它含有3级缓存设计（2级内存，1级磁盘）。

特性包含允许使用很多种方式来自定义图片的加载过程，比如：
- 为同一个图片指定不同的远程路径，或者使用已经存在本地缓存中的图片
- 先显示一个低清晰度的图片，等高清图下载完之后再显示高清图
- 加载完成回调通知
- 对于本地图，如有EXIF缩略图，在大图加载完成之前，可先显示缩略图
- 缩放或者旋转图片
- 对已下载的图片再次处理
- 支持WebP解码，即使在早先对WebP支持不完善的Android系统上也能正常使用！

### 1.2 Drawee
Fresco 中设计有一个叫做 `Drawees` 模块，它会在图片加载完成前显示占位图，加载成功后自动替换为目标图片。当图片不再显示在屏幕上时，它会及时地释放内存和空间占用。

Fresco 的 Drawees 设计，带来一些有用的特性：
- 自定义居中焦点
- 圆角图，当然圆圈也行
- 下载失败之后，点击重现下载
- 自定义占位图，自定义overlay, 或者进度条
- 指定用户按压时的overlay

#### 1.2.1 `SimpleDraweeView`的使用
在layout文件中使用`SimpleDraweeView()`：

``` xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:fresco="http://schemas.android.com/apk/res-auto" >

    <com.facebook.drawee.view.SimpleDraweeView
        android:id="@+id/my_image_view"
        android:layout_width="130dp"
        android:layout_height="130dp"
        fresco:placeholderImage="@drawable/ic_launcher_background"/>

</LinearLayout>
```
其中的`placeholderImage`属性指的就是在将正式的图片加载出来之前所放置的占位符。除了这个占位符属性之外，我们也可以设置其他的属性：比如:
``` xml
fresco:progressBarImage="@drawable/icon_progress_bar"
fresco:progressBarImageScaleType="centerInside"
fresco:progressBarAutoRotateInterval="5000"
```
设置了加载时所防止的图片，加载图片的缩放类型和自动旋转的间隔时间，直到图片加载出来。
``` xml
fresco:failureImage="@mipmap/icon_failure"
fresco:failureImageScaleType="centerInside"
```
设置了图像无法加载出来所放置的图片，也即是所说的失败图。
``` xml
fresco:retryImage="@mipmap/icon_retry"
fresco:retryImageScaleType="centerCrop"
```
设置了重试图和重试图的缩放类型。

#### 1.2.2 使用`DraweeHierarchy`来在代码中设置`SimpleDraweeView`的属性

虽然推荐使用XML来进行配置，但是所有的属性也能通过代码来进行设置。你可以在指定图像的URI之前通过`DraweeHierarchy`来办到：
``` java
GenericDraweeHierarchy hierarchy =
    GenericDraweeHierarchyBuilder.newInstance(getResources())
        .setActualImageColorFilter(colorFilter)
        .setActualImageFocusPoint(focusPoint)
        .setActualImageScaleType(scaleType)
        .setBackground(background)
        .setDesiredAspectRatio(desiredAspectRatio)
        .setFadeDuration(fadeDuration)
        .setFailureImage(failureImage)
        .setFailureImageScaleType(scaleType)
        .setOverlays(overlays)
        .setPlaceholderImage(placeholderImage)
        .setPlaceholderImageScaleType(scaleType)
        .setPressedStateOverlay(overlay)
        .setProgressBarImage(progressBarImage)
        .setProgressBarImageScaleType(scaleType)
        .setRetryImage(retryImage)
        .setRetryImageScaleType(scaleType)
        .setRoundingParams(roundingParams)
        .build();
mSimpleDraweeView.setHierarchy(hierarchy);
mSimpleDraweeView.setImageURI(uri);
```
有一些选项可以直接在一个现有的Hierarchy上被设置，而不需要我们重新创建一个Hierarchy。我们可以直接从SimpleDraweeView的`getHierarchy()`方法来得到这个Hierarchy，然后对它进行设置：
``` java
mSimpleDraweeView.getHierarchy().setPlaceHolderImage(placeholderImage);
```

## 2. ImageRequest
如果你需要的`ImageRequest`只有一个URI，只需要用它的`formUri`方法就可以载入图片了。

### 2.1 使用`ImageRequestBuilder`构建`ImageRequest`
但是如果你要的ImageRequest不止是一个URI，而是有别的配置（如后面写的Postprocessor），这时候你就需要一个ImageRequest了：
``` java
Uri uri;

ImageDecodeOptions decodeOptions = ImageDecodeOptions.newBuilder()
        .setBackgroundColor(Color.GREEN)
        .build();

ImageRequest request = ImageRequestBuilder
        .newBuilderWithSource(uri)
        .setImageDecodeOptions(decodeOptions)
        .setAutoRotateEnabled(true)
        .setLocalThumbnailPreviewsEnabled(true)
        .setLowestPermittedRequestLevel(RequestLevel.FULL_FETCH)
        .setProgressiveRenderingEnabled(false)
        .setResizeOptions(new ResizeOptions(width, height))
        .build();
```
在上面的这些参数中，只有Uri是必要的，其他都是可选的。

### 2.2 通过ImageRequest和`DraweeController`加载多个图像
当我们想向用户展示一张高分辨率而且下载很慢的图像，我们可能希望先快速地将它的缩略图下载下来。我们可以使用两个URI，一个是高分辨率的，一个是低分辨率的：
``` java
Uri lowResUri, highResUri;
DraweeController controller = Fresco.newDraweeControllerBuilder()
    .setLowResImageRequest(ImageRequest.fromUri(lowResUri))
    .setImageRequest(ImageRequest.fromUri(highResUri))
    .setOldController(mSimpleDraweeView.getController())
    .build();
mSimpleDraweeView.setController(controller);
```
在高分图像加载出来之前，Fresco就会先把低分辨率的图像展示出来。

除此之外，不仅可以往DraweeController中加载低分辨率图像，我们还可以只加载第一张能够拿到的图像。这个方法的逻辑就是在内存中一张一张地检查所有的图像。
创建ImageRequest的数组，把它们传递给builder：
``` java
Uri uri1, uri2;
ImageRequest request = ImageRequest.fromUri(uri1);
ImageRequest request2 = ImageRequest.fromUri(uri2);
ImageRequest[] requests = { request1, request2 };

DraweeController controller = Fresco.newDraweeControllerBuilder()
    .setFirstAvailableImageRequests(requests)
    .setOldController(mSimpleDraweeView.getController())
    .build();
mSimpleDraweeView.setController(controller);
```
只有第一个ImageRequest请求会被展示。第一个被找到的，不管是在内存、cache或是网络层面上，就是最后被返回的那个。

## 3. 用Postprocessor类来做图像的后处理
想要对图像进行后处理，我们最好写一个`Postprocessor`类来继承`BasePostprocessor`类，然后重写其中的`process`方法，在里面写自己的图像处理的逻辑。下面的例子是一个将图像进行黑白处理的Postprocessor：
``` java
public class FastGreyScalePostprocessor extends BasePostprocessor {

    @Override
    public void process(Bitmap bitmap) {
        final int w = bitmap.getWidth();
        final int h = bitmap.getHeight();
        final int[] pixels = new int[w * h];

        bitmap.getPixels(pixels, 0, w, 0, 0, w, h);

        for (int x = 0; x < w; x++) {
            for (int y = 0; y < h; y++) {
                final int offset = y * w + x;
                pixels[offset] = getGreyColor(pixels[offset]);
            }
        }

        // this is much faster then calling #getPixel and #setPixel as it crosses
        // the JNI barrier only once
        bitmap.setPixels(pixels, 0, w, 0, 0, w, h);
    }

    static int getGreyColor(int color) {
        final int alpha = color & 0xFF000000;
        final int r = (color >> 16) & 0xFF;
        final int g = (color >> 8) & 0xFF;
        final int b = color & 0xFF;

        // see: https://en.wikipedia.org/wiki/Relative_luminance
        final int luminance = (int) (0.2126 * r + 0.7152 * g + 0.0722 * b);

        return alpha | luminance << 16 | luminance << 8 | luminance;
    }
}
```
然后我们根据这个Postprocessor创建一个新的ImageRequest就行了：
``` java
        ImageRequest imageRequest = ImageRequestBuilder
                .newBuilderWithSource(highResUri)
                .setPostprocessor(new FastGreyScalePostprocessor())
                .build();

        DraweeController controller = Fresco.newDraweeControllerBuilder()
                .setImageRequest(imageRequest)
                .setOldController(mSimpleDraweeView.getController())
                .build();

        mSimpleDraweeView.setController(controller);
```
注意要使用ImageRequestBuilder的set方法才能将Postprocessor和ImageRequest绑定（ImageRequest的Postprossor成员变量是`final`，无法修改）。

## Reference
1. https://www.fresco-cn.org/
2. https://frescolib.org/docs/
3. https://www.jianshu.com/p/a44ee3dd5ac4
