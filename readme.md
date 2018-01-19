# unfetch

> 超小的 `fetch` polyfill in 500 字节. 

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)
    
Explanation

> "version": "3.0.0"

[github source](https://github.com/developit/unfetch)

~~[english](./README.en.md)~~

---

好了，这么小，只有一个主文件

[src/index.js](./unfetch/src/index.js)

不如先看看例子

[jsfiddle-unfetch-例子](https://jsfiddle.net/developit/qrh7tLc0/)
---

- `ponyfill vs polyfill`

- [index-XMLHttpRequest的设置](#index)

- [response-数据的处理](#response)

---

## ponyfill vs polyfill

作为 [ponyfill](https://ponyfill.com/): 只使用`unfetch`

``` js
import fetch from 'unfetch';

fetch('/foo.json')
  .then( r => r.json() )
  .then( data => {
    console.log(data);
  });

```

全局, 作为 [polyfill](https://ponyfill.com/#polyfill): 如果有原生，用原生，没有则用`unfetch`

``` js
import 'unfetch/polyfill';

// "fetch" 已经加载了，如果不存在的话

fetch('/foo.json')
  .then( r => r.json() )
  .then( data => {
    console.log(data);
  });
```

> `要`，还是`不要`《-🌟-》原生api，这是一个问题

---

例子-jsfiddle

``` js
unfetch('//jsonplaceholder.typicode.com/posts', {
	method: 'POST',
	headers: { 'Content-Type': 'application/json' },
	body: JSON.stringify({
		title: 'foo',
		body: 'bar',
		userId: 1
	})
})
	.then( r => r.json() )
	.then( data => {
		log(data);
		
		unfetch('//jsonplaceholder.typicode.com/posts/1')
			.then( r => r.json() )
			.then( log )
	});
```

## index

``` js
export default typeof fetch=='function' ? fetch.bind()      // fetch.bind() ,使用原生
    : function(url, options) {
        
	options = options || {};
	return new Promise( (resolve, reject) => { // 增加 Promise --> async/await
		let request = new XMLHttpRequest();

		request.open(options.method || 'get', url, true);

		for (let i in options.headers) {
            // headers: { 'Content-Type': 'application/json' },
			request.setRequestHeader(i, options.headers[i]); // 设置请求-头 ---- 1
		}

		request.withCredentials = options.credentials=='include'; // ----- 2

		request.onload = () => { //        --- 3
			resolve(response()); // 获取-函数触发
		};

		request.onerror = reject; //   --- 4

        request.send(options.body);  // --- 5
    //     	body: JSON.stringify({
        // 	title: 'foo',
        // 	body: 'bar',
        // 	userId: 1
        // })

//  ---- 获取数据后，函数运行
		function response() {
            // ... 
        }
	});
}

```

1. `request.setRequestHeader`

> 给指定的HTTP请求头赋值.

2. `request.withCredentials`

> 表明在进行跨站(cross-site)的访问控制(Access-Control)请求时，是否使用认证信息(例如cookie或授权的header)。 默认为 false。

3. `request.onload`

> 当一个资源及其依赖资源已完成加载时，将触发load事件。[mdn文档](https://developer.mozilla.org/zh-CN/docs/Web/Events/load)

4. `request.onerror`

> 当一个资源加载失败时会触发error事件。[mdn文档](https://developer.mozilla.org/zh-CN/docs/Web/Events/error)

5. `request.send`

> 发送请求. 如果该请求是异步模式(默认),该方法会立刻返回. 相反,如果请求是同步模式,则直到请求的响应完全接受以后,该方法才会返回.

[有关-XMLHttpRequest-更多](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest)

---

## response

``` js
		request.onload = () => { //        --- 3
			resolve(response()); // 获取-函数触发
		};

```

数据获取后，`response` 触发

``` js
		function response() {
			let keys = [],
				all = [],
				headers = {},
				header;

			request.getAllResponseHeaders().replace(/^(.*?):\s*?([\s\S]*?)$/gm, (m, key, value) => {
				keys.push(key = key.toLowerCase());
				all.push([key, value]);
				header = headers[key];
				headers[key] = header ? `${header},${value}` : value;
			});

			return {
				ok: (request.status/100|0) == 2,		// 200-299
				status: request.status,
				statusText: request.statusText,
				url: request.responseURL,
				clone: response,
				text: () => Promise.resolve(request.responseText),
				json: () => Promise.resolve(request.responseText).then(JSON.parse),
				blob: () => Promise.resolve(new Blob([request.response])),
				headers: {
					keys: () => keys,
					entries: () => all,
					get: n => headers[n.toLowerCase()],
					has: n => n.toLowerCase() in headers
				}
			};
		}
```

- 