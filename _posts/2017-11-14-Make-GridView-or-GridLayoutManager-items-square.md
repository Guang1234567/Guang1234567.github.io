---
title: Make GridView or GridLayoutManager items square.
layout: post
categories: Android
tags: Android_UI
---

> 转载至 <https://stackoverflow.com/questions/26566954/square-layout-on-gridlayoutmanager-for-recyclerview>

> 目的: 使 `GridView` or `GridLayoutManager` 的每个 `item` 变成正方形格子

* content
{:toc}




To have the square elements in my RecyclerView, I provide a simple wrapper for my root View element; I use the following SquareRelativeLayout in place of RelativeLayout.

```java
    package net.simplyadvanced.widget;

    import android.content.Context;
    import android.util.AttributeSet;
    import android.widget.RelativeLayout;

    /** A RelativeLayout that will always be square -- same width and height,
     * where the height is based off the width. */
    public class SquareRelativeLayout extends RelativeLayout {

        public SquareRelativeLayout(Context context) {
            super(context);
        }

        public SquareRelativeLayout(Context context, AttributeSet attrs) {
            super(context, attrs);
        }

        public SquareRelativeLayout(Context context, AttributeSet attrs, int defStyleAttr) {
            super(context, attrs, defStyleAttr);
        }

        @TargetApi(VERSION_CODES.LOLLIPOP)
        public SquareRelativeLayout(Context context, AttributeSet attrs,         int defStyleAttr, int defStyleRes) {
            super(context, attrs, defStyleAttr, defStyleRes);
        }

        @Override
        protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
            // Set a square layout.
            super.onMeasure(widthMeasureSpec, widthMeasureSpec);
        }

    }
```

Then, in my XML layout for the adapter, I've just referenced the custom view as shown in the following. Though, you can do this programmatically also.

```xml
    <?xml version="1.0" encoding="utf-8"?>
    <net.simplyadvanced.widget.SquareRelativeLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/elementRootView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content">

        <!-- More widgets here. -->

    </net.simplyadvanced.widget.SquareRelativeLayout>
```


> Note: Depending on which orientation your grid is, then you may want to have the width based off of height (GridLayoutManager.HORIZONTAL) instead of the height being based off the width (GridLayoutManager.VERTICAL).