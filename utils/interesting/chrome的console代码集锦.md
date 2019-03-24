# NoTitle
- [JavaScript 教程](http://www.runoob.com/js/js-tutorial.html)

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
var test=document.getElementsByTagName('html')[0].innerHTML;
for (var i=0;i<test.length;i++)
{ 
    console.info(test.substr(4000,1000));
}
```
