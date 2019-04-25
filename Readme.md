1. 入口文件抛出`application`、`req`、`res`对象，路由构造函数`Route`、`Router`，常用的中间件，`createApplication`主方法；
2. 如下代码引入 express：

```javascript
var express = require("./index");
var app = express();
```

执行 express 的时候，实际执行的函数是`createApplication`方法，代码如下：

```javascript
function createApplication() {
  var app = function(req, res, next) {
    app.handle(req, res, next);
  };
  mixin(app, EventEmitter.prototype, false);
  mixin(app, proto, false);

  //expose the prototype that will get set on requests
  // expose the prototype that will get set on requests
  app.request = Object.create(req, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  });

  // expose the prototype that will get set on responses
  app.response = Object.create(res, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  });

  app.init();
  return app;
}
```

app 混合了 nodejs 方法 EventEmitter、application 的属性和方法，并把 req、res 挂载在 app 上面，然后执行`app.init`方法。

3. `app.init`方法如下：

```javascript
/**
 * Initialize the server.
 *
 *   - setup default configuration
 *   - setup default middleware
 *   - setup route reflection methods
 *
 * @private
 */
app.init = function init() {
  this.cache = {};
  this.engines = {};
  this.settings = {};

  //Initialize application configuration
  this.defaultConfiguration();
};
```

4.在业务代码中：

```javascript
app.get("/", function(req, res) {
  res.send("Hello World!");
});
```

源码中执行逻辑：

```javascript
/**
 * http的请求方法全部定义到app上，并做相关处理
 * Delegate `.VERB(...)` calls to `router.VERB(...)`.
 */

methods.forEach(function(method) {
  app[method] = function(path) {
    if (method === "get" && arguments.length === 1) {
      // app.get(setting)
      return this.set(path);
    }

    /**
     * lazyrouter方法，检查app._router是否已经生成，在_router上维护了所有的中间件，通过layer生成了路由与中间件的关系
     *{
     *  caseSensitive: false 对于路由的配置，是否识别大小写
     *  mergeParams: undefined
     *  params: {}
     *  stack: (2) [Layer, Layer]  //维护的所有中间件队里
     *  strict: false 路由是否特别严格
     *  _params: []
     * }
     *
     */
    this.lazyrouter();

    /**
     * 生成一个路由对象，放入到堆栈中，并返回这个路由对象，
     *
     * function route(path) {
     *     var route = new Route(path);
     *     var layer = new Layer(
     *       path,
     *       {
     *         sensitive: this.caseSensitive,
     *         strict: this.strict,
     *         end: true
     *       },
     *       route.dispatch.bind(route)
     *     );
     *     layer.route = route;
     *     this.stack.push(layer);
     *     return route;
     *   };
     *
     *
     *
     * route.dispatch.bind(route)
     * 是正常路由的时候，把route.dispatch存起来
     */
    var route = this._router.route(path);

    /**
     * 把当前路由下所有的中间件函数放入到单签路由堆栈中
     */
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```

5. 挂载 app 到某个端口的时候，app 的 handle 方法就被挂载到了回调中，当请求过来的时候，触发流程如下：

```javascript
//[https://github.com/liang520/express/blob/f3addcc00b64ef9e5d2080d6e1397b00e9eea7c9/lib/express.js#L38](app())=>
//app.handle()=>
//app._route.handle()=>
//next()  next 函数中通过while stack来处理 逻辑，当前游标的位置=>
//self.process_params() 解析是否有参数=>回调
//trim_prefix(layer, layerError, layerPath, path)=>
//layer.handle_request(req, res, next) 每个中间件的处理逻辑=>
// fn(req, res, next) =>
// fn中间件函数具体处理逻辑 fn中含有next方法
// 如果是某个路由上，手动挂载的，就触发Route.prototype.dispatch ,这个方法中也维护了某个路由下的中间件处理函数，中间也包含用next
// next() =>
//最后到了自己的处理函数
```

[![Express Logo](https://i.cloudup.com/zfY6lL7eFa-3000x3000.png)](http://expressjs.com/)
