---
layout:     post
title:      Android视频播放
subtitle:   视频播放之ExoPlayer库
date:       2020-07-29
author:     HYC
header-img: img/android-large.png
catalog: true
tags:
    - Android
    - ExoPlayer
---

## 1. HelloWorld
### 1.1 添加依赖
在builde.gradle中加入如下代码，确保Google和JCenter的仓库被包括进去：
``` groovy
repositories {
    google()
    jcenter()
}
```
然后添加依赖：
``` groovy
implementation 'com.google.android.exoplayer:exoplayer:2.11.7'
```

### 1.2 创建播放器
你可以使用`SimpleExoPlayer.Builder`或是`ExoPlayer.Builder`来创建一个`ExoPlayer`的实例。这些建造者模式提供了一系列的定制方法来创建一个实例。
``` java
SimpleExoPlayer player = new SimpleExoPlayer.Builder(context).build();
```

### 1.3 将播放器附在视图上
ExoPlayer库提供了`PlayerView`，它封装了一个`PlayerControlView`，一个`SubtitleView`和一个渲染视频的`Surface`。我们在layout文件中添加这个视图：
<com.google.android.exoplayer2.ui.PlayerView
    android:id="@+id/player_view"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" />

我们将播放器绑定到视图上，很简单，只有一行代码：
``` java
playerView.setPlayer(player);
```

### 1.4 准备进行播放
在ExoPlayer中，每一个多媒体都是被`MediaSource`被代表。如果想要播放一个多媒体你必须先创建一个对应的`MediaSource`，然后把这个对象传递给`ExoPlayer.prepare`。

> The ExoPlayer library provides MediaSource implementations for DASH (DashMediaSource), SmoothStreaming (SsMediaSource), HLS (HlsMediaSource) and regular media files (ProgressiveMediaSource). 
``` java
// Produces DataSource instances through which media data is loaded
val dataSourceFactory = DefaultDataSourceFactory(this, Util.getUserAgent(this, "My App"))
// This is the MediaSource representing the media to be played
val videoSource = ProgressiveMediaSource.Factory(dataSourceFactory).createMediaSource(mp4VideoUri)
// Prepare the player with the source
player.prepare(videoSource)
```

## 2. 监听播放器事件
将播放器注册在事件监听的接口上：
``` java
player.addListener(eventListener)
```

播放器的状态主要有4个：
- `Player.STATE_IDLE`：初始状态，当播放器停止时或回放失败时进入这个状态
- `Player.STATE_BUFFERING`：这时播放器不能立即从现在的位置播放，因为需要载入更多的视频数据
- `Player.STATE_READY`：播放器能立即从当前位置开始播放
- `Player.STATE_ENDED`：播放器完成播放所有的媒体

通过实现接口我们可以感知播放器是否正在播放：
``` java
@Override
public void onIsPlayingChanged(boolean isPlaying) {
  if (isPlaying) {
    // Active playback.
  } else {
    // Not playing because playback is paused, ended, suppressed, or the player
    // is buffering, stopped or failed. Check player.getPlaybackState,
    // player.getPlayWhenReady, player.getPlaybackError and
    // player.getPlaybackSuppressionReason for details.
  }
}
```

以及获取到使回放失败的异常：
``` java
@Override
public void onPlayerError(ExoPlaybackException error) {
  if (error.type == ExoPlaybackException.TYPE_SOURCE) {
    IOException cause = error.getSourceException();
    if (cause instanceof HttpDataSourceException) {
      // An HTTP error occurred.
      HttpDataSourceException httpError = (HttpDataSourceException) cause;
      // This is the request for which the error occurred.
      DataSpec requestDataSpec = httpError.dataSpec;
      // It's possible to find out more about the error both by casting and by
      // querying the cause.
      if (httpError instanceof HttpDataSource.InvalidResponseCodeException) {
        // Cast to InvalidResponseCodeException and retrieve the response code,
        // message and headers.
      } else {
        // Try calling httpError.getCause() to retrieve the underlying cause,
        // although note that it may be null.
      }
    }
  }
}
```

## Reference
1. https://exoplayer.dev/hello-world.html
