---
layout: post
title: "解决AppCompat_v7:21.0.0兼容行问题"
date: 2014-11-14 16:26:49 +0800
comments: true
categories: 技术
---

1.实现新的ActionBarDrawerToggle动画
ActionBarDrawerToggle使用最新的AppCompat_v7 21会出现一个很帅的动画，使用方式在Androidstudio下面先添加compile
[html] view plaincopyprint?在CODE上查看代码片派生到我的代码片
dependencies {  
    compile fileTree(dir: 'libs', include: ['*.jar'])  
    compile 'com.android.support:appcompat-v7:21.0.0'  
}  

然后直接将ActionBarDrawerToggle的impot使用import android.support.v7.app.ActionBarDrawerToggle;
之后的构造方法就可以使用
 mDrawerToggle = new ActionBarDrawerToggle(this, mDrawerLayout, toolbar, R.string.app_name, R.string.app_name);
当然需要一个ToolBar。

2.使用ToolBar代替ActionBar
Android L添加了一个新控件来一步步的替换ActionBar，ToolBar有着更灵活的扩展，完全能够代替ActionBar，并且有着自己作为一个View的灵活性。只是有点不方便的是，ToolBar需要在每个Activity中声明，不管是在XML中或者代码
修改主题
[html] view plaincopyprint?在CODE上查看代码片派生到我的代码片
<resources>  
    <style name="AppTheme" parent="Theme.AppCompat">  
        <!-- Customize your theme here. -->  
        <item name="windowActionBar">false</item>  
        <item name="android:windowNoTitle">true</item>  
  
        <!-- Actionbar color -->  
        <item name="colorPrimary">@color/accent_material_dark</item>  
        <!--Status bar color-->  
        <item name="colorPrimaryDark">@color/accent_material_light</item>  
        <!--Window color-->  
        <item name="android:windowBackground">@color/dim_foreground_material_dark</item>  
    </style>  
</resources>  
如果你原来使用ActionBarPullToRefresh控件这个时候会发现，进度条和底边有俩个dp的间隔，如果使用了ToolBar，那么你就可以控制ActionBar的高度，当然你可以修改ActionBarPullToRefresh源码来解决这个问题
布局中添加ToolBar
[html] view plaincopyprint?在CODE上查看代码片派生到我的代码片
<android.support.v7.widget.Toolbar  
    android:id="@+id/toolbar"  
    android:layout_width="match_parent"  
    android:layout_height="wrap_content"  
    android:minHeight="?attr/actionBarSize"  
    android:background="?attr/colorPrimary"  
    ></android.support.v7.widget.Toolbar>  

[java] view plaincopyprint?在CODE上查看代码片派生到我的代码片
Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);  
setSupportActionBar(toolbar);  

这样就得到了一个代替ActionBar的Toolbar，测试发现内部的Fragment getActionBar还是可以直接使用，因为在Activiyty中将ToolBar设置为了ActionBar，所以内部的Fragment 对ActionBar的操作完全可以和以前一样使用。当然也可以在fragment中设置自己的Toolbar。

3.Actionbar的ListModel错位问题
ActionBar.NAVIGATION_MODE_LIST在替换了AppCompat 之后在4.x上显示会出现错位的情况，并且显示这个方法已经被废弃，如下图


解决方案是使用TintSpinner代替Spinner，奇怪的是TintSpinner在官网竟然查不到相关的信息，当然它也在android.support.v7.internal.widget包中，打开看源码发现
[java] view plaincopyprint?在CODE上查看代码片派生到我的代码片
public class TintSpinner extends android.widget.Spinner {  
    private static final int[] TINT_ATTRS;  
  
    public TintSpinner(android.content.Context context) { /* compiled code */ }  
  
    public TintSpinner(android.content.Context context, android.util.AttributeSet attrs) { /* compiled code */ }  
  
    public TintSpinner(android.content.Context context, android.util.AttributeSet attrs, int defStyleAttr) { /* compiled code */ }  
}  

他继承Spinner，但是没有做任何的改动，好奇观，替换了之后确实解决了问题。



4.待续。。。