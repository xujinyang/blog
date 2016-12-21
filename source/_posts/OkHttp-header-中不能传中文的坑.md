title: OkHttp header 中不能传中文的坑
date: 2016-02-24 22:52:05
tags: Android
---


运营同事反馈说，有不止一个魅族用户说，登录不上我们的应用；我听到这个问题，很自信的回复他，肯定是用户的网络有问题，最多是后端的接口挂了，和app 没有关系。但是bugly上看了一下，突然发现有个bug

		IllegalArgumentException: Unexpected char ... in header value: ... at com.squareup.okhttp.Headers$Builder.checkNameAndValue (Headers.java:295)


我赶紧到魅族官方网站上下载了新春特别版的Rom，安装，打开，果然网络请求失败，崩溃了.
	
    问题是这样的，我们的UA中存在上传手机系统版本号的字段：android.os.Build.DISPLAY，这个本无问题，但是魅族春节的时候发了一个新春特别版本，他的系统版本号为：Flyme OS 5.1.3.0A 新春特别版.
	
这个锅我不能接，赶紧查了okhttp的[issues](https://github.com/square/okhttp/issues/2016)
有个印度哥哥说：

	Non-ASCII characters are allowed according to both the original and updated HTTP 1.1 specs.
	
	The updated spec recommends against using non-ASCII characters for newly-defined headers, but clearly they're allowed:
	
	Historically, HTTP has allowed field content with text in the ISO-8859-1 charset [ISO-8859-1], supporting other charsets only through use of [RFC2047] encoding. In practice, most HTTP header field values use only a subset of the US-ASCII charset [USASCII]. Newly defined header fields SHOULD limit their field values to US-ASCII octets. A recipient SHOULD treat other octets in field content (obs-text) as opaque data.
	Because of the way UTF-8 is encoded, all codepoints above 127 are composed of bytes whose values are 128 or greater, which means they should pass through untouched. The only characters you have to be careful with are ASCII control characters and, in certain situations, spaces, quotes, and backslashes.

不翻译了，简单来说就是不支持中文		

		Hi, this restriction about just ASCII chars is very strange, we support chinese users etc... So almost all of our request fail when we try to send some chinese value using the header. This works fine in the HttpRequest apache. Why this restriction?

用了那么久的OkHttp，第一次知道这个坑，望各位绕过，但是能看到这篇bolg的应该已经中招了，下面怎么解决尼？

 **避免出现中文**

我的解决方式:

```
 private String getOsDisplay() {
        String osStr = StringUtil.removeSpace(android.os.Build.DISPLAY);
        if (osStr.length() < osStr.getBytes().length) {
            try {
                return URLEncoder.encode(StringUtil.removeSpace(android.os.Build.DISPLAY), "UTF-8");
            } catch (Exception e) {
                e.printStackTrace();
                return "";
            }
        } else {
            return osStr;
        }
    }
```