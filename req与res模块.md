# req与res模块

这个模块其实比较容易理解，基本可以将其与源码分离。

首先明白其目的：为了给用户提供请求与返回的各种方法。

在express中，request与response的各种方法都是封装在request.js与response.js文件中的，而这两个文件中原始版本是nodejs提供给框架的。

```javascript
var req = Object.create(http.IncomingMessage.prototype);
```

```javascript
var res = Object.create(http.ServerResponse.prototype);
```

#### 基本步骤

1. 首先在express中继承req与res，req、res来自request.js与response.js文件

~~~  
  //使request继承req，并且可配置，可枚举、可写
  app.request = Object.create(req, {

​    app: { configurable: true, enumerable: true, writable: true, value: app }

  })



  // expose the prototype that will get set on responses

   //使response继承res，并且可配置，可枚举、可写

  app.response = Object.create(res, {

​    app: { configurable: true, enumerable: true, writable: true, value: app }

  })
~~~

2. 在request.js与response.js文件中封装需要的各种方法。在express中是var req和res继承nodejs的对象原型，然后将新的方法加在了res与req的对象上。例如：

   ~~~ javascript
   res.json = function json(obj) {
     var val = obj;
   
     // allow status / body
     if (arguments.length === 2) {
       // res.json(body, status) backwards compat
       if (typeof arguments[1] === 'number') {
         deprecate('res.json(obj, status): Use res.status(status).json(obj) instead');
         this.statusCode = arguments[1];
       } else {
         deprecate('res.json(status, obj): Use res.status(status).json(obj) instead');
         this.statusCode = arguments[0];
         val = arguments[1];
       }
     }
   
     // settings
     var app = this.app;
     var escape = app.get('json escape')
     var replacer = app.get('json replacer');
     var spaces = app.get('json spaces');
     var body = stringify(val, replacer, spaces, escape)
   
     // content-type
     if (!this.get('Content-Type')) {
       this.set('Content-Type', 'application/json');
     }
   
     return this.send(body);
   };
   ~~~

