---
title: "XMLHttpRequest 到 Fetch，SSE，WebSocket - 金山课件"
source: "https://campus.wps.cn/contentpreview/cc1e203f-ebb8-42f9-acaa-e05eeaf7cf42"
author:
published:
created: 2026-05-11
description: "学生最好的课件系统"
tags:
  - "clippings"
---
XMLHttpRequest 到 Fetch，SSE，WebSocket

![](https://github.com/shadcn.png)

## XMLHttpRequest 到 Fetch，SSE，WebSocket

> **基于 [现代 JavaScript 教程](https://zh.javascript.info/network) 与 [MDN Web Docs](https://developer.mozilla.org/zh-CN/docs/Web/API) 编写**  
> 从传统的 XMLHttpRequest 到现代的 Fetch API，从单向推送的 SSE 到全双工的 WebSocket —— 全面掌握浏览器网络通信的四大核心技术。

## 第一章 浏览器网络通信全景

### 1.1 为什么需要网络通信

现代 Web 应用早已不是"点击链接 → 刷新整页"的时代了。我们需要：

- **异步加载数据** —— 不刷新页面就能获取服务器数据（AJAX）
- **实时推送通知** —— 服务器有新消息时立即通知浏览器
- **双向实时交互** —— 聊天室、协同编辑、在线游戏等场景

JavaScript 提供了一整套网络 API 来满足这些需求：
```
传统请求-响应          实时单向推送           实时双向通信
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ XMLHttp-    │    │ Server-Sent │    │  WebSocket  │
│ Request     │    │ Events(SSE) │    │             │
│             │    │             │    │             │
│ Fetch API   │    │ EventSource │    │ ws:// / wss │
└─────────────┘    └─────────────┘    └─────────────┘
    HTTP               HTTP              WebSocket
  请求-响应          服务器→客户端         客户端⇄服务器
```

### 1.2 技术演进时间线

```
1999    XMLHttpRequest 诞生（IE5）
2004    Gmail 让 AJAX 家喻户晓
2006    XMLHttpRequest Level 2（进度事件、跨域等）
2011    Server-Sent Events 规范
2011    WebSocket 协议 RFC 6455
2015    Fetch API 进入标准（基于 Promise）
2017    async/await 让 Fetch 写法更优雅
2020+   Fetch API 持续完善（AbortController、Streams 等）
```

### 1.3 AJAX 的本质

AJAX（ **A** synchronous **J** avaScript **A** nd **X** ML）是一个术语，而不是一项具体技术。它的核心思想是：

> 使用 JavaScript 在后台发起 HTTP 请求，获取数据后局部更新页面，无需整页刷新。
尽管名字里有"XML"，但现在绝大多数场景都使用 JSON 格式。实现 AJAX 的技术手段主要有两个： `XMLHttpRequest` 和 `Fetch API` 。
[[计算机网络/HTTP]]
## 第二章 XMLHttpRequest：经典的网络请求

### 2.1 认识 XMLHttpRequest

`XMLHttpRequest` （简称 XHR）是一个内建的浏览器对象，它允许使用 JavaScript 发送 HTTP 请求。虽然名字里有"XML"，但它可以操作 **任何数据** ，而不仅仅是 XML。

现如今，更现代的 `fetch` 方法在很大程度上取代了 XHR。但在以下场景中，XHR 仍然有用：

1. **跟踪上传进度** —— `fetch` 目前无法监听上传进度（只能监听下载）
2. **兼容旧浏览器** —— 不想引入 polyfill 时
3. **维护遗留代码** —— 大量既有项目仍在使用 XHR

### 2.2 基本用法：三步走

发送一个 XHR 请求只需三步： **创建 → 配置 → 发送** 。

javascript 复制代码

```javascript
// 第一步：创建 XMLHttpRequest 对象
let xhr = new XMLHttpRequest();

// 第二步：配置请求（HTTP 方法、URL）
xhr.open('GET', '/api/users');

// 第三步：发送请求
xhr.send();

// 监听响应
xhr.onload = function() {
  if (xhr.status === 200) {
    console.log('响应数据:', xhr.response);
  } else {
    console.error('请求失败:', xhr.status, xhr.statusText);
  }
};

xhr.onerror = function() {
  console.error('网络错误，无法发出请求');
};
```

#### open() 方法详解

javascript 复制代码

```javascript
xhr.open(method, URL, [async, user, password]);
```

| 参数 | 说明 |
| --- | --- |
| `method` | HTTP 方法： `"GET"` 、 `"POST"` 、 `"PUT"` 、 `"DELETE"` 等 |
| `URL` | 请求的 URL，可以是字符串或 `URL` 对象 |
| `async` | 是否异步，默认 `true` （几乎不应该设为 `false` ） |
| `user` | HTTP 基本认证的用户名（可选） |
| `password` | HTTP 基本认证的密码（可选） |

> **注意** ： `open()` 只是 **配置** 请求，并不会建立连接。真正的网络活动在调用 `send()` 时才开始。

#### send() 方法

javascript 复制代码

```javascript
xhr.send([body]);
```
- `GET` 请求通常不需要 body，直接 `xhr.send()`
- `POST` 请求将数据放在 body 中发送

### 2.3 三大核心事件

javascript 复制代码

```javascript
xhr.onload = function() {
  // 请求完成（即使 HTTP 状态码是 4xx 或 5xx）
  // 响应已完全下载
  console.log(\`状态码: ${xhr.status}, 数据: ${xhr.response}\`);
};

xhr.onerror = function() {
  // 无法发出请求（网络中断、DNS 解析失败等）
  // 注意：HTTP 错误（如 404）不会触发此事件
  console.error('网络错误');
};

xhr.onprogress = function(event) {
  // 下载过程中周期性触发
  if (event.lengthComputable) {
    console.log(\`已下载 ${event.loaded} / ${event.total} 字节\`);
  } else {
    console.log(\`已下载 ${event.loaded} 字节\`);
  }
};
```

### 2.4 响应属性

当服务器响应后，可以从以下属性获取结果：

| 属性 | 说明 |
| --- | --- |
| `xhr.status` | HTTP 状态码（数字），如 `200` 、 `404` 、 `500` ；非 HTTP 错误时为 `0` |
| `xhr.statusText` | HTTP 状态消息（字符串），如 `"OK"` 、 `"Not Found"` |
| `xhr.response` | 服务器响应体（格式取决于 `responseType` ） |
| `xhr.responseText` | 以字符串形式获取响应（旧写法） |

### 2.5 响应类型（responseType）

通过 `xhr.responseType` 属性可以指定期望的响应格式：

javascript 复制代码

```javascript
xhr.responseType = 'json'; // 自动解析为 JavaScript 对象
```

| 值               | 响应数据类型                  |
| --------------- | ----------------------- |
| `""` （默认）       | 字符串                     |
| `"text"`        | 字符串                     |
| `"json"`        | JSON 自动解析为对象            |
| `"blob"`        | `Blob` 对象（二进制数据）        |
| `"arraybuffer"` | `ArrayBuffer` （底层二进制数据） |
| `"document"`    | XML/HTML Document 对象    |

**示例：获取 JSON 数据**
```javascript
let xhr = new XMLHttpRequest();
xhr.open('GET', '/api/user/1');
xhr.responseType = 'json';
xhr.send();

xhr.onload = function() {
  // xhr.response 已经是解析好的 JavaScript 对象
  let user = xhr.response;
  console.log(user.name); // "张三"
};
```

### 2.6 readyState 与生命周期

XHR 对象有一个 `readyState` 属性，反映了请求的处理进度：

javascript 复制代码

```javascript
0 - UNSENT           // 对象刚创建
1 - OPENED           // open() 已调用
2 - HEADERS_RECEIVED // 收到响应头
3 - LOADING          // 正在接收响应体（可能触发多次）
4 - DONE             // 请求完成
```

可以用 `readystatechange` 事件监听状态变化（ **老式写法** ，现已被 `load/error/progress` 取代）：

javascript 复制代码

```javascript
xhr.onreadystatechange = function() {
  if (xhr.readyState === 4) {
    if (xhr.status === 200) {
      console.log(xhr.responseText);
    }
  }
};
```

**完整的事件生命周期（按触发顺序）** ：

复制代码

```
loadstart → progress → progress → ... → load/error/abort/timeout → loadend
```
- `loadstart` —— 请求开始
- `progress` —— 每收到一个数据包触发一次
- `load` —— 请求成功完成
- `error` —— 网络层错误
- `abort` —— 调用 `xhr.abort()` 取消
- `timeout` —— 超时取消
- `loadend` —— 在 `load` / `error` / `abort` / `timeout` 之后触发

> `load` 、 `error` 、 `abort` 、 `timeout` 四个事件 **互斥** ，同一请求只会触发其中一个。

### 2.7 超时与中止

javascript 复制代码

```javascript
// 设置超时时间（毫秒）
xhr.timeout = 10000; // 10 秒

xhr.ontimeout = function() {
  console.error('请求超时');
};

// 中止请求
xhr.abort();

xhr.onabort = function() {
  console.log('请求已取消');
};
```

### 2.8 HTTP Header 操作

javascript 复制代码

```javascript
// 设置请求头（必须在 open() 之后、send() 之前调用）
xhr.setRequestHeader('Content-Type', 'application/json');
xhr.setRequestHeader('X-Custom-Header', 'hello');

// 注意：同名 header 不会覆盖，而是追加
xhr.setRequestHeader('X-Auth', '123');
xhr.setRequestHeader('X-Auth', '456');
// 结果: X-Auth: 123, 456

// 获取响应头
let contentType = xhr.getResponseHeader('Content-Type');

// 获取所有响应头（Set-Cookie 和 Set-Cookie2 除外）
let allHeaders = xhr.getAllResponseHeaders();
// 返回格式："Header-Name: value\r\n..."
```

**解析所有响应头为对象：**

javascript 复制代码

```javascript
let headers = xhr.getAllResponseHeaders()
  .split('\r\n')
  .reduce((result, line) => {
    let [name, value] = line.split(': ');
    if (name) result[name] = value;
    return result;
  }, {});
```

### 2.9 POST 请求与发送数据

#### 发送 JSON

javascript 复制代码

```javascript
let xhr = new XMLHttpRequest();
xhr.open('POST', '/api/users');

xhr.setRequestHeader('Content-Type', 'application/json; charset=utf-8');

let data = { name: '张三', age: 25 };
xhr.send(JSON.stringify(data));

xhr.onload = function() {
  console.log(xhr.response);
};
```

#### 发送 FormData

javascript 复制代码

```javascript
let formData = new FormData();
formData.append('name', '张三');
formData.append('avatar', fileInput.files[0]); // 上传文件

let xhr = new XMLHttpRequest();
xhr.open('POST', '/api/upload');
// 不需要手动设置 Content-Type，FormData 会自动设置为 multipart/form-data
xhr.send(formData);
```

#### 从表单元素创建 FormData

html 复制代码

```html
<form id="myForm">
  <input name="name" value="张三">
  <input name="email" value="zhangsan@example.com">
</form>

<script>
let formData = new FormData(document.getElementById('myForm'));
formData.append('extra', '附加数据');

let xhr = new XMLHttpRequest();
xhr.open('POST', '/api/submit');
xhr.send(formData);
</script>
```

### 2.10 上传进度跟踪

`xhr.onprogress` 只跟踪 **下载** 进度。要跟踪 **上传** 进度，需要使用 `xhr.upload` 对象：

javascript 复制代码

```javascript
function uploadFile(file) {
  let xhr = new XMLHttpRequest();

  // 上传进度
  xhr.upload.onprogress = function(event) {
    if (event.lengthComputable) {
      let percent = Math.round((event.loaded / event.total) * 100);
      console.log(\`上传进度: ${percent}%\`);
      progressBar.value = percent;
    }
  };

  xhr.upload.onload = function() {
    console.log('上传完成');
  };

  xhr.upload.onerror = function() {
    console.error('上传出错');
  };

  // 服务器响应
  xhr.onload = function() {
    if (xhr.status === 200) {
      console.log('服务器确认:', xhr.response);
    }
  };

  xhr.open('POST', '/api/upload');
  xhr.send(file);
}
```

> **这是 XHR 相对于 Fetch 的一大优势** ： `fetch` 目前没有原生的上传进度跟踪能力。

### 2.11 跨域请求

XHR 使用与 `fetch` 相同的 CORS（跨域资源共享）策略：

javascript 复制代码

```javascript
let xhr = new XMLHttpRequest();
xhr.open('GET', 'https://api.other-domain.com/data');

// 如需携带 Cookie 和认证信息
xhr.withCredentials = true;

xhr.send();
```

设置 `withCredentials = true` 后，服务器必须返回：

- `Access-Control-Allow-Origin` （不能是 `*` ，必须指定具体域名）
- `Access-Control-Allow-Credentials: true`

### 2.12 同步请求（不推荐）

将 `open()` 的第三个参数设为 `false` 可发起同步请求：

javascript 复制代码

```javascript
let xhr = new XMLHttpRequest();
xhr.open('GET', '/api/data', false); // false = 同步

try {
  xhr.send();
  if (xhr.status === 200) {
    console.log(xhr.response);
  }
} catch (err) {
  console.error('请求失败');
}
```

> **强烈不推荐使用同步请求** ：它会阻塞整个页面的 JavaScript 执行，导致页面"卡死"。现代浏览器在主线程上的同步 XHR 已被标记为废弃（deprecated），并会在控制台发出警告。

### 2.13 封装 XHR 为 Promise

在实际项目中，将 XHR 包装成 Promise 可以获得更好的使用体验：

javascript 复制代码

```javascript
function request(method, url, data = null, options = {}) {
  return new Promise((resolve, reject) => {
    let xhr = new XMLHttpRequest();
    xhr.open(method, url);

    // 设置响应类型
    xhr.responseType = options.responseType || 'json';

    // 设置超时
    if (options.timeout) {
      xhr.timeout = options.timeout;
    }

    // 设置请求头
    if (options.headers) {
      for (let [key, value] of Object.entries(options.headers)) {
        xhr.setRequestHeader(key, value);
      }
    }

    xhr.onload = function() {
      if (xhr.status >= 200 && xhr.status < 300) {
        resolve(xhr.response);
      } else {
        reject(new Error(\`HTTP ${xhr.status}: ${xhr.statusText}\`));
      }
    };

    xhr.onerror = () => reject(new Error('网络错误'));
    xhr.ontimeout = () => reject(new Error('请求超时'));

    // 上传进度回调
    if (options.onUploadProgress) {
      xhr.upload.onprogress = options.onUploadProgress;
    }

    // 下载进度回调
    if (options.onDownloadProgress) {
      xhr.onprogress = options.onDownloadProgress;
    }

    if (data && typeof data === 'object' && !(data instanceof FormData)) {
      xhr.setRequestHeader('Content-Type', 'application/json; charset=utf-8');
      xhr.send(JSON.stringify(data));
    } else {
      xhr.send(data);
    }
  });
}

// 使用示例
async function loadUsers() {
  try {
    let users = await request('GET', '/api/users');
    console.log(users);
  } catch (err) {
    console.error(err.message);
  }
}
```

## 第三章 Fetch API：现代的网络请求

### 3.1 为什么需要 Fetch

`fetch()` 是浏览器提供的现代网络请求方法，相比 XHR 有以下优势：

| 特性                 | XMLHttpRequest | Fetch API          |
| ------------------ | -------------- | ------------------ |
| API 风格             | 基于事件回调         | 基于 Promise         |
| 语法简洁度              | 冗长             | 简洁                 |
| 与 async/await 配合   | 需要手动封装         | 天然支持               |
| Stream 支持          | 无              | 原生 ReadableStream  |
| 请求/响应对象            | 无独立对象          | Request / Response |
| Service Worker 中可用 | 否              | 是                  |
| 上传进度               | ✅ 原生支持         | ❌ 暂不支持             |

### 3.2 基本语法

```javascript
let promise = fetch(url, [options]);
```
- `url` —— 要请求的 URL
- `options` —— 可选配置对象（method、headers、body 等）

不传 `options` 时就是一个简单的 GET 请求。

### 3.3 两阶段响应处理

Fetch 获取响应分为 **两个阶段** ：

**第一阶段** ：服务器发送响应头后， `fetch` 返回的 Promise 就会 resolve，得到一个 `Response` 对象。此时我们可以检查状态码和响应头，但 **还没有响应体** 。

```javascript
let response = await fetch('/api/users');

// 第一阶段：检查响应头和状态
console.log(response.status);    // 200
console.log(response.ok);        // true（状态码 200-299）
console.log(response.headers.get('Content-Type'));
```

**第二阶段** ：调用 body 读取方法来获取响应体。

javascript 复制代码

```javascript
// 第二阶段：读取响应体
let data = await response.json(); // 解析为 JSON
```

#### 响应体读取方法

| 方法 | 返回值 | 适用场景 |
| --- | --- | --- |
| `response.text()` | 字符串 | 纯文本、HTML |
| `response.json()` | JS 对象/数组 | JSON 数据（最常用） |
| `response.blob()` | Blob 对象 | 图片、文件等二进制数据 |
| `response.arrayBuffer()` | ArrayBuffer | 底层二进制处理 |
| `response.formData()` | FormData 对象 | 表单数据 |
| `response.body` | ReadableStream | 流式读取大数据 |

> **重要** ：响应体只能读取一次！调用上述任一方法后，再次调用其他方法会失败。

javascript 复制代码

```javascript
let text = await response.text();     // ✅ 第一次读取
let json = await response.json();     // ❌ 失败！body 已被消费
```

### 3.4 完整的 GET 请求示例

javascript 复制代码

```javascript
async function getUsers() {
  try {
    let response = await fetch('/api/users');

    if (!response.ok) {
      throw new Error(\`HTTP 错误: ${response.status}\`);
    }

    let users = await response.json();
    console.log(users);
    return users;

  } catch (err) {
    console.error('请求失败:', err.message);
  }
}
```

用 Promise 链写法（不使用 async/await）：

javascript 复制代码

```javascript
fetch('/api/users')
  .then(response => {
    if (!response.ok) throw new Error(\`HTTP 错误: ${response.status}\`);
    return response.json();
  })
  .then(users => console.log(users))
  .catch(err => console.error('请求失败:', err.message));
```

### 3.5 Response 对象详解

javascript 复制代码

```javascript
let response = await fetch(url);

// 状态信息
response.status;       // HTTP 状态码: 200, 404, 500...
response.statusText;   // 状态文本: "OK", "Not Found"...
response.ok;           // 是否成功: true（200-299）或 false
response.url;          // 最终 URL（考虑重定向后的）
response.redirected;   // 是否经过重定向
response.type;         // 响应类型: "basic", "cors", "opaque"...

// 响应头（类似 Map 的对象）
response.headers.get('Content-Type');   // 获取单个 header
response.headers.has('Content-Type');   // 检查是否存在

// 遍历所有响应头
for (let [key, value] of response.headers) {
  console.log(\`${key}: ${value}\`);
}
```

### 3.6 POST 请求

#### 发送 JSON

javascript 复制代码

```javascript
let response = await fetch('/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json; charset=utf-8'
  },
  body: JSON.stringify({
    name: '张三',
    age: 25
  })
});

let result = await response.json();
```

> 当 body 是字符串时， `Content-Type` 默认为 `text/plain;charset=UTF-8` 。发送 JSON 时需要手动设置为 `application/json` 。

#### 发送 FormData（支持文件上传）

javascript 复制代码

```javascript
let formData = new FormData();
formData.append('name', '张三');
formData.append('avatar', fileInput.files[0]);

let response = await fetch('/api/upload', {
  method: 'POST',
  body: formData
  // 不需要设置 Content-Type，浏览器会自动设置为 multipart/form-data
});
```

#### 发送二进制数据（Blob）

javascript 复制代码

```javascript
// 从 canvas 获取图片并上传
let blob = await new Promise(resolve =>
  canvas.toBlob(resolve, 'image/png')
);

let response = await fetch('/api/upload-image', {
  method: 'POST',
  body: blob
  // Blob 有内建类型，会自动设置 Content-Type
});
```

### 3.7 设置请求头

javascript 复制代码

```javascript
let response = await fetch('/api/data', {
  headers: {
    'Authorization': 'Bearer eyJhbGci...',
    'X-Custom-Header': 'hello'
  }
});
```

浏览器出于安全考虑，有些 header 是 **禁止设置** 的（由浏览器独占管理）：  
`Host` 、 `Referer` 、 `Cookie` 、 `Content-Length` 、 `Connection` 、 `Origin` 、 `Sec-*` 、 `Proxy-*` 等。

### 3.8 Request 对象

除了直接传参数，还可以创建 `Request` 对象：

javascript 复制代码

```javascript
let request = new Request('/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: '张三' })
});

// Request 对象可以传给 fetch
let response = await fetch(request);

// 也可以克隆后复用
let request2 = request.clone();
```

这在 Service Worker 中拦截和修改请求时特别有用。

## 第四章 Fetch 进阶

### 4.1 中止请求（AbortController）

Fetch 本身不提供取消机制，而是通过 `AbortController` 来实现：

javascript 复制代码

```javascript
let controller = new AbortController();

// 将 signal 传递给 fetch
let response = fetch('/api/large-data', {
  signal: controller.signal
});

// 在需要时取消请求
controller.abort();

// 取消后，fetch 的 Promise 会 reject，抛出 AbortError
try {
  let data = await response;
} catch (err) {
  if (err.name === 'AbortError') {
    console.log('请求已被取消');
  } else {
    throw err;
  }
}
```

**实际场景：搜索框防抖取消**

javascript 复制代码

```javascript
let currentController = null;

async function search(query) {
  // 取消上一次请求
  if (currentController) {
    currentController.abort();
  }

  currentController = new AbortController();

  try {
    let response = await fetch(\`/api/search?q=${encodeURIComponent(query)}\`, {
      signal: currentController.signal
    });
    let results = await response.json();
    renderResults(results);
  } catch (err) {
    if (err.name !== 'AbortError') {
      console.error('搜索失败:', err);
    }
  }
}

searchInput.addEventListener('input', (e) => {
  search(e.target.value);
});
```

**一个 AbortController 可以同时取消多个请求** ：

javascript 复制代码

```javascript
let controller = new AbortController();

let urls = ['/api/a', '/api/b', '/api/c'];
let requests = urls.map(url =>
  fetch(url, { signal: controller.signal })
);

// 一次取消全部
controller.abort();
```

### 4.2 超时控制

Fetch 没有内建的 timeout 选项，但可以用 `AbortController` 轻松实现：

javascript 复制代码

```javascript
async function fetchWithTimeout(url, options = {}, timeout = 5000) {
  let controller = new AbortController();
  let timer = setTimeout(() => controller.abort(), timeout);

  try {
    let response = await fetch(url, {
      ...options,
      signal: controller.signal
    });
    return response;
  } catch (err) {
    if (err.name === 'AbortError') {
      throw new Error(\`请求超时 (${timeout}ms)\`);
    }
    throw err;
  } finally {
    clearTimeout(timer);
  }
}

// 使用
try {
  let response = await fetchWithTimeout('/api/slow', {}, 3000);
  let data = await response.json();
} catch (err) {
  console.error(err.message); // "请求超时 (3000ms)"
}
```

### 4.3 下载进度跟踪

虽然 `fetch` 没有像 XHR 那样的 `onprogress` 事件，但可以通过 `ReadableStream` 实现下载进度跟踪：

javascript 复制代码

```javascript
async function fetchWithProgress(url, onProgress) {
  let response = await fetch(url);

  // 获取总大小（可能不存在）
  let total = +response.headers.get('Content-Length') || 0;
  let loaded = 0;

  // 从 response.body 获取 ReadableStream
  let reader = response.body.getReader();
  let chunks = [];

  while (true) {
    let { done, value } = await reader.read();
    if (done) break;

    chunks.push(value);
    loaded += value.length;

    if (onProgress) {
      onProgress({ loaded, total });
    }
  }

  // 合并所有块为一个 Uint8Array
  let allChunks = new Uint8Array(loaded);
  let position = 0;
  for (let chunk of chunks) {
    allChunks.set(chunk, position);
    position += chunk.length;
  }

  // 解码为文本
  return new TextDecoder().decode(allChunks);
}

// 使用
let text = await fetchWithProgress('/api/large-file', ({ loaded, total }) => {
  if (total) {
    console.log(\`下载进度: ${Math.round(loaded / total * 100)}%\`);
  } else {
    console.log(\`已下载: ${loaded} 字节\`);
  }
});
```

### 4.4 跨域请求（CORS）

当请求的 URL 与当前页面不同源时（协议、域名或端口不同），就是跨域请求。

javascript 复制代码

```javascript
// 简单跨域请求
let response = await fetch('https://api.other-domain.com/data');

// 携带 Cookie（需要服务器配合）
let response = await fetch('https://api.other-domain.com/data', {
  credentials: 'include'
});
```

`credentials` 选项：

| 值 | 说明 |
| --- | --- |
| `"same-origin"` | 默认值，只对同源请求发送 Cookie |
| `"include"` | 总是发送 Cookie（跨域也发） |
| `"omit"` | 永不发送 Cookie |

> 当使用 `credentials: 'include'` 时，服务器必须返回 `Access-Control-Allow-Credentials: true` ，且 `Access-Control-Allow-Origin` 不能是通配符 `*` 。

### 4.5 Fetch 的完整选项参考

javascript 复制代码

```javascript
fetch(url, {
  method: "GET",               // HTTP 方法
  headers: {},                 // 请求头
  body: undefined,             // 请求体
  mode: "cors",                // "cors" | "same-origin" | "no-cors"
  credentials: "same-origin",  // "same-origin" | "include" | "omit"
  cache: "default",            // 缓存策略
  redirect: "follow",          // "follow" | "manual" | "error"
  referrer: "about:client",    // Referer 头
  referrerPolicy: "no-referrer-when-downgrade",
  integrity: "",               // 子资源完整性校验 hash
  keepalive: false,            // 页面卸载后是否保持请求
  signal: undefined            // AbortSignal，用于取消请求
});
```

#### cache 缓存策略

| 值 | 行为 |
| --- | --- |
| `"default"` | 遵循标准 HTTP 缓存规则 |
| `"no-store"` | 完全绕过缓存 |
| `"reload"` | 不读缓存，但会用响应更新缓存 |
| `"no-cache"` | 先验证缓存是否过期，再决定是否使用 |
| `"force-cache"` | 优先使用缓存，即使已过期 |
| `"only-if-cached"` | 只用缓存，没有则报错（需 `mode: "same-origin"` ） |

#### keepalive

javascript 复制代码

```javascript
// 用于页面关闭前发送统计数据
window.addEventListener('unload', () => {
  fetch('/api/analytics', {
    method: 'POST',
    body: JSON.stringify(analyticsData),
    keepalive: true  // 即使页面关闭也会完成请求
  });
});
```

> `keepalive` 请求的 body 总大小限制为 64KB。

### 4.6 错误处理最佳实践

`fetch` 只有在 **网络层面失败** 时才会 reject（如 DNS 错误、网络断开）。HTTP 错误（如 404、500） **不会** 导致 reject，需要手动检查：

javascript 复制代码

```javascript
async function safeFetch(url, options) {
  let response;

  try {
    response = await fetch(url, options);
  } catch (err) {
    // 网络错误或请求被取消
    if (err.name === 'AbortError') {
      throw new Error('请求已取消');
    }
    throw new Error('网络错误: ' + err.message);
  }

  // HTTP 错误
  if (!response.ok) {
    let errorBody;
    try {
      errorBody = await response.json();
    } catch {
      errorBody = await response.text();
    }
    throw new Error(\`HTTP ${response.status}: ${JSON.stringify(errorBody)}\`);
  }

  return response;
}

// 使用
try {
  let response = await safeFetch('/api/users');
  let users = await response.json();
} catch (err) {
  console.error(err.message);
}
```

## 第五章 Server-Sent Events：服务器单向推送

### 5.1 什么是 SSE

**Server-Sent Events（SSE）** 是一种允许服务器通过 HTTP 持久连接，向客户端单向推送数据的技术。

浏览器提供了内建的 `EventSource` 类来使用 SSE。

#### SSE vs WebSocket

| 特性 | SSE（EventSource） | WebSocket |
| --- | --- | --- |
| 通信方向 | 单向：服务器 → 客户端 | 双向：服务器 ⇄ 客户端 |
| 数据类型 | 仅文本 | 文本和二进制 |
| 协议 | HTTP | WebSocket（ws/wss） |
| 自动重连 | ✅ 内建支持 | ❌ 需要手动实现 |
| 消息 ID / 断线续传 | ✅ 内建支持 | ❌ 需要手动实现 |
| 浏览器支持 | 除 IE 外全部支持 | 全部现代浏览器 |

**何时选择 SSE** ：

- 服务器→客户端的 **单向推送** 场景
- 新闻/股票/通知的实时更新
- AI 聊天流式输出（如 ChatGPT 的逐字输出效果）
- 相比 WebSocket 更简单，不需要额外的服务器库

### 5.2 客户端基本用法

javascript 复制代码

```javascript
// 创建 EventSource 连接
let eventSource = new EventSource('/api/events');

// 接收消息
eventSource.onmessage = function(event) {
  console.log('收到消息:', event.data);
};

// 连接建立
eventSource.onopen = function(event) {
  console.log('连接已建立');
};

// 发生错误（包括连接断开后自动重连）
eventSource.onerror = function(event) {
  if (eventSource.readyState === EventSource.CONNECTING) {
    console.log('连接断开，正在重连...');
  } else {
    console.error('连接错误');
  }
};

// 关闭连接
eventSource.close();
```

### 5.3 连接状态

javascript 复制代码

```javascript
EventSource.CONNECTING = 0; // 连接中 或 重连中
EventSource.OPEN       = 1; // 已连接
EventSource.CLOSED     = 2; // 连接已关闭
```

### 5.4 服务器端格式

服务器必须以 `Content-Type: text/event-stream` 响应，并保持连接打开。

#### 基本消息格式

每条消息以 **两个换行符** `\n\n` 分隔：

复制代码

```
data: 这是第一条消息\n
\n
data: 这是第二条消息\n
\n
data: 第三条消息第一行\n
data: 第三条消息第二行\n
\n
```
- `data:` 后是消息内容，冒号后的空格可选
- 多个 `data:` 行会被合并为一条消息，中间用 `\n` 连接

#### 消息 ID（用于断线续传）

复制代码

```
data: 消息内容\n
id: 1\n
\n
data: 另一条消息\n
id: 2\n
\n
```

当连接断开并自动重连时，浏览器会在请求头中发送 `Last-Event-ID` ，服务器可以据此重发遗漏的消息。

客户端可通过 `eventSource.lastEventId` 获取最后收到的 ID。

#### 自定义重连间隔

复制代码

```
retry: 5000\n
data: 服务器设置了 5 秒重连间隔\n
\n
```

#### 自定义事件类型

复制代码

```
event: notification\n
data: {"title": "新消息", "body": "你有一封新邮件"}\n
\n

event: heartbeat\n
data: ping\n
\n

data: 这是默认的 message 事件\n
\n
```

### 5.5 处理自定义事件

使用 `event:` 字段的消息 **不会** 触发 `onmessage` ，必须用 `addEventListener` 监听：

javascript 复制代码

```javascript
let eventSource = new EventSource('/api/events');

// 监听自定义的 notification 事件
eventSource.addEventListener('notification', function(event) {
  let data = JSON.parse(event.data);
  showNotification(data.title, data.body);
});

// 监听自定义的 heartbeat 事件
eventSource.addEventListener('heartbeat', function(event) {
  console.log('心跳:', event.data);
});

// 监听默认的 message 事件（没有 event: 字段的消息）
eventSource.onmessage = function(event) {
  console.log('普通消息:', event.data);
};
```

### 5.6 跨域 SSE

javascript 复制代码

```javascript
let source = new EventSource('https://other-domain.com/events', {
  withCredentials: true  // 发送 Cookie
});
```

服务器需要返回相应的 CORS header。

### 5.7 服务器端实现（Node.js）

javascript 复制代码

```javascript
const http = require('http');

http.createServer((req, res) => {
  if (req.url === '/events') {
    // 设置 SSE 必需的响应头
    res.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
      'Access-Control-Allow-Origin': '*'
    });

    let id = 0;

    // 每秒发送一条消息
    let timer = setInterval(() => {
      id++;
      let data = JSON.stringify({
        time: new Date().toISOString(),
        value: Math.random()
      });

      res.write(\`id: ${id}\n\`);
      res.write(\`data: ${data}\n\n\`);
    }, 1000);

    // 客户端断开时清理
    req.on('close', () => {
      clearInterval(timer);
      console.log('客户端断开');
    });

  } else {
    res.writeHead(404);
    res.end('Not Found');
  }
}).listen(3000);
```

### 5.8 实战：AI 流式输出

SSE 非常适合实现类似 ChatGPT 的流式文字输出效果：

javascript 复制代码

```javascript
// 客户端
async function streamChat(prompt) {
  let eventSource = new EventSource(
    \`/api/chat?prompt=${encodeURIComponent(prompt)}\`
  );

  let outputDiv = document.getElementById('output');
  outputDiv.textContent = '';

  eventSource.onmessage = function(event) {
    if (event.data === '[DONE]') {
      eventSource.close();
      return;
    }

    let data = JSON.parse(event.data);
    outputDiv.textContent += data.content;
  };

  eventSource.onerror = function() {
    eventSource.close();
  };
}
```

> **提示** ：很多 AI API（如 OpenAI）的流式响应就是基于 SSE 协议的。在需要向服务器发送 POST body 的场景中，可以用 `fetch` + `ReadableStream` 来手动解析 SSE 流，因为 `EventSource` 只支持 GET 请求。

#### 用 Fetch 读取 SSE 流（支持 POST）

javascript 复制代码

```javascript
async function fetchSSE(url, body) {
  let response = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body)
  });

  let reader = response.body.getReader();
  let decoder = new TextDecoder();
  let buffer = '';

  while (true) {
    let { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });

    // 按双换行分割消息
    let messages = buffer.split('\n\n');
    buffer = messages.pop(); // 最后一个可能不完整

    for (let msg of messages) {
      let lines = msg.split('\n');
      for (let line of lines) {
        if (line.startsWith('data: ')) {
          let data = line.slice(6);
          if (data === '[DONE]') return;
          console.log('收到:', JSON.parse(data));
        }
      }
    }
  }
}
```

## 第六章 WebSocket：全双工实时通信

### 6.1 什么是 WebSocket

WebSocket 是一种在浏览器和服务器之间建立 **持久的全双工连接** 的协议（ [RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455) ）。数据可以在两个方向上实时传递，无需重新发起请求。

复制代码

```
传统 HTTP：
客户端 ──请求──> 服务器
客户端 <──响应── 服务器
（每次通信都需要新请求）

WebSocket：
客户端 ⇄ 服务器
（建立连接后，双方可随时发送数据）
```

**典型场景** ：

- 即时聊天
- 多人在线游戏
- 实时协同编辑
- 实时交易系统
- 物联网数据推送

### 6.2 建立连接

javascript 复制代码

```javascript
// 使用 ws:// 或加密的 wss:// 协议
let socket = new WebSocket("wss://example.com/chat");
```

> **始终优先使用 `wss://`** （基于 TLS 加密），原因：
> 
> 1. 数据安全，传输过程中不会被中间人窃听
> 2. 某些代理服务器可能会中断 `ws://` 连接（因为不认识非标准 header），而 `wss://` 的加密流量能顺利通过

### 6.3 四大事件

javascript 复制代码

```javascript
let socket = new WebSocket("wss://example.com/chat");

// 1. 连接建立
socket.onopen = function(event) {
  console.log('连接已建立');
  socket.send('你好，服务器！');
};

// 2. 接收消息
socket.onmessage = function(event) {
  console.log('收到消息:', event.data);
};

// 3. 连接关闭
socket.onclose = function(event) {
  if (event.wasClean) {
    console.log(\`连接正常关闭, code=${event.code}, reason=${event.reason}\`);
  } else {
    console.log('连接异常断开');
  }
};

// 4. 发生错误
socket.onerror = function(error) {
  console.error('WebSocket 错误:', error.message);
};
```

### 6.4 连接握手过程

WebSocket 连接始于一个 HTTP "升级"请求：

**客户端请求：**

http 复制代码

```http
GET /chat HTTP/1.1
Host: example.com
Connection: Upgrade
Upgrade: websocket
Origin: https://example.com
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

**服务器响应：**

http 复制代码

```http
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

握手成功后，通信协议从 HTTP 切换到 WebSocket，后续数据以 WebSocket "帧（frame）"传输。

### 6.5 发送数据

javascript 复制代码

```javascript
// 发送文本
socket.send('Hello!');

// 发送 JSON
socket.send(JSON.stringify({
  type: 'message',
  content: '你好',
  timestamp: Date.now()
}));

// 发送二进制数据（Blob）
let blob = new Blob(['binary data'], { type: 'application/octet-stream' });
socket.send(blob);

// 发送 ArrayBuffer
let buffer = new ArrayBuffer(8);
socket.send(buffer);
```

### 6.6 接收数据

文本数据直接以字符串形式接收。二进制数据的格式由 `binaryType` 属性决定：

javascript 复制代码

```javascript
// 默认为 "blob"
socket.binaryType = "blob";

// 改为 "arraybuffer"
socket.binaryType = "arraybuffer";

socket.onmessage = function(event) {
  if (typeof event.data === 'string') {
    // 文本消息
    let msg = JSON.parse(event.data);
    console.log(msg);
  } else {
    // 二进制消息（Blob 或 ArrayBuffer，取决于 binaryType）
    console.log('收到二进制数据:', event.data.byteLength, '字节');
  }
};
```

### 6.7 流量控制（bufferedAmount）

当网速较慢时，发送的数据会在内存中缓冲。通过 `socket.bufferedAmount` 可以检查缓冲区的大小：

javascript 复制代码

```javascript
setInterval(() => {
  if (socket.bufferedAmount === 0) {
    socket.send(getNextData());
  }
}, 100);
```

### 6.8 关闭连接

javascript 复制代码

```javascript
// 正常关闭
socket.close(1000, '操作完成');

// 关闭事件中获取信息
socket.onclose = function(event) {
  console.log('关闭码:', event.code);
  console.log('关闭原因:', event.reason);
  console.log('是否正常关闭:', event.wasClean);
};
```

**常见关闭码** ：

| 码 | 含义 |
| --- | --- |
| `1000` | 正常关闭（默认） |
| `1001` | 一方正在离开（页面关闭、服务器关闭等） |
| `1006` | 连接异常断开（无 close frame） |
| `1009` | 消息太大 |
| `1011` | 服务器内部错误 |

### 6.9 连接状态

javascript 复制代码

```javascript
socket.readyState;

// 0 - CONNECTING  连接建立中
// 1 - OPEN        已连接，可以通信
// 2 - CLOSING     正在关闭
// 3 - CLOSED      已关闭
```

### 6.10 子协议

WebSocket 支持通过子协议来约定数据格式：

javascript 复制代码

```javascript
let socket = new WebSocket("wss://example.com/chat", ["json", "xml"]);

// 连接建立后，可查看服务器选择的子协议
socket.onopen = function() {
  console.log('使用的子协议:', socket.protocol); // 例如 "json"
};
```

### 6.11 自动重连实现

WebSocket 没有内建的重连机制（这点不如 SSE），需要手动实现：

javascript 复制代码

```javascript
function createWebSocket(url) {
  let socket;
  let retryCount = 0;
  let maxRetries = 10;
  let baseDelay = 1000;

  function connect() {
    socket = new WebSocket(url);

    socket.onopen = function() {
      console.log('连接成功');
      retryCount = 0; // 重置重试计数
    };

    socket.onmessage = function(event) {
      handleMessage(event.data);
    };

    socket.onclose = function(event) {
      if (!event.wasClean && retryCount < maxRetries) {
        // 指数退避重连
        let delay = baseDelay * Math.pow(2, retryCount);
        delay = Math.min(delay, 30000); // 最多 30 秒
        console.log(\`${delay / 1000}秒后重连... (第${retryCount + 1}次)\`);
        setTimeout(connect, delay);
        retryCount++;
      }
    };

    socket.onerror = function(error) {
      console.error('WebSocket 错误:', error);
    };
  }

  function send(data) {
    if (socket.readyState === WebSocket.OPEN) {
      socket.send(data);
    } else {
      console.warn('连接未就绪，消息未发送');
    }
  }

  function close() {
    retryCount = maxRetries; // 阻止重连
    socket.close(1000, '用户主动关闭');
  }

  connect();

  return { send, close };
}

// 使用
let ws = createWebSocket('wss://example.com/chat');
ws.send(JSON.stringify({ type: 'hello' }));
```

### 6.12 心跳保活

长时间没有数据传输时，某些代理服务器或防火墙会关闭空闲连接。解决方案是定时发送心跳包：

javascript 复制代码

```javascript
function setupHeartbeat(socket, interval = 30000) {
  let heartbeatTimer;

  socket.onopen = function() {
    heartbeatTimer = setInterval(() => {
      if (socket.readyState === WebSocket.OPEN) {
        socket.send(JSON.stringify({ type: 'ping' }));
      }
    }, interval);
  };

  socket.onclose = function() {
    clearInterval(heartbeatTimer);
  };
}
```

> WebSocket 协议本身有 ping/pong 帧（由浏览器自动处理），但应用层的心跳可以确保更好的可靠性。

### 6.13 服务器端实现（Node.js）

使用 [ws](https://github.com/websockets/ws) 库：

javascript 复制代码

```javascript
const WebSocket = require('ws');
const http = require('http');

const server = http.createServer();
const wss = new WebSocket.Server({ server });

// 在线用户集合
const clients = new Set();

wss.on('connection', (ws, req) => {
  clients.add(ws);
  console.log(\`新连接，当前在线: ${clients.size}\`);

  // 接收消息
  ws.on('message', (message) => {
    let data = JSON.parse(message);

    // 广播给所有在线用户
    for (let client of clients) {
      if (client !== ws && client.readyState === WebSocket.OPEN) {
        client.send(JSON.stringify({
          type: 'message',
          from: data.from,
          content: data.content,
          time: new Date().toISOString()
        }));
      }
    }
  });

  // 连接关闭
  ws.on('close', () => {
    clients.delete(ws);
    console.log(\`连接断开，当前在线: ${clients.size}\`);
  });

  // 错误处理
  ws.on('error', (error) => {
    console.error('WebSocket 错误:', error);
  });
});

server.listen(3000, () => {
  console.log('服务器运行在 ws://localhost:3000');
});
```

### 6.14 实战：简单聊天室

**客户端 HTML + JavaScript：**

html 复制代码

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>WebSocket 聊天室</title>
  <style>
    #messages {
      border: 1px solid #ccc;
      height: 300px;
      overflow-y: auto;
      padding: 10px;
      margin-bottom: 10px;
    }
    .msg { margin: 4px 0; }
    .msg-self { color: blue; }
    .msg-other { color: green; }
    .msg-system { color: gray; font-style: italic; }
  </style>
</head>
<body>
  <div id="messages"></div>
  <form id="form">
    <input id="input" type="text" placeholder="输入消息..." autocomplete="off" />
    <button type="submit">发送</button>
  </form>

  <script>
    let socket = new WebSocket('wss://example.com/chat');
    let messagesDiv = document.getElementById('messages');
    let form = document.getElementById('form');
    let input = document.getElementById('input');

    function appendMessage(text, className) {
      let div = document.createElement('div');
      div.className = 'msg ' + className;
      div.textContent = text;
      messagesDiv.append(div);
      messagesDiv.scrollTop = messagesDiv.scrollHeight;
    }

    socket.onopen = () => appendMessage('已连接到聊天室', 'msg-system');

    socket.onmessage = (event) => {
      let data = JSON.parse(event.data);
      appendMessage(\`${data.from}: ${data.content}\`, 'msg-other');
    };

    socket.onclose = (event) => {
      appendMessage(
        event.wasClean ? '已断开连接' : '连接异常断开',
        'msg-system'
      );
    };

    form.onsubmit = (e) => {
      e.preventDefault();
      let text = input.value.trim();
      if (text && socket.readyState === WebSocket.OPEN) {
        socket.send(JSON.stringify({
          from: '我',
          content: text
        }));
        appendMessage(\`我: ${text}\`, 'msg-self');
        input.value = '';
      }
    };
  </script>
</body>
</html>
```

## 第七章 实战对比与最佳实践

### 7.1 四种技术全面对比

| 特性 | XMLHttpRequest | Fetch API | SSE (EventSource) | WebSocket |
| --- | --- | --- | --- | --- |
| 通信模式 | 请求-响应 | 请求-响应 | 服务器→客户端 | 全双工 |
| 协议 | HTTP | HTTP | HTTP | WS/WSS |
| API 风格 | 回调/事件 | Promise | 事件 | 事件 |
| 数据格式 | 任意 | 任意 | 文本 | 文本+二进制 |
| 自动重连 | ❌ | ❌ | ✅ | ❌（需手动） |
| 上传进度 | ✅ | ❌ | N/A | N/A |
| 下载进度 | ✅ | ✅ (Stream) | N/A | N/A |
| 取消请求 | ✅ abort() | ✅ AbortController | ✅ close() | ✅ close() |
| Service Worker | ❌ | ✅ | ❌ | ❌ |
| 跨域 | CORS | CORS | CORS | Origin 校验 |
| 浏览器兼容 | IE7+ | 现代浏览器 | 除 IE 外 | 现代浏览器 |

### 7.2 选型决策树

复制代码

```
你的需求是什么？
│
├─ 普通的数据请求（获取/提交数据）
│  ├─ 新项目 ──────────────> Fetch API ✅
│  └─ 需要上传进度跟踪 ────> XMLHttpRequest ✅
│
├─ 服务器单向推送数据
│  ├─ 只需要接收推送 ──────> SSE (EventSource) ✅
│  └─ 需要 POST 请求 ──────> Fetch + ReadableStream ✅
│
├─ 双向实时通信
│  └─ 聊天/游戏/协作 ──────> WebSocket ✅
│
└─ 简单的定时更新
   └─ 更新频率 > 30秒 ─────> 轮询 (Fetch/XHR) ✅
```

### 7.3 从 XHR 迁移到 Fetch

以下是常见 XHR 用法对应的 Fetch 写法：

#### GET 请求

javascript 复制代码

```javascript
// XHR 写法
let xhr = new XMLHttpRequest();
xhr.open('GET', '/api/users');
xhr.responseType = 'json';
xhr.onload = () => console.log(xhr.response);
xhr.onerror = () => console.error('失败');
xhr.send();

// Fetch 写法
let response = await fetch('/api/users');
let data = await response.json();
console.log(data);
```

#### POST JSON

javascript 复制代码

```javascript
// XHR 写法
let xhr = new XMLHttpRequest();
xhr.open('POST', '/api/users');
xhr.setRequestHeader('Content-Type', 'application/json');
xhr.onload = () => console.log(xhr.response);
xhr.send(JSON.stringify({ name: '张三' }));

// Fetch 写法
let response = await fetch('/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: '张三' })
});
let data = await response.json();
```

#### 带超时的请求

javascript 复制代码

```javascript
// XHR 写法
let xhr = new XMLHttpRequest();
xhr.open('GET', '/api/data');
xhr.timeout = 5000;
xhr.ontimeout = () => console.error('超时');
xhr.send();

// Fetch 写法
let controller = new AbortController();
setTimeout(() => controller.abort(), 5000);

try {
  let response = await fetch('/api/data', { signal: controller.signal });
  let data = await response.json();
} catch (err) {
  if (err.name === 'AbortError') console.error('超时');
}
```

### 7.4 通信方案对比示例

假设需要实现"实时股票行情"功能，以下是不同方案的对比：

#### 方案一：轮询（Fetch）

javascript 复制代码

```javascript
async function pollStockPrice() {
  while (true) {
    try {
      let response = await fetch('/api/stock/AAPL');
      let data = await response.json();
      updateUI(data.price);
    } catch (err) {
      console.error(err);
    }
    await new Promise(r => setTimeout(r, 1000)); // 每秒查一次
  }
}
```
- 优点：实现简单
- 缺点：大量无效请求，延迟高（最多 1 秒），服务器压力大

#### 方案二：SSE

javascript 复制代码

```javascript
let source = new EventSource('/api/stock/stream?symbol=AAPL');
source.onmessage = (event) => {
  let data = JSON.parse(event.data);
  updateUI(data.price);
};
```
- 优点：服务器有更新才推送，实时性好，自动重连
- 缺点：单向通信，每个股票可能需要一个连接

#### 方案三：WebSocket

javascript 复制代码

```javascript
let socket = new WebSocket('wss://example.com/stock');
socket.onopen = () => {
  socket.send(JSON.stringify({ subscribe: ['AAPL', 'GOOGL', 'MSFT'] }));
};
socket.onmessage = (event) => {
  let data = JSON.parse(event.data);
  updateUI(data.symbol, data.price);
};
```
- 优点：双向通信，可动态订阅/退订，一个连接处理多个股票
- 缺点：服务器端实现复杂，需要处理重连

### 7.5 安全最佳实践

1. **始终使用加密协议**
	- `https://` 而非 `http://`
		- `wss://` 而非 `ws://`
2. **验证 Origin** （WebSocket 服务器端）
	javascript 复制代码
	```javascript
	wss.on('connection', (ws, req) => {
	  let origin = req.headers.origin;
	  if (origin !== 'https://your-site.com') {
	    ws.close(1008, 'Origin 不允许');
	    return;
	  }
	});
	```
3. **防止 CSRF**
	- Fetch/XHR：使用 CSRF Token
		- WebSocket：验证 Origin + 认证 Token
4. **输入验证**
	- 服务器端始终验证接收到的数据
		- 客户端不信任任何来自服务器的数据
5. **速率限制**
	- 服务器端限制消息频率，防止恶意客户端

### 7.6 性能优化建议

1. **减少请求数量**
	- 合并多个小请求为一个批量请求
		- 使用 WebSocket/SSE 替代频繁轮询
2. **合理使用缓存**
	javascript 复制代码
	```javascript
	fetch('/api/static-data', { cache: 'force-cache' });
	```
3. **压缩传输数据**
	- 服务器开启 gzip/brotli 压缩
		- WebSocket 可使用 `permessage-deflate` 扩展
4. **连接复用**
	- HTTP/2 自动多路复用
		- WebSocket 一个连接处理多种消息
5. **页面卸载时的数据发送**
	javascript 复制代码
	```javascript
	// 优先使用 sendBeacon（专为此场景设计）
	navigator.sendBeacon('/api/analytics', data);
	// 或 fetch + keepalive
	fetch('/api/analytics', {
	  method: 'POST',
	  body: data,
	  keepalive: true
	});
	```

> **参考资料**
> 
> - [现代 JavaScript 教程 - 网络请求](https://zh.javascript.info/network)
> - [MDN - XMLHttpRequest](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest)
> - [MDN - Fetch API](https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API)
> - [MDN - EventSource](https://developer.mozilla.org/zh-CN/docs/Web/API/EventSource)
> - [MDN - WebSocket](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket)
> - [RFC 6455 - WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)
> - [HTML Living Standard - Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html)