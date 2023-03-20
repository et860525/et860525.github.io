---
title: "初試 WebSocket"
date: 2023-03-20
draft: false
author: "Chen Yu Fan"
tags: ["Express", "TypeScript", "Backend"]
---

[WebSocket](https://zh.wikipedia.org/zh-tw/WebSocket) 是由 [HTML 5](https://zh.wikipedia.org/zh-tw/HTML5) 所提供用於讓瀏覽器與伺服器進行互動通訊的技術。

WebSocket 只需要連線一次，就能保持與伺服器的**雙向溝通**，無須重新發送 Request，這也讓回應更即時與快速。

<!--more-->

## 歷史

在早期，網站為了實現推播的技術，都是使用**輪詢 ( polling )**。所謂的輪詢就是瀏覽器每隔一段時間 ( 如每秒 ) 像伺服器發出 HTTP Request，伺服器便會回傳最新的資料給客戶端，很明顯的缺點就是瀏覽器必須不斷的發出 Request。

## Why WebSocket ?

![WebSocket-connection.png](/images/WebSocket/WebSocket-connection.png)

- **雙向溝通**：從以上可以得知，在以前都是由 Client 端進行「單向」發送 Request，而無法由 Server 主動發出 Request。而 WebSocket 協定可以讓 Server 端主動向 Client 推播資料，以此實現「雙向溝通」
- **實際使用**：推播、即時聊天室、共同編輯

## 使用 WebSocket

WebSocket 是由瀏覽器使用 JavaScript 來建立的，發起一個 HTTP Request。一般 WebSocket 請求網址為：

```text
ws://example.com/wsapi
```

經過 SSL 加密後就會變成：

```text
wss://secure.example.com/wsapi
```

### 交握 ( Handshaking )

Websocket 通過 HTTP/1.1 協定進行交握。

客戶端 ( Client ) Request：

```HTTP
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

- `GET /chat HTTP/1.1` 使用 HTTP/1.1 協定進行交握
- `Upgrade: websocket` 表示客戶端想升級為 `websocket` 協定
- `Connection: Upgrade` 表示客戶端想連接升級
- `Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==` 計算 SHA-1 摘要，之後進行 Base64 編碼，該結果會成為伺服器回傳「Sec-WebSocket-Accept」頭的值並返回給客戶端，以避免普通 HTTP 被誤認為 WebSocket 協定 ( 每一次握手隨機產生 )
- `Origin: http://example.com` 客戶端的 URL
- `Sec-WebSocket-Protocol: chat, superchat` URL 下不同 Server 需要的協議
- `Sec-WebSocket-Version: 13` 支援的Websocket版本

伺服器 ( Server ) Response：

```HTTP
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

- `HTTP/1.1 101 Switching Protocols` 建立成功
- `Sec-WebSocket-Protocol: chat` 伺服器端使用的協議

# Server 端：建立 WebSocket 環境

先安裝兩個套件：

```bash
pnpm install express
pnpm install ws
```

安裝後，建立 `server.js` 檔案：

```javascript
const express = require('express')
const ServerSocket = require('ws').Server   // 引用 Server

// 指定一個 port
const PORT = 8080

// 建立 express 物件並用來監聽 8080 port
const server = express()
    .listen(PORT, () => console.log(`[Server] Listening on https://localhost:${PORT}`))

// 透過 ServerSocket 開啟 WebSocket 的服務
const wss = new ServerSocket({ server })

// Connection opened
wss.on('connection', ws => {
    console.log('Client connected')

    // Connection closed
    ws.on('close', () => {
        console.log('Close connected')
    })
})
```

使用以下指令執行 Server：

```bash
node server.js
```

# Client 端：連線到 WebSocket Server

建立完 Server 後，接下來要建立 Client 來連線到 WebSocket Server。建立一個新的專案，裡面會有 `index.html` 與 `index.js` 兩個檔案。

首先，先建立 `index.html`：

```html
<html>
    <head>
    </head>
    <body>
        <!-- Connect or Disconnect WebSocket Server -->
        <button id="connect">Connect</button>
        <button id="disconnect">Disconnect</button>
        
        <!-- Send Message to Server -->
        <div>
            Message: <input type="text" id="sendMsg" ><button id="sendBtn">Send</button>
        </div>
        
        <!-- Import index.js after UI rendered -->
        <script src='./index.js'></script>
    </body>
</html>
```

新增一個 `index.js` 檔案來處理邏輯：

```javascript
var ws

// 監聽 click 事件
document.querySelector('#connect')?.addEventListener('click', (e) => {
    console.log('[click connect]')
    connect()
})

document.querySelector('#disconnect')?.addEventListener('click', (e) => {
    console.log('[click disconnect]')
    disconnect()
})

document.querySelector('#sendBtn')?.addEventListener('click', (e) => {
    const msg = document.querySelector('#sendMsg')
    sendMessage(msg?.value)
})

function connect() { 
    // Create WebSocket connection
    ws = new WebSocket('ws://localhost:8080') 
    // 在開啟連線時執行
    ws.onopen = () => console.log('[open connection]')
}

function disconnect() {
    ws.close()
    // 在關閉連線時執行
    ws.onclose = () => console.log('[close connection]')
}
```

如果使用 `console.log(ws)`  可以看到物件：

![WebSocket-ws-object.png](/images/WebSocket/WebSocket-ws-object.png)

**readyState** 有四種 code：

- **0**：連接尚未建立
- **1**：連接建立，可以進行通訊
- **2**：連接正在進行關閉
- **3**：連接已經關閉或連接打不開

這裡可以看到 WebSocket 有四個事件：

- **onopen**
- **onerror**
- **onclose**
- **onmessage**

## 監聽 WebSocket 事件

#### open

與 WebSocket Server 連接建立時會觸發：

```js
ws.addEventListener('open', function() {
    console.log('連結建立成功。')
})
```

#### error

通信錯誤時觸發

#### close

關閉 WebSocket Server 連線時觸發：

```js
ws.addEventListener('close', function() {
    console.log('連結關閉。')
})
```

#### message

監聽由 Server 主動發來的訊息：

```js
ws.addEventListener('message', function(e) {
    var msg = JSON.parse(e.data);
    console.log(msg)
})
```

以下有更詳細的 Message 處理。

# 處理 Message

因為 WebSocket 可以雙向溝通，所以 Server 端可以發送訊息給 Client 端，Client 端也可以發送訊息給 Server 端。

## Server 端

Server 端使用 `send` 發送訊息，Client 透過監聽 `message` 事件來接收訊息：

```javascript
// Connection opened
wss.on('connection', ws => {
    console.log('Client connected')

    // Listen for messages from client
    ws.on('message', data => {
      console.log('[Message from client]: ', data)
      // Send message to client
      ws.send('[Get message from server]')
    })

    // ...
})
```

## Client 端

Clinet 端使用 `send` 發送訊息，Server 端使用 `onmessage` 接收訊息：

```javascript
// Client: 監聽 click 事件
document.querySelector('#sendBtn')?.addEventListener('click', (e) => {
    const msg = document.querySelector('#sendMsg')
    sendMessage(msg?.value)
})

// Server: Listen for messages
function sendMessage(msg) {
    // Send messages to Server
    ws.send(msg)
    // Listen for messages from Server
    ws.onmessage = event => console.log('[send message]', event)
}
```

# 一個簡單的即時聊天室

> 以下的程式碼實作會放在 [et860525/websocket-test](https://github.com/et860525/websocket-test)

WebSocket 可以運用在聊天室等功能，實現一個 Server 同時讓多個 Client 連線，並讓 Client A 傳送訊息給 Server 的同時，讓 Client B 接收來自 Server 的訊息。

這時候就要使用**廣播功能**，首先透過 `ws` 提供的方法來讓 Client 取得目前所有其他 Clients 的資訊，再使用 `forEach` 迴圈送出訊息給每一個 Client：

```javascript
// Connection opened
wss.on('connection', ws => {
    console.log('Client connected')

    // Listen for messages from client
    ws.on('message', data => {
      console.log('[Message from client]: ', data)
      // Get clients who connected
      let clients = wss.clients
      // Use loop for sending messages to each client
      clients.forEach(client => {
        client.send('[Broadcast][Get message from server]')
      })
    })
	
    // ...
})
```

那要如何給不同的 Client 一個獨一無二的 ID 呢？

根據 [unique identifier for each client request to websocket server](https://github.com/websockets/ws/issues/859)，這裡可以直接取得 request header 中的 `sec-websocket-key` 給每個 Client 不同的 ID：

```javascript
wss.on('connection', (ws, req) => {
  var id = req.headers['sec-websocket-key']
  // Do something...
})
```

以下為完整的程式碼：

## Server 端

`server.js` 
```javascript
// import library
const express = require('express')
const ServerSocket = require('ws').Server   // 引用 Server

const PORT = 8080

// 建立 express 物件並用來監聽 8080 port
const server = express()
	.listen(PORT, () => console.log(`[Server] Listening on https://localhost:${PORT}`))

// 建立實體，透過 ServerSocket 開啟 WebSocket 的服務
const wss = new ServerSocket({ server })

// Connection opened
wss.on('connection', (ws, req) => {
    ws.id = req.headers['sec-websocket-key'].substring(0, 8)
    ws.send(`[Client ${ws.id} is connected!]`)
	
    // Listen for messages from client
    ws.on('message', data => {
      console.log('[Message from client] data: ', data)
      // Get clients who has connected
      let clients = wss.clients
      // Use loop for sending messages to each client
      clients.forEach(client => {
        client.send(`${ws.id}: ` + data)
      })
    })
	
	  // Connection closed
    ws.on('close', () => {
      console.log('[Close connected]')
    })
})
```

嘗試使用 TypeScript 來建構 Server 端 ( `server.ts` )：

```typescript
import express from 'express';
import WebSocket, { Server } from 'ws';
const PORT = 8080;
// 建立 express 物件並用來監聽 8080 port
const server = express().listen(PORT, () =>
    console.log(`[Server] Listening on https://localhost:${PORT}`)
);

// 透過 Server 開啟 WebSocket 的服務
const wss = new Server({ server: server });

// Connection opened
wss.on('connection', (ws: WebSocket, req) => {
    let id = req.headers['sec-websocket-key'];
    if (id) id = id.substring(0, 8);

    ws.send(`[Client ${id} is connected!]`);

    // Listen for messages from client
    ws.on('message', (data) => {
      console.log('[Message from client] data: ', data);
      // Get clients who has connected
      let clients = wss.clients;
      // Use loop for sending messages to each client
      clients.forEach((client) => {
        client.send(`${id}: ` + data);
    });
});

// Connection closed
ws.on('close', () => {
    console.log('[Close connected]');
    });
});
```

## Client 端 

`index.js` 
```javascript
var ws

// 監聽 click 事件
document.querySelector('#connect')?.addEventListener('click', (e) => {
    connect()
})

document.querySelector('#disconnect')?.addEventListener('click', (e) => {
    disconnect()
})

document.querySelector('#sendBtn')?.addEventListener('click', (e) => {
    const msg = document.querySelector('#sendMsg')
    sendMessage(msg?.value)
})

function connect() { 
// Create WebSocket connection
ws = new WebSocket('ws://localhost:8080') 
// 也可以連線到指定的 IP
// ws = new WebSocket('ws://192.168.17.35:58095') 

// 在開啟連線時執行
    ws.onopen = () => {
      console.log('[open connection]')
      // Listen for messages from Server
      ws.onmessage = event => {
        console.log(`[Message from server]:\n %c${event.data}` , 'color: blue')
      }
    }
}

function sendMessage(msg) {
    // Send messages to Server
    ws.send(msg)
}

function disconnect() {
    ws.close()
    // 在關閉連線時執行
    ws.onclose = () => console.log('[close connection]')
}
```

# Socket vs WebSocket vs Socket.io

![WebSocket-compare.png](/images/WebSocket/WebSocket-compare.png)

## Socket

- 使用者可以透過 Socket 來操作 TCP/IP 協議
- 定義在 **傳輸層跟應用層之間**
- 傳輸方式為 **stream**
- 傳輸載體為 **binary**

## WebSocket

- 為了讓 Web 即時雙向通訊而創造的協議
- 屬於**應用層**
- 瀏覽器底層使用 socket 當作通訊界面
- 載體有兩種：**binary**或**text**，只能擇一使用

## Socket.io

- Socket.io 是一個框架一個套件
- 建立在 WebSocket 上的套件，支持代理和負載平衡器
- 兩部分組成：Server 端為 Node.js；Client 端為 JavaScript
- **socket.io**並沒有任何連線功能，主要是靠核心 [engine.io](https://github.com/socketio/engine.io) 來連接伺服器與客戶端

# 結語

研究了一下 WebSocket 技術，其重點為：客戶端可以不用為了確認資料在伺服器裡的狀態，而不斷的送出 Request，而是當伺服器裡的資料狀態更新時，伺服器可以主動的將資料推播給客戶端。

# Reference

- [淺談 WebSocket 協定：實作一個簡單的即時聊天室吧！](https://hackmd.io/@Heidi-Liu/javascript-websocket)
- [JavaScript | WebSocket 讓前後端沒有距離](https://medium.com/enjoy-life-enjoy-coding/javascript-websocket-%E8%AE%93%E5%89%8D%E5%BE%8C%E7%AB%AF%E6%B2%92%E6%9C%89%E8%B7%9D%E9%9B%A2-34536c333e1b)
- [Socket，Websocket，Socket.io的差異](https://leesonhsu.blogspot.com/2018/07/socketwebsocketsocketio.html)
- [WebSocket 基本介紹及使用筆記](https://www.letswrite.tw/websocket/)