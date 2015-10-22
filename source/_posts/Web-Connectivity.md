title: "传统轮询、长轮询、服务器发送事件与WebSocket"
date: 2015-10-22 14:27:00
categories:
- Front-End
- JavaScript
tags:
- Server
- JavaScript
---

构建网络应用的过程中，我们经常需要与服务器进行持续的通讯以保持双方信息的同步。通常这种持久通讯在不刷新页面的情况下进行，消耗一定的内存资源常驻后台，并且对于用户不可见。本文将简要介绍Web通信中常用的四种方式。

<!-- more -->

### 传统轮询(Traditional Polling)
当前Web应用中较常见的一种持续通信方式，通常采取setInterval或者setTimeout实现。例如如果我们想要定时获取并刷新页面上的数据，可以结合Ajax写出如下实现：
```javascript
setInterval(function() {
    $.get("/path/to/server", function(data, status) {
        console.log(data);
    });
}, 10000);
```
上面的程序会每隔10秒向服务器请求一次数据，并在数据到达后存储。这个实现方法通常可以满足简单的需求，然而同时也存在着很大的缺陷：在网络情况不稳定的情况下，服务器从接收请求、发送请求到客户端接收请求的总时间有可能超过10秒，而请求是以10秒间隔发送的，这样会导致接收的数据到达先后顺序与发送顺序不一致。于是出现了采用setTimeout的轮询方式：
```javascript
function poll() {
    setTimeout(function() {
        $.get("/path/to/server", function(data, status) {
            console.log(data);
            // 发起下一次请求
            poll();
        });
    }, 10000);
}
```
程序首先设置10秒后发起请求，当数据返回后再隔10秒发起第二次请求，以此类推。这样的话虽然无法保证两次请求之间的时间间隔为固定值，但是可以保证到达数据的顺序。

### 长轮询(Long Polling)
上面两种传统的轮询方式都存在一个严重缺陷：程序在每次请求时都会新建一个HTTP请求，然而并不是每次都能返回所需的新数据。当同时发起的请求达到一定数目时，会对服务器造成较大负担。这时我们可以采用长轮询方式解决这个问题。
> **注意**
>
> 长轮询与以下将要提到的服务器发送事件和WebSocket不能仅仅依靠客户端JavaScript实现，我们同时需要服务器支持并实现相应的技术。

长轮询的基本思想是在每次客户端发出请求后，服务器检查上次返回的数据与此次请求时的数据之间是否有更新，如果有更新则返回新数据并结束此次连接，否则服务器“hold”住此次连接，直到有新数据时再返回相应。而这种长时间的保持连接可以通过设置一个较大的HTTP timeout实现。下面是一个简单的长连接示例：

服务器（PHP）：
```php
<?php
    // 示例数据为data.txt
    $filename= dirname(__FILE__)."/data.txt";
    // 从请求参数中获取上次请求到的数据的时间戳
    $lastmodif = isset( $_GET["timestamp"])? $_GET["timestamp"]: 0 ;
    // 将文件的最后一次修改时间作为当前数据的时间戳
    $currentmodif = filemtime($filename);

    // 当上次请求到的数据的时间戳*不旧于*当前文件的时间戳，使用循环"hold"住当前连接，并不断获取文件的修改时间
    while ($currentmodif <= $lastmodif) {
        // 每次刷新文件信息的时间间隔为10秒
        usleep(10000);
        // 清除文件信息缓存，保证每次获取的修改时间都是最新的修改时间
        clearstatcache();
        $currentmodif = filemtime($filename);
    }

    // 返回数据和最新的时间戳，结束此次连接
    $response = array();
    $response["msg"] =Date("h:i:s")." ".file_get_contents($filename);
    $response["timestamp"]= $currentmodif;
    echo json_encode($response);
?>
```

客户端：
```javascript
function longPoll (timestamp) {
    var _timestamp;
    $.get("/path/to/server?timestamp=" + timestamp)
    .done(function(res) {
        try {
            var data = JSON.parse(res);
            console.log(data.msg);
            _timestamp = data.timestamp;
        } catch (e) {}
    })
    .always(function() {
        setTimeout(function() {
            longPoll(_timestamp || Date.now()/1000);
        }, 10000);
    });
}
```
长轮询可以有效地解决传统轮询带来的带宽浪费，但是每次连接的保持是以消耗服务器资源为代价的。尤其对于Apache+PHP服务器，由于有默认的“worker threads”数目的限制，当长连接较多时，服务器便无法对新请求进行相应。

### 服务器发送事件(Server-Sent Event)
服务器发送事件（以下简称SSE）是HTML 5规范的一个组成部分，可以实现服务器到客户端的单向数据通信。通过SSE，客户端可以自动获取数据更新，而不用重复发送HTTP请求。一旦连接建立，“事件”便会自动被推送到客户端。服务器端SSE通过“事件流(Event Stream)”的格式产生并推送事件。事件流对应的MIME类型为“text/event-stream”，包含四个字段：event、data、id和retry。event表示事件类型，data表示消息内容，id用于设置客户端EventSource对象的“last event ID string”内部属性，retry指定了重新连接的时间。

服务器（PHP）：
```php
<?php
    header("Content-Type: text/event-stream");
    header("Cache-Control: no-cache");
    // 每隔1秒发送一次服务器的当前时间
    while (1) {
        $time = date("r");
        echo "event: ping\n";
        echo "data: The server time is: {$time}\n\n";
        ob_flush();
        flush();
        sleep(1);
    }
?>
```
客户端中，SSE借由EventSource对象实现。EventSource包含五个外部属性：onerror, onmessage, onopen, readyState、url，以及两个内部属性：“reconnection time”与“last event ID string”。在onerror属性中我们可以对错误捕获和处理，而onmessage则对应着服务器事件的接收和处理。另外也可以使用addEventListener方法来监听服务器发送事件，根据event字段区分处理。

客户端：
```javascript
var eventSource = new EventSource("/path/to/server");
eventSource.onmessage = function (e) {
    console.log(e.event, e.data);
}
// 或者
eventSource.addEventListener("ping", function(e) {
    console.log(e.event, e.data);
}, false);
```
SSE相较于轮询具有较好的实时性，使用方法也非常简便。然而SSE只支持服务器到客户端单向的事件推送，而且所有版本的IE（包括到目前为止的Microsoft Edge）都不支持SSE。如果需要强行支持IE和部分移动端浏览器，可以尝试[EventSource Polyfill](https://github.com/Yaffle/EventSource)（本质上仍然是轮询）。SSE的浏览器支持情况如下图所示：
![SSE Support](/images/server-sent-event.JPG)

### WebSocket
WebSocket同样是HTML 5规范的组成部分之一，现标准版本为RFC 6455。WebSocket相较于上述几种连接方式，实现原理较为复杂，用一句话概括就是：客户端向WebSocket服务器通知（notify）一个带有所有接收者ID（recipients IDs）的事件（event），服务器接收后立即通知所有活跃的（active）客户端，只有ID在接收者ID序列中的客户端才会处理这个事件。由于WebSocket本身是基于TCP协议的，所以在服务器端我们可以采用构建TCP Socket服务器的方式来构建WebSocket服务器。这里为了略过协议解析的具体细节，我们采用Node.js的ws库来实现简单的WebSocket服务器。

服务器（Node.js）：
```javascript
var WebSocketServer = require('ws').Server;
var wss = new WebSocketServer({port: 8080});

wss.on("connection", function(socket) {
    socket.on("message", function(msg) {
        console.log(msg);
        socket.send("Nice to meet you!");
    });
});
```

客户端同样可以使用Node.js或者是浏览器实现，这里选用浏览器作为客户端：
```javascript
// WebSocket 为客户端JavaScript的原生对象
var ws = new WebSocket("ws://localhost:8080");
ws.onopen = function (event) {
    ws.send("Hello there!");
}
ws.onmessage = function (event) {
    console.log(event.data);
}
```
WebSocket同样具有实时性，每次通讯无需重发请求头部，节省带宽，而且它的浏览器支持非常好（详见下图）。
![SSE Support](/images/websocket.JPG)

下面总结一下四种通信方式的优缺点：

|>>>>>>>>>>>>  | 传统轮询                                | 长轮询                               | 服务器发送事件                                       | WebSocket                                                                                             |
|------------|-----------------------------------------|--------------------------------------|------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| 浏览器支持 | 几乎所有现代浏览器                      | 几乎所有现代浏览器                   | Firefox 6+ Chrome 6+ Safari 5+ Opera 10.1+           | IE 10+ Edge Firefox 4+ Chrome 4+ Safari 5+ Opera 11.5+                                                |
| 服务器负载 | 较少的CPU资源，较多的内存资源和带宽资源 | 与传统轮询相似，但是占用带宽较少     | 与长轮询相似，除非每次发送请求后服务器不需要断开连接 | 无需循环等待（长轮询），CPU和内存资源不以客户端数量衡量，而是以客户端事件数衡量。四种方式里性能最佳。 |
| 客户端负载 | 占用较多的内存资源与请求数。            | 与传统轮询相似。                     | 浏览器中原生实现，占用资源很小。                     | 同Server-Sent Event。                                                                                 |
| 延迟       | 非实时，延迟取决于请求间隔。            | 同传统轮询。                         | 非实时，默认3秒延迟，延迟可自定义。                  | 实时。                                                                                                |
| 实现复杂度 | 非常简单。                              | 需要服务器配合，客户端实现非常简单。 | 需要服务器配合，而客户端实现甚至比前两种更简单。     | 需要Socket程序实现和额外端口，客户端实现简单。                                                        |


最后分享一个通（ji）俗（qi）易（dou）懂（bi）的介绍轮询和WebSocket的文章：[知乎：WebSocket 是什么原理？为什么可以实现持久连接？](http://zhi.hu/gECL);