# express源码学习

## 基本知识

#### 目录结构

1. 目录结构 

   middleware/init.is:主要作用是初始化request、reponse

   middleware/query.js:主要作用是格式化url，将url中的参数剥离，储存到req.query中。

   router/indes.js/route.js/layer.js:主要负责中间件的插入和链式执行。

   application.js:框架的主要文件，很多定义了很多重要的api。

   express.js：主要入口文件，封装了包括application在内的所有的属性，并将其暴露给外部调用。

   request.js/reponse.js:两者提供了一些方法丰富request和response实例的功能，如req.is、req.get、req.params、req.originalUrl等。

   utils.js:工具函数。

   view.js:模板渲染相关。

2. express是主入口，在内部创建了一个方法：createApplication，它既是一个方法，同时也继承了来自application等其它模块的各种属性，并暴露出去。其中最重要的模块是application。详细可见下方的代码及备注。

   ~~~ javascript
   function createApplication() {
     var app = function(req, res, next) {// 持续监听请求
       app.handle(req, res, next);//处理路由
     };
   
     //当参数为false的时候，则原对象app的已有的属性不继承后面对象的属性。
     mixin(app, EventEmitter.prototype, false);//继承用于事件监听的EventEmitter.prototype
     mixin(app, proto, false);//继承来自application的属性
   
     // expose the prototype that will get set on requests
     //使request继承req，并且可配置，可枚举、可写
     app.request = Object.create(req, {
       app: { configurable: true, enumerable: true, writable: true, value: app }
     })
   
     // expose the prototype that will get set on responses
      //使response继承res，并且可配置，可枚举、可写
     app.response = Object.create(res, {
       app: { configurable: true, enumerable: true, writable: true, value: app }
     })
     app.init();
     return app;
   } 
   ~~~

3. 当在使用express的时候，

   a、实例化express，var app = express();

   b、为之赋予更多的属性，如app.use('/', indexRouter);app.use('/users', usersRouter);等。

#### 添加中间件

当为app添加属性的时候，不可避免的需要用到express的中间件的功能。

1. 在理解express的源码的时候，首先区别express中间件的概念。其中最重要的两个就是非路由中间件和路由中间件。

   ~~~ JavaScript
   //非路由中间件：
   app.use('/test',function(req,res,next) {
     console.log('Time:', Date.now())
     next()
   })
   //路由中间件：
   app.get('/test',function(req,res,next){ 
   	console.log("app.get('/test') handler1");
   	next();
   })
   ~~~

2. 区分了中间件，现面分两条线理解express源码的路由概念。

   1. app.use:主要用来添加非路由中间件。
      在application.js中，首先将参数扁平化为数组，然后逐个调用router.use。

      ~~~ JavaScript
      app.use = function use(fn) {
        var offset = 0;//该变量用来在arguments中定位handler的起始位置，在没有传入path的时候，handler是arguments的第一个元素，所以为0
        var path = '/';
      
        // default path to '/'
        // disambiguate app.use([fn])
      
        //判断app.use传进来的是否是函数
        if (typeof fn !== 'function') {
          var arg = fn;
      
          while (Array.isArray(arg) && arg.length !== 0) {//如果第一个参数是数组的话，取出数组第一个元素
            arg = arg[0];
          }
      
          // first arg is the path
          //第一个参数不是function
          //取出第一个参数，将第一个参数赋值给path。
          if (typeof arg !== 'function') {
            offset = 1;
            path = fn;
          }
        }
      
        //slice.call(arguments,offset),通过slice转为数据，slice可以改变具有length的类数组。flatten:扁平化数组
        //arguments是一个类数组对象。
         //从参数中取出处理函数列表
      
        //处理多种中间件使用方式。
        // app.use(r1, r2);
        // app.use('/', [r1, r2]);
        // app.use(mw1, [mw2, r1, r2], subApp);
        var fns = flatten(slice.call(arguments, offset));
      
        //抛出错误
        if (fns.length === 0) {
          throw new TypeError('app.use() requires a middleware function')
        }
      
        // setup router
       //实例化Router，并将其赋值给this._router
        this.lazyrouter();
        var router = this._router;
      
        fns.forEach(function (fn) {
          // non-express app
          //遍历参数中的function，逐个调用router.use，从这个地方可以看出，app.use()中可以传入多个function，将其都添加到stack中
          if (!fn || !fn.handle || !fn.set) {
            return router.use(path, fn);
          }
      
          debug('.use app under %s', path);
          fn.mountpath = path;
          fn.parent = this;
      
          // restore .app property on req and res
          router.use(path, function mounted_app(req, res, next) {
            var orig = req.app;
            fn.handle(req, res, function (err) {
              setPrototypeOf(req, orig.request)
              setPrototypeOf(res, orig.response)
              next(err);
            });
          });
      
          // mounted an app
          // app mounted 触发emit
          fn.emit('mount', this);
        }, this);
      
        return this;
      };
      ~~~

      在router/index.js中的router.use方法内，主要是对传入的参数进行判断，并创建layer的实例，将layer添加到 **router.stack**，且**layer.route = undefined;**

      到这里，一个非路由中间件添加完成。具体的区别在说完添加路由中间件之后一起总结。

      router.use代码及备注如下：

      ~~~javascript
      proto.use = function use(fn) {
        var offset = 0;
        var path = '/';
      
        // default path to '/'
        // disambiguate router.use([fn])
        // 默认路径  '/'
        // 消除歧义 router.use([fn])
        // 判断是否是函数
        if (typeof fn !== 'function') {
          var arg = fn;
      
          while (Array.isArray(arg) && arg.length !== 0) {
            arg = arg[0];
          }
      
          // first arg is the path
          // 第一个参数可能是路径，所以要从第二个参数开始截取
          if (typeof arg !== 'function') {
            offset = 1;
            path = fn;
          }
        }
      
           //将arguments转为数组，然后扁平化多维数组
        var callbacks = flatten(slice.call(arguments, offset));
      
        //如果callbacks内没有传递函数，抛错
        if (callbacks.length === 0) {
          throw new TypeError('Router.use() requires a middleware function')
        }
      
         //循环中间件callbacks数组
        for (var i = 0; i < callbacks.length; i++) {
          var fn = callbacks[i];
      
          if (typeof fn !== 'function') {//错误提示
            throw new TypeError('Router.use() requires a middleware function but got a ' + gettype(fn))
          }
      
          // add the middleware
          //解析下query和expressInit的含义
          // 添加中间件
            // 实例化一个layer 对象
          //匿名 anonymous 函数
          debug('use %o %s', path, fn.name || '<anonymous>')
      
          var layer = new Layer(path, {
            sensitive: this.caseSensitive,
            strict: false,
            end: false
          }, fn);
      
           // 非路由中间件，该字段赋值为undefined
          layer.route = undefined;
      
          this.stack.push(layer);// 将layer添加到 router.stack
        }
      
        return this;
      };
      ~~~

      2. 添加路由中间件

         在applicati.js是没有app.get/post等方法的，express将方法封装在一个methods的数组之中，实际上添加路由组件使用的是application.js中的methods.forEach  方法。

         ~~~ JavaScript
         // methods是一个数组集合[get, post, put ...]，里面存放了一系列http请求方法 通过遍历给app上挂载一系列请求方法，即app.get()、app.post() 等
         methods.forEach(function (method) {
           app[method] = function (path) {// app上添加 app.get app.post app.put 等请求方法
         
             //如果只有一个参数，则返回setting的值
             if (method === 'get' && arguments.length === 1) {
               // app.get(setting)
               return this.set(path);
             }
         
             this.lazyrouter();// 创建一个Router实例 this._router = new Router();
         
              // 调用this._router.route => router/index.js中的proto.route
             var route = this._router.route(path);
             route[method].apply(route, slice.call(arguments, 1)); // 添加路由中间件// 调用route中对应的method方法 注册路由
             return this;
           };
         });
         ~~~

         在这个方法内method其实指代的就是get/post等方法。

         其中会创建一个this._router = new Router();实例。因为需要用到router实例的route的方法。

          var route = this._router.route(path);

         那么router.route方法主要是做什么用了？

         ~~~ JavaScript
         proto.route = function route(path) {
           var route = new Route(path);// app[method]注册一个路由 就会创建一个route对象
         
           var layer = new Layer(path, {// 生成一个route layer
             sensitive: this.caseSensitive,
             strict: this.strict,
             end: true
           }, route.dispatch.bind(route));// 将生成的route对象的dispatch作为参数传给layer里面
         
            // 指向刚实例化的路由对象（非常重要），通过该字段将Router和Route关联来起来
           layer.route = route;
         
           this.stack.push(layer);// 将layer添加到Router的stack中
           return route;// 将生成的route对象返回
         };
         ~~~

         可以看到方法内与上面的非路由中间件一样也是创建了一个layer对象，不同的是，这里  **layer.route = route;**。最后返回一个route对象。

         最后调用返回的route的route[method]方法。

         route[method].apply(route, slice.call(arguments, 1)); 

         这一步是添加非路由中间的时候所没有的，查看其源码,注意实在router/route下。

         ~~~ JavaScript
         // 又是相同的一段代码 在给Route的原型上添加http方法
         methods.forEach(function(method){
           Route.prototype[method] = function(){
             var handles = flatten(slice.call(arguments));// 传递进来的处理函数
         
             for (var i = 0; i < handles.length; i++) {
               var handle = handles[i];
         
               if (typeof handle !== 'function') {
                 var type = toString.call(handle);
                 var msg = 'Route.' + method + '() requires a callback function but got a ' + type
                 throw new Error(msg);
               }
         
               debug('%s %o', method, this.path)
         
               var layer = Layer('/', {}, handle);// 在route中也有layer 里面保存着 对于不同HTTP方法的不同method和handle
               layer.method = method;
         
               this.methods[method] = true;// 标识 存在这个method的路由处理函数
               this.stack.push(layer);// 将layer 添加到route的stack中
             }
         
             return this;
           };
         });
         ~~~

         可以看到与上面的router.route代码十分相似，也创建了一个layer对象，但实际上，这里面接受的参数是路径之后的参数如

         app.get('/test',function(req,res,next){ 
              console.log("app.get('/test') handler1");
         next();
         })

         //当/test后面的中间件方法很多时，这里的stack储存的就是对于不同HTTP方法的不同method和handle。

         到这里路由中间件的添加也已完成。

      ## 最后总结一下：

![Image text](https://github.com/sq-github/express-learn-sourceCode/raw/master/imgs/img1.png)

1、添加一个非路由中间和添加路由中间件都会产生一个layer，这个layer都是router的layer，不同的时，非路由中间件：layer.route = undefined;路由中间件：layer.route = route;

2、router和route是两个不同的实例，其中的stack是不同的。路由中间件在产生router的layer后，还会调用route的方法，产生新的layer，这个layer是存在route中的，且没有layer.route属性，只有layer.method = method;







