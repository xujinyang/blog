---
layout: post
title: "为什么要使用ToolBar代替ActionBar"
date: 2014-11-14 11:42:46 +0800
comments: true
categories: 技术
---
- *ActionBar也是一种Toolbar，从API 21开始 Toolbar可以来替代ActionBar*
- 

		A primary toolbar within the activity that may display the activity title, application-level navigation affordances, and other interactive items.
        Beginning with Android L (API level 21), the action bar may be represented by any Toolbar widget within the application layout. The application may signal to the Activity which Toolbar should be treated as the Activity's action bar. Activities that use this feature should use one of the supplied .NoActionBar themes, set the windowActionBar attribute to false or otherwise not request the window feature.
        
        
- *Toolbar可以使用setSupportActonBar()来作为actionbar使用，也可以使用在页面的任何地方作为一个View*


        A Toolbar is a generalization of action bars for use within application layouts. While an action bar is traditionally part of an Activity's opaque window decor controlled by the framework, a Toolbar may be placed at any arbitrary level of nesting within a view hierarchy. An application may choose to designate a Toolbar as the action bar for an Activity using the setSupportActionBar() method.
   
   
### Toolbar 的特点


-  作为导航（使用appCompat v7 21.0.0 的ActionBarDrawerToggle 可以使用google的一个很有趣的动画） 

        A navigation button. This may be an Up arrow, navigation menu toggle, close, collapse, done or another glyph of the app's choosing. This button should always be used to access other navigational destinations within the container of the Toolbar and its signified content or otherwise leave the current context signified by the Toolbar.
        
        
-使用夸张的设计来彰显品牌，因为是一个customView，所有高度可以任意的设置

![](http://attach.bbs.miui.com/forum/201407/04/193328g5bqgbalqbq9aj9b.jpg)


	A branded logo image. This may extend to the height of the bar and can be arbitrarily wide.
    
    
- 显示标题和副标题


        A title and subtitle. The title should be a signpost for the Toolbar's current position in the navigation hierarchy and the content contained there. The subtitle, if present should indicate any extended information about the current content. If an app uses a logo image it should strongly consider omitting a title and subtitle.
    
    
- 任意的在Toolbar中添加子view，这也是他能代替actionBar的关键点
    
    
    
        One or more custom views. The application may add arbitrary child views to the Toolbar. They will appear at this position within the layout. If a child view's Toolbar.LayoutParams indicates a Gravity value of CENTER_HORIZONTAL the view will attempt to center within the available space remaining in the Toolbar after all other elements have been measured.
        
        
- 类似ActionBar menu的机制，多的话会几个小点


        An action menu. The menu of actions will pin to the end of the Toolbar offering a few frequent, important or typical actions along with an optional overflow menu for additional actions.
  
  
在布局中可以实现层叠的效果，很美妙哦，其实没有Toolbar之前这样的效果也很容易实现，但是Google的material design的推行，将这样的设计提升到了一个高度，形成了一种独特的风格，有种难言的简洁和抽象美


      <android.support.v7.widget.Toolbar
        android:id=”@+id/my_awesome_toolbar”
        android:layout_height=”wrap_content”
        android:layout_width=”match_parent”
        android:minHeight=”?attr/actionBarSize”
        android:background=”?attr/colorPrimary” />


![](http://2.bp.blogspot.com/-opO8On-LSWU/VEgoMYJiAOI/AAAAAAAAA3I/i0eM6kUktkU/s1600/md.png)     