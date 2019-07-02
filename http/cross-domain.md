## 跨域问题总结

### 跨域原因
跨域(CORS: cross-origin resource sharing)源于浏览器的同源策略：同源策略控制了不同源之间的交互

当浏览器使用js脚本在网址http(s)://a.com/xxx访问域名b.com的内容的时候，处于安全考虑，会先发送Options请求询问b.com是否允许跨域名访问，如果不被允许，则调用会出错，js不清楚出错具体原因，可通过控制台查看

如果没有跨域，假设我在我的网站里插入一段访问银行api的脚本，浏览网站的用户会有银行账户被盗的风险(CSRF攻击)

### 添加跨域头

浏览器发出跨域请求时，会先发出OPTIONS请求，查询目的地的跨域策略，目的地允许跨域时会会返回一下header：

1. Access-Control-Allow-Origin: http://foo.example  
	允许哪些网站跨域，为*时表示任意网址
2. Access-Control-Allow-Methods: POST, GET, OPTIONS  
	允许哪些方法
3. Access-Control-Allow-Headers: X-PINGOTHER  
	允许哪些header

### 简单请求与非简单请求
使用CORS时，请求会被浏览器分为简单请求和非简单请求。  
有以下两个特点的请求归于简单请求： 

请求方法是以下三种之一： 
<pre>
HEAD
GET
POST
</pre>

HTTP 的头信息不超出以下几种字段：
<pre>
Accept
Accept-Language
Content-Language
Last-Event-ID
Content-type:只限于3个值application/x-www-from-urlencoded、multipart/from-data、text/plain
</pre>
不满足以上任何条件时，均归为非简单请求  

对于简单请求，浏览器直接发出CORS请求，请求头中添加Origin字段指明本次请求的源，服务器根据这个值，决定是否同意这次请求。如果Origin指定的源，不在许可范围内，服务器返回一个正常的HTTP回应。头信息中没有包含Access-Control-Allow-Origin字段，便会抛出错误：被XMLHttpRequest的onerror回调函数捕获。

非简单请求是那种对服务器有特殊要求的请求，比如请求方法是PUT或DELETE，或者Content-Type字段的类型是application/json。
非简单请求之前，会增加一次HTTP查询请求，称为“预检”请求。
浏览器先询问服务器，当前网页所在域名是否在服务器许可名单中，以及可以使用哪些HTTP动词和头信息字段，只有得到肯定的答复，浏览器才会发出XMLHttpRequest请求。"预检"请求方法是OPTIONS，表示这个请求是用来询问的，头信息里边的关键字Origin表示来源。此外还有：  
1. Access-Control-Request-Method： 该字段是必须的，用来列出浏览器的CORS会用到哪些HTTP方法。
2. Access-Control-Request-Headers： 指定浏览器CORS请求会额外发送的头信息字段
"预检"请求的回应:  
1. Access-Control-Allow-Methods： 返回所有支持的方法：避免多次"预检"请求；  
2. Access-Control-Allow-Headers： 如果浏览器有Access-Control-Request-Headers请求，则为必须；  
3. Access-Control-Allow-Credentials： 该字段可选，是一个布尔值，表示是否发送Cookie,默认为false；  
4. Access-Control-Max-Age： 可选，指定本次预检请求的有效期；

### 使用jsonp绕开跨域

通常为了减轻web服务器的负载，我们把js、css，img等静态资源分离到另一台独立域名的服务器上，在html页面中再通过相应的标签从不同域名下加载静态资源，而被浏览器允许，基于此原理，我们可以通过动态创建script，再请求一个带参网址实现跨域通信。
这种方法只能进行get请求