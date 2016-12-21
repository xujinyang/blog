title: 如何用React-native Post Form表单
date: 2016-01-27 23:00:18
tags: React Native
---
今天使用react native 发送请求的时候，发现使用

```
fetch('https://mywebsite.com/endpoint/', {%
  method: 'POST',
  headers: {
  },
  body: JSON.stringify({
    firstParam: 'yourValue',
    secondParam: 'yourOtherValue',
  })
})
```
默认发送'Content-Type': 'application/json' 的请求，但是现在想发送Form表单,该怎么写尼？查阅[官方文档](http://facebook.github.io/react-native/docs/network.html#content)
无解，各种stackoverflow还是没有类似的答案，每当stackoverflow上都没有答案的时候，我就准备看源码了，这时候，突然想到立成同学的一句话，github的issues上面比文档还可靠，抱着试试看的态度，翻了一下issue，原来github的issue还有内部搜索功能，真方便。功夫不负有心人，在[#5308](https://github.com/facebook/react-native/issues/5308)
这里找到了答案：

```
let formData = new FormData();
formData.append('image', {uri: image.uri, type: 'image/jpg', name: 'image.jpg'});
formData.append('description', String(data.description));

let options = {};
options.headers['Content-Type'] = 'multipart/form-data; boundary=6ff46e0b6b5148d984f148b6542e5a5d';
options.body = formData;
return fetch(uri, options).then((response) => {
   .... 
});
```

只要把body从Json换成FromData就可以了，解决完问题然后下班的感觉真好。