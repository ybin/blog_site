title: android selector layer-list shape
date: 2015-08-07 16:43:29
categories: [android]
tags: [android, shape, layer-list, level-list, inset]
---

Android中的简单几何矢量图示例。

<!-- more -->

# layer-list，将多个图片合成为一张

```xml
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item>
        <shape
            android:shape="oval">
            <solid android:color="#f00"/>
            <stroke
                android:color="#000"
                android:dashGap="3dp"
                android:dashWidth="6dp"
                android:width="2dp"/>
        </shape>
    </item>

    <item>
        <!-- 可以设置左上右下的inset值，从而达到将图片放到任意位置的意图 -->
        <inset android:inset="20dp">
            <shape
                android:shape="rectangle">
                <solid android:color="#000"/>
            </shape>
        </inset>
    </item>
</layer-list>
```

# level-list，通过level值切换图片

```xml
<?xml version="1.0" encoding="utf-8"?>
<level-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:maxLevel="0"
        android:minLevel="0">
        <shape
            android:shape="oval">
            <solid android:color="#f00"/>
        </shape>
    </item>

    <item
        android:maxLevel="1"
        android:minLevel="1">
        <shape
            android:shape="rectangle">
            <solid android:color="#f00"/>
        </shape>
    </item>

    <item
        android:drawable="@drawable/ic_launcher"
        android:maxLevel="2"
        android:minLevel="2"/>
</level-list>
```

这个需要Java代码设置`setLevel(int)`，设置哪个level，那个图片就显示出来。

# selector，根据状态自动切换图片

```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_pressed="true">
        <shape
            android:shape="rectangle">
            <solid android:color="#0f0"/>
        </shape>
    </item>

    <item>
        <shape
            android:shape="rectangle">
            <solid android:color="#000"/>
        </shape>
    </item>
</selector>
```

# transition，用动画形式切换图片

```xml
<?xml version="1.0" encoding="utf-8"?>
<transition xmlns:android="http://schemas.android.com/apk/res/android">
    <item>
        <shape
            android:shape="rectangle">
            <solid android:color="#ff0"/>
        </shape>
    </item>

    <item>
        <shape
            android:shape="rectangle">
            <solid android:color="#00f"/>
        </shape>
    </item>
</transition>
```

需要Java代码配合调用TransationDrawable的startTransition(int)来设置动画时常。
注意，这玩意儿只是一个动画，并没有实际的切换到另一个图片。

(over)