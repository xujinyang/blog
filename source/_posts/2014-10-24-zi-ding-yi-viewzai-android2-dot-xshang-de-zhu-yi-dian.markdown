---
layout: post
title: "自定义View在Android2.x上的坑"
date: 2014-10-24 18:41:24 +0800
comments: true
categories: Android
---

自定义LinearLayout的时候有三个构造方法

``` 

	public OrderDetailView(Context context) {
        super(context);
        init(context);
    }

    public OrderDetailView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init(context);
    }

    public OrderDetailView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        init(context);
    }

```
很常识的事情，但是如果你写成下面这样

``` 

	public OrderDetailView(Context context) {
        this(context,null);
    }

    public OrderDetailView(Context context, AttributeSet attrs) {
        this(context, attrs,0);
    }

    public OrderDetailView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        init(context);
    }

```

在Android3.0以上没有任何问题，但是3.0以下奇怪的事情发生了，应用直接crash，查了发现,android2.x的sdk中没有三个参数的构造方法，如果用代码生成控件会调用一个参数的构造方法，用布局生成的控件会调用俩个参数的构造方法，三个参数的构造方法是3.0以上的系统在两个构造方法中主动调用才能执行的，这个坑踩过的人才会知道，没踩过的看到就不要踩了。
