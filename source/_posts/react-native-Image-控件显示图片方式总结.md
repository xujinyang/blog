title: react native Image 控件显示图片方式总结
date: 2016-02-24 22:13:40
tags: React-Native
---
> React Native 每一个控件都能让人把玩半天，每一个都能让你玩的泪流满面，这里总结一下Image控件的几种使用方式，不用想也是踩着坑过来的。

### 1. 打开Android 项目中的图片
		
		Android 项目中的图片房子draw文件夹中，比如ico.png
	
	

```
<Image
	source={{uri: "ico", isStatic: true}}
 />
```
**isStatic 代表是是否是静态文件，默认false**
### 2. 打开js文件夹下面的图片
	比如图片放在react-core/images/ico.png
```
<Image
    source={require('../images/ico.png')}
 />
```
**相对路径下面的，一定要写对路径地址，还有就是要写上.png图片类型**

### 3. 打开网络图片-url
	比如图片地址为：http://xx.xx.xx/ico.png
    
```
<Image
	source={{uri: 'http://xx.xx.xx/ico.png'}}
 />
```
 
### 4. 打开js对象中的本地文件
	这个有点蛋疼，很多时候我们要对象造数据，数据里面如果有本地的图片地址，比如
```
	var data={
			src:'../images/ico.png',
			type: "1"
			};
```
这个时候要显示使用Image显示这个图片

```
<Image
    source={require(data.src)}
 />
```
就会报错，提示图片找不到，这个时候黑科技就来了，需要这样写：

```
var data={
			src:require('../images/ico.png'),
			type: "1"
			};
<Image
    source={data.src}
 />			
```
不要问我为什么，对于一个Android开发人员来说，JS真是有容乃大，各种感觉不可思议的用法已经让我刮目相看。

***总结完毕，如果一个功能你使用了React Native，那么恭喜你选择了困难模式，反正我已经在这条踩坑的路上，越走越远，越陷越深。***
