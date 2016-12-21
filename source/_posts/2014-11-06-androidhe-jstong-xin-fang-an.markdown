---
layout: post
title: "Android和JS通信方案"
date: 2014-11-06 00:04:05 +0800
comments: true
categories: Android
---

在Android的webview中如果要想调用Javascript的接可以用			

	WebView.loadUrl("javascript:onJsAndroid()");  

在Javascript中调用Android的接口可以

	mWebView.addJavascriptInterface(new function(),"androd");  
    
	public class function(){
    		@JavascriptInterface
    		public void methodOne(){
            
            }
    } 
    
但是有点美中不足的是，Android调用js的方法获取不到返回值，网上关于这个问题的解决方案也有，但是要全平台的实现还是有风险的，所以需要一种解决方案来实现获取返回值。

#### 方案：

Android调用完js接口后，js再调用默认约定好的java接口，将值返回，这样有点hardcode的感觉，但是能解决问题，如果想更灵活，可以约定握手code，比如java调用接口的时候，传入一个时间戳，并hashmap保存到本地，当js调用java方法返回内容的时候将这个时间戳传回来，hashmap中查找到对应的时间戳，就可以知道是那个请求，并找到对应的处理。
