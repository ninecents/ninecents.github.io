#

## 浏览器网页刷新

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