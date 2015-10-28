title: Android View变换
date: 2015-03-27 15:43:37
categories: android
tags: [android, view]
---

View的平移、缩放、旋转等变换。

<!-- more -->

我想通过一个实例来掩饰view的旋转、缩放等变换。
需求是这样的：

用TextureView实现一个简单的视频播放器，要求播放时最大程度的利用屏幕，如竖屏时
播放一个宽屏的视频时，比如一个1080x1920的屏幕(竖屏)播放一个1920x1080的视频，
此时要将视频旋转90度，这样就可以实现全屏播放，从而最大程度的利用屏幕。

我的做法就是对TextureView先旋转再缩放，

```java
    private void changeLayoutParams(TextureView view, int videoWidth, int videoHeight) {
        if (videoWidth == 0 || videoHeight == 0) return;

        int targetWidth = mScreenWidth;
        int targetHeight = mScreenHeight;
        Log.d(TAG, "target width & height: " + targetWidth + ", " + targetHeight);
        Log.d(TAG, "video width & height: " + videoWidth + ", " + videoHeight);

        boolean needRotate = true;
        float ratio;
        float scaleX;
        float scaleY;
        float rotation = 0.0f;

        int newVideoWidth = videoWidth;
        int newVideoHeight = videoHeight;
        if ((targetWidth - targetHeight) * (videoWidth - videoHeight) < 0 && needRotate) {
            newVideoWidth = videoHeight;
            newVideoHeight = videoWidth;
            rotation = 90.0f;
        }

        // rotate
        view.setRotation(rotation);

        // scale
        // 1. scale to video size
        scaleX = (float)videoWidth/targetWidth;
        scaleY = (float)videoHeight/targetHeight;
        // 2. scale to fix target size
        ratio = Math.min((float)targetWidth/newVideoWidth, (float)targetHeight/newVideoHeight);
        Log.d(TAG, "ratio: " + ratio + " ,scale x: " + scaleX
                + " ,scale y: " + scaleY + ", rotation: " + rotation);
        view.setScaleX(scaleX * ratio);
        view.setScaleY(scaleY * ratio);
    }
```

变换的过程就是：

1. 旋转view，使view和视频的长边对长边、短边对短边
2. 缩放view，使view的长边等于视频的长边、短边等于视频的短边
3. 缩放view，使view保持画面比例的情况下，最大程度的适配屏幕尺寸

这里需要注意的是：view的旋转、缩放、平移都是在draw阶段发生的，因而这些操作不会影响view的
width, height，和位置等信息，事实上这些操作跟onDraw()时对Canvas的操作是一样的。

无论怎么操作，view的Surface(盛放view内容的内存区域)都是不变的，它在measure, layout之后就固定了，
draw的任何操作只是去使用这块内存区域，至于怎么往上画东西这是view定义者的自由，不过超出该区域的
内容将被无情的忽略掉！

我们这里的情况就是这样的，TextureView是match parent的，即尺寸等于屏幕尺寸(1080x1920)，所以为其
分配的内存区域就是1080x1920，但是视频确实1920x1080的，默认情况下media player只是简单的将视频内容
填充到这块内存中，从而使得画面显示不全或者拉伸等情况。

### 代码讲解

首先，选择90度，

```
        origin
          +----+  --> X(width)
          |    |          
          |    |          
          |    |  Y(height) <--    
+---------+----+---------+ new origin
|         |    |         |
+---------+----+---------+
          |    |         v 
          |    |         X(width) 
          |    |          
          |    |          
          +----+          
          v
          Y(height)
```

开始时，Canvas坐标原点跟Surface左上角重合，旋转90度后原点变为new origin，但是Surface仍然是竖直的
那个区域，此时视频画面从new origin开始画，结果就是面目全非，所以还需要缩放操作。

其次，缩放view适配视频大小。view的X轴原本是屏幕宽度，现在要scale到视频的宽度，Y轴原本是屏幕高度，现在
要scale到视频的高度，所以缩放因子是

```java
        scaleX = (float)videoWidth/targetWidth;
        scaleY = (float)videoHeight/targetHeight;
```

然后还要考虑屏幕比例跟视频比例不一致的情况，此时要将view等比例缩放，是按照X轴缩放还是按照Y轴缩放？
答案是哪个缩放比例小就按照那个来，

```java
ratio = Math.min((float)targetWidth/newVideoWidth, (float)targetHeight/newVideoHeight);
```

自始至终，Surface都是不变的，切记！！！

### 另一种解法？！

还有一种想法，就是先旋转，然后通过LayoutParams将view的宽高互换，这样就不用中间的那步缩放了，
但是这样是不行的，为啥？因为view的parent就是1080x1920，view怎么能申请到1920x1080的尺寸呢？
measure过程就被改为1080x1080了！

(over)