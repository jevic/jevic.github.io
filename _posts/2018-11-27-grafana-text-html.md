---
layout: post
categories: OPS
title: grafana html 请求显示docker镜像信息
date: 2018-11-27 19:56:06 +0800
description: grafana
keywords: grafana
---

>通过grafana 请求应用接口显示Docker镜像的构建版本信息


### 图例
![](https://raw.githubusercontent.com/jevic/images/master/other/grafana-text-version01.png)
![](https://raw.githubusercontent.com/jevic/images/master/other/grafana-text-version02.png)

### api
```
curl -sL http://192.168.2.146:9010/version |python -mjson.tool
{
    "data": {
        "buildtime": "2018-11-27_17:25:43",
        "hash": "7cd3a541053a9d67eee1706d380af3fe",
        "version": "fa92b4d"
    },
    "ok": true
}
```

### html 代码

```
<div class="version">

</div>

<style>
    .version ul{
        list-style:none;
        padding:0px;
        margin:0px;
        width:590px;
        height:20px;
        line-height:20px;
        font-size:12px;
    }
    .version ul li{
        display:block;
        width:33%;
        float:left;
        text-indent:2em
    }
    .version .th{
        background:"gray";
        font-weight:bold;
        border-top:0px
    }
</style>

<script>
    var hosts = ["192.168.2.2","192.168.2.27","192.168.2.23","192.168.2.25"];
    var port = 6010;

    function fetch(host,port) {
        $.ajax({
            type: "GET",
            url: "http://"+host+":"+port+"/version",
            dataType: "json",
            success: function(data){
                var id = "seg-"+host.replace(/\./g,"")+port;
                $("#"+id+">.version").text(data["data"]["version"])
                $("#"+id+">.buildtime").text(data["data"]["buildtime"])
            }
        });
    }

    function renderFrame(hosts, port) {
        var header = "<ul class=\"th\">\n" +
            "        <li>Host</li>\n" +
            "        <li>版本号</li>\n" +
            "        <li>构建时间</li>\n" +
            "    </ul>";
        var template = "<ul id=\"{{id}}\">\n" +
            "        <li>{{host}}</li>\n" +
            "        <li class=\"version\"></li>\n" +
            "        <li class=\"buildtime\"></li>\n" +
            "    </ul>";

        var frame = header;
        for(var i = 0; i < hosts.length; i++){
            var host = hosts[i];
            var id = "seg-"+host.replace(/\./g,"")+port;
            var item = template.replace("{{id}}",id).replace("{{host}}",host+":"+port);
            frame += item
        }

        $(".version").append(frame);
    }

    $(document).ready(function () {
        renderFrame(hosts, port);

        for(var i = 0; i < hosts.length; i++){
            var host = hosts[i];
            window.setInterval(function (host) {
                fetch(host,port)
            },10000,host)
        }

    })
</script>
```
