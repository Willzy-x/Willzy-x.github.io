---
layout:     post
title:      Android 基础知识
subtitle:   在制作仿App Store碰到的问题
date:       2020-07-17
author:     HYC
header-img: img/android-large.png
catalog: true
tags:
    - Android
    - Java
---

## 1. 用ToolBar代替ActionBar

在布局中加入：
``` xml
<android.support.v7.widget.Toolbar
      android:id="@+id/toolbar"
      android:layout_width="match_parent"
      android:layout_height="?attr/actionBarSize"
      android:background="?attr/colorPrimary"
      android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
      app:popupTheme="@style/ThemeOverlay.AppCompat.Light" />
```

- `app:popupTheme` 属性单独将弹出的菜单项指定成了淡色主题。

因为Toolbar是在android 5.0之后加入的，所以我们必须在父父布局中声明app这个命名空间，从而达到使用`app:attribute`类似的属性。

然后在主活动中添加如下的代码：
``` java
Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
setSupportActionBar(toolbar);
```

## 2. AppBarLayout
它其实是一个垂直方向的`LinearLayout`，内部做了很多的滚动事件的封装。并应用了一些Material Design的概念。

### 2.1 防止AppBarLayout遮挡
然后在下面的LinearLayout里面加入`app:layout_behavior` 属性，指定了一个布局行为。其中`appbar_scrolling_view_behavior` 这个字符串也是由Design Support库提供的。这样就可以防止这个Layout被`AppBarLayout`遮挡了。
``` xml
<LinearLayout
    android:id="@+id/overall_layout"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    app:layout_behavior="@string/appbar_scrolling_view_behavior">

</LinearLayout>
```

### 2.2 实现滚动隐藏AppBarLayout效果
向上滚动AppBar自动隐藏：在Toolbar中添加了一个`app:layout_scrollFlags` 属性，并将这个属性的值指定成了`scroll|enterAlways|snap` 。其中，`scroll` 表示当`RecyclerView`向上滚动的时候，`Toolbar`会跟着一起向上滚动并实现隐藏；`enterAlways` 表示当`RecyclerView`向下滚动的时候，`Toolbar`会跟着一起向下滚动并重新显示。`snap` 表示当`Toolbar`还没有完全隐藏或显示的时候，会根据当前滚动的距离，自动选择是隐藏还是显示。
``` xml
<android.support.design.widget.AppBarLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content" >

    <android.support.v7.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:background="?attr/colorPrimary"
        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
        app:popupTheme="@style/ThemeOverlay.AppCompat.Light"
        app:layout_scrollFlags="scroll|enterAlways|snap"/>

</android.support.design.widget.AppBarLayout>
```

## 3. Parcelable接口
`Parcalable`实现原理是将一个完整的对象进行分解,而分解后的每一部分对象都支持intent所支持的数据类型，这样也实现了传递对象的功能。

> Interface for classes whose instances can be written to and restored from a Parcel. Classes implementing the Parcelable interface must also have a non-null static field called CREATOR of a type that implements the Parcelable.Creator interface.

先实现`Parcelable`接口,还有必须要重写的两个方法：
- `describeContents()`，一般返回0。
- `writeToParcel()`，我们需要在这个方法中将App类中的字段一一写出。
``` java
@Override
public int describeContents() {
    return 0;
}

@Override
public void writeToParcel(Parcel parcel, int i) {
    parcel.writeString(this.appTitle);
    parcel.writeInt(this.imageId);
    parcel.writeInt(this.appSize);
}

同时需要创建非空的静态域叫CREATOR的一个读Parcel 对象的Creator。
public App(Parcel in) {
    this.appTitle = in.readString();
    this.imageId = in.readInt();
    this.appSize = in.readInt();
}

public static final Creator<App> CREATOR = new Creator<App>() {
    @Override
    public App createFromParcel(Parcel in) {
        return new App(in);
    }

    @Override
    public App[] newArray(int size) {
        return new App[size];
    }
};
```

## 4. Fragment的show和hide实现切换

调用`FragmentTransaction`进行操作：
``` java
FragmentTransaction transaction = MainActivity.this.mFragmentManager.beginTransaction();
...
transaction.hide(MainActivity.this.anotherFragment);
transaction.show(MainActivity.this.gameFragment);
```

最后别忘了提交事务：
``` java
transaction.commit();
```

如果没有加载fragment，别忘了`add`：
``` java
if (!MainActivity.this.mFragmentManager.getFragments().contains(MainActivity.this.anotherFragment)) {
    transaction.add(R.id.overall_layout, MainActivity.this.anotherFragment);
}
```

### 附上Github仓库地址：
https://github.com/Willzy-x/app-store-demo