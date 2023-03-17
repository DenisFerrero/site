title: API Gateway
---
## moleculer-web [![npm](https://img.shields.io/npm/v/moleculer-web.svg?maxAge=3600)](https://www.npmjs.com/package/moleculer-web)
[Moleculer-web](https://github.com/moleculerjs/moleculer-web)是Moleculer框架的官方API网关服务。 使用它将您的服务发布为RESTful API。

## 特性概览
* 支持 HTTP & HTTPS
* 提供静态文件服务
* 多样的路由
* 支持在全局级别、路由级别和别名级别使用类似于Connect的中间件。
* 路由别名（支持命名参数和REST路由）
* 白名单
* 多种Body Parsers （Json，Urlencoded）
* 跨域（CORS headers）
* 访问频率限制
* before & after 钩子（Hook）方法
* Buffer & Stream 处理
* 中间件（Middleware）模式（类似Express中间件）

{% note info Try it in your browser! %}
[![Edit moleculer-web](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/github/moleculerjs/sandbox-moleculer-api-routing/tree/master/?fontsize=14)
{% endnote %}

## 安装
```bash
npm i moleculer-web
```

## 使用

### 运行默认设置
这个示例使用默认设置的API Gateway服务。 您可以通过`http://localhost:3000/`访问所有服务（包括内部的`$node.`）。

```js
const { ServiceBroker } = require("moleculer");
const ApiService = require("moleculer-web");

const broker = new ServiceBroker();

// 加载 API 网关服务
broker.createService(ApiService);

// 启动服务器
broker.start();
```

**URLs 示例:**
- 调用`test.hello` action: `http://localhost:3000/test/hello`
- 使用参数调用`math.add` action: `http://localhost:3000/math/add?a=25&b=13`

- 获取节点的健康信息：`http://localhost:3000/~node/health`
- 列出所有Action：`http://localhost:3000/~node/actions`

## 白名单（Whitelist）
如果您不想发布所有的action，可以使用白名单选项进行过滤。 在列表中使用匹配字符串或正则表达式。 _要启用所有action，请使用` "**" `。_

```js
broker.createService({
    mixins: [ApiService],

    settings: {
        routes: [{
            path: "/api",

            whitelist: [
                // Access any actions in 'posts' service
                "posts.*",
                // Access call only the `users.list` action
                "users.list",
                // Access any actions in 'math' service
                /^math\.\w+$/
            ]
        }]
    }
});
```

## Aliases
You can use alias names instead of action names. You can also specify the method. Otherwise it will handle every method types.

Using named parameters in aliases is possible. Named parameters are defined by prefixing a colon to the parameter name (`:name`).

```js
broker.createService({
    mixins: [ApiService],

    settings: {
        routes: [{
            aliases: {
                // Call `auth.login` action with `GET /login` or `POST /login`
                "login": "auth.login",

                // Restrict the request method
                "POST users": "users.create",

                // The `name` comes from named param. 
                // You can access it with `ctx.params.name` in action
                "GET greeter/:name": "test.greeter",
            }
        }]
    }
});
```

{% note info %}
The named parameter is handled with [path-to-regexp](https://github.com/pillarjs/path-to-regexp) module. Therefore you can use [optional](https://github.com/pillarjs/path-to-regexp#optional) and [repeated](https://github.com/pillarjs/path-to-regexp#zero-or-more) parameters, as well.
{% endnote %}

{% note info Aliases Action%}
The API gateway implements `listAliases` [action](actions.html) that lists the HTTP endpoints to actions mappings.
{% endnote %}

You can also create RESTful APIs.
```js
broker.createService({
    mixins: [ApiService],

    settings: {
        routes: [{
            aliases: {
                "GET users": "users.list",
                "GET users/:id": "users.get",
                "POST users": "users.create",
                "PUT users/:id": "users.update",
                "DELETE users/:id": "users.remove"
            }
        }]
    }
});
```

For REST routes you can also use this simple shorthand alias:
```js
broker.createService({
    mixins: [ApiService],

    settings: {
        routes: [{
            aliases: {
                "REST users": "users"
            }
        }]
    }
});
```
{% note warn %}
To use this shorthand alias, create a service which has `list`, `get`, `create`, `update` and `remove` actions.
{% endnote %}

You can make use of custom functions within the declaration of aliases. In this case, the handler's signature is `function (req, res) {...}`.

{% note info %}
Please note that Moleculer uses native Node.js [HTTP server](https://nodejs.org/api/http.html)
{% endnote %}

```js
broker.createService({
    mixins: [ApiService],

    settings: {
        routes: [{
            aliases: {
                "POST upload"(req, res) {
                    this.parseUploadedFile(req, res);
                },
                "GET custom"(req, res) {
                    res.end('hello from custom handler')
                }
            }
        }]
    }
});
```

{% note info %}
There are some internal pointer in `req` & `res` objects:
* `req.$ctx` are pointed to request context.
* `req.$service` & `res.$service` are pointed to this service instance.
* `req.$route` & `res.$route` are pointed to the resolved route definition.
* `req.$params` is pointed to the resolved parameters (from query string & post body)
* `req.$alias` is pointed to the resolved alias definition.
* `req.$action` is pointed to the resolved action.
* `req.$endpoint` is pointed to the resolved action endpoint.
* `req.$next` is pointed to the `next()` handler if the request comes from ExpressJS.

E.g.: To access the broker, use `req.$service.broker`.
{% endnote %}

### Mapping policy
The `route` has a `mappingPolicy` property to handle routes without aliases.

**Available options:**
- `all` - enable to request all routes with or without aliases (default)
- `restrict` - enable to request only the routes with aliases.

```js
broker.createService({
    mixins: [ApiService],

    settings: {
        routes: [{
            mappingPolicy: "restrict",
            aliases: {
                "POST add": "math.add"
            }
        }]
    }
});
```
You can't request the `/math.add` or `/math/add` URLs, only `POST /add`.

### File upload aliases
API Gateway has implemented file uploads. You can upload files as a multipart form data (thanks to [busboy](https://github.com/mscdex/busboy) library) or as a raw request body. In both cases, the file is transferred to an action as a `Stream`. In multipart form data mode you can upload multiple files, as well.

**示例**
```js
const ApiGateway = require("moleculer-web");

module.exports = {
    mixins: [ApiGateway],
    settings: {
        path: "/upload",

        routes: [
            {
                path: "",

                aliases: {
                    // File upload from HTML multipart form
                    "POST /": "multipart:file.save",

                    // File upload from AJAX or cURL
                    "PUT /:id": "stream:file.save",

                    // File upload from HTML form and overwrite busboy config
                    "POST /multi": {
                        type: "multipart",
                        // Action level busboy config
                        busboyConfig: {
                            limits: { files: 3 }
                        },
                        action: "file.save"
                    }
                },

                // Route level busboy config.
                // More info: https://github.com/mscdex/busboy#busboy-methods
                busboyConfig: {
                    limits: { files: 1 }
                    // Can be defined limit event handlers
                    // `onPartsLimit`, `onFilesLimit` or `onFieldsLimit`
                },

                mappingPolicy: "restrict"
            }
        ]
    }
});
```
**Multipart parameters**

In order to access the files passed by multipart-form these specific fields can be used inside the action:
- `ctx.params` is the Readable stream containing the file passed to the endpoint
- `ctx.meta.$params` parameters from URL querystring
- `ctx.meta.$multipart` contains the additional text form-data fields _must be sent before other files fields_.

### Auto-alias
The auto-alias feature allows you to declare your route alias directly in your services. The gateway will dynamically build the full routes from service schema.

{% note info %}
Gateway will regenerate the routes every time a service joins or leaves the network.
{% endnote %}

Use `whitelist` parameter to specify services that the Gateway should track and build the routes.

**示例**
```js
// api.service.js
module.exports = {
    mixins: [ApiGateway],

    settings: {
        routes: [
            {
                path: "/api",

                whitelist: [
                    "v2.posts.*",
                    "test.*"
                ],

                aliases: {
                    "GET /hi": "test.hello"
                },

                autoAliases: true
            }
        ]
    }
};
```

```js
// posts.service.js
module.exports = {
    name: "posts",
    version: 2,

    settings: {
        // Base path
        // rest: "posts/" // If you want to change the base 
        // path with /api/posts instead 
        // of /api/v2/posts, you can uncomment this line.
    },

    actions: {
        list: {
            // Expose as "/api/v2/posts/"
            rest: "GET /",
            handler(ctx) {}
        },

        get: {
            // Expose as "/api/v2/posts/:id"
            rest: "GET /:id",
            handler(ctx) {}
        },

        create: {
            rest: "POST /",
            handler(ctx) {}
        },

        update: {
            rest: "PUT /:id",
            handler(ctx) {}
        },

        remove: {
            rest: "DELETE /:id",
            handler(ctx) {}
        }
    }
};
```

**The generated aliases**

```bash
    GET     /api/hi             => test.hello
    GET     /api/v2/posts       => v2.posts.list
    GET     /api/v2/posts/:id   => v2.posts.get
    POST    /api/v2/posts       => v2.posts.create
    PUT     /api/v2/posts/:id   => v2.posts.update
    DELETE  /api/v2/posts/:id   => v2.posts.remove
```

**Service level rest parameters**

- **fullPath**, override all the path generated with a new custom one
- **basePath**, path to the service, by default is the one declared in `settings.rest`
- **path**, path to the action
- **method**, method used to access the action

path is appended after the basePath The combination path+basePath it's not the same as using fullPath. For example:

```js
// posts.service.js
module.exports = {
    name: "posts",
    version: 2,

    settings: {
        // Base path
        rest: "posts/"
    },

    actions: {
        tags: {
            // Expose as "/tags" instead of "/api/v2/posts/tags"
            rest: [{
                method: "GET",
                fullPath: "/tags"
            }, {
                method: "GET",
                basePath: "/my/awesome"
            }],
            handler(ctx) {}
        }
    }
};
```

Will create those endpoints:

```bash
    GET     /tags
    GET     /api/my/awesome/tags
```

fullPath ignores that prefix applied in the API gateway!

The *rest* param can be also be an array with elements with same structure discussed before. The can be applied both on settings and action level. For example:

```js
// posts.service.js
module.exports = {
  name: "posts",
  settings: {
    rest: 'my/awesome/posts'
  },
  actions: {
    get: {
      rest: [
        "GET /:id",
        { method: 'GET', fullPath: '/posts' }
        { method: 'GET', path: '/' }, 
        { method: 'GET', path: '/:id', basePath: 'demo_posts' }
    ],
      handler(ctx) {}
    },
  }
};
```

Produce those endpoints

```bash
    GET     /api/my/awesome/posts/:id/  => posts.get
    GET     /posts                      => posts.get
    GET     /api/my/awesome/posts/      => posts.get
    POST    /api/demo_posts/:id         => posts.get
```

## Parameters
API gateway collects parameters from URL querystring, request params & request body and merges them. The results is placed to the `req.$params`.

### Disable merging
To disable parameter merging set `mergeParams: false` in route settings. In this case the parameters is separated.

**示例**
```js
broker.createService({
    mixins: [ApiService],
    settings: {
        routes: [{
            path: "/",
            mergeParams: false
        }]
    }
});
```

**Un-merged `req.$params`:**
```js
{
    // Querystring params
    query: {
        category: "general",
    }

    // Request body content
    body: {
        title: "Hello",
        content: "...",
        createdAt: 1530796920203
    },

    // Request params
    params: {
        id: 5
    }
}
```

### Query string parameters
More information: https://github.com/ljharb/qs

**Array parameters** URL: `GET /api/opt-test?a=1&a=2`
```js
a: ["1", "2"]
```

**Nested objects & arrays** URL: `GET /api/opt-test?foo[bar]=a&foo[bar]=b&foo[baz]=c`
```js
foo: { 
    bar: ["a", "b"], 
    baz: "c" 
}
```

## 中间件
It supports Connect-like middlewares in global-level, route-level & alias-level. Signature: `function(req, res, next) {...}`. For more info check [express middleware](https://expressjs.com/en/guide/using-middleware.html)

**示例**
```js
broker.createService({
    mixins: [ApiService],
    settings: {
        // Global middlewares. Applied to all routes.
        use: [
            cookieParser(),
            helmet()
        ],

        routes: [
            {
                path: "/",

                // Route-level middlewares.
                use: [
                    compression(),

                    passport.initialize(),
                    passport.session(),

                    serveStatic(path.join(__dirname, "public"))
                ],

                aliases: {
                    "GET /secret": [
                        // Alias-level middlewares.
                        auth.isAuthenticated(),
                        auth.hasRole("admin"),
                        "top.secret" // Call the `top.secret` action
                    ]
                }
            }
        ]
    }
});
```
Use [swagger-stats UI](https://swaggerstats.io/) for quick look on the "health" of your API (TypeScript)
```ts
import { Service, ServiceSchema } from "moleculer";
import ApiGatewayService from "moleculer-web";
const swStats = require("swagger-stats");

const swMiddleware = swStats.getMiddleware();

broker.createService({
    mixins: [ApiGatewayService],
    name: "gw-main",

    settings: {
        cors: {
            methods: ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"],
            origin: "*",
        },

        routes: [
            // ...
        ],

        use: [swMiddleware],
    },

    async started(this: Service): Promise<void> {
        this.addRoute({
            path: "/",
            use: [swMiddleware],
        });
    },
} as ServiceSchema);
```

### Error-handler middleware
There is support to use error-handler middlewares in the API Gateway. So if you pass an `Error` to the `next(err)` function, it will call error handler middlewares which have signature as `(err, req, res, next)`.

```js
broker.createService({
    mixins: [ApiService],
    settings: {
        // Global middlewares. Applied to all routes.
        use: [
            cookieParser(),
            helmet()
        ],

        routes: [
            {
                path: "/",

                // Route-level middlewares.
                use: [
                    compression(),

                    passport.initialize(),
                    passport.session(),

                    function(err, req, res, next) {
                        this.logger.error("Error is occured in middlewares!");
                        this.sendError(req, res, err);
                    }
                ],
```

## Serve static files
It serves assets with the [serve-static](https://github.com/expressjs/serve-static) module like ExpressJS.

```js
broker.createService({
    mixins: [ApiService],

    settings: {
        assets: {
            // Root folder of assets
            folder: "./assets",

            // Further options to `serve-static` module
            options: {}
        }       
    }
});
```

## Calling options
The `route` has a `callOptions` property which is passed to `broker.call`. So you can set `timeout`, `retries` or `fallbackResponse` options for routes. [Read more about calling options](actions.html#Call-services)

{% note info %}
Please note that you can also set the timeout for an action directly in its [definition](actions.html#Timeout)
{% endnote %}

```js
broker.createService({
    mixins: [ApiService],

    settings: {
        routes: [{

            callOptions: {
                timeout: 500,
                retries: 3,
                fallbackResponse(ctx, err) { ... }
            }

        }]
    }
});
```

## Multiple routes
You can create multiple routes with different prefix, whitelist, alias, calling options & authorization.

{% note info %}
When using multiple routes you should explicitly set the body parser(s) for each route.
{% endnote %}

```js
broker.createService({
    mixins: [ApiService],

    settings: {
        routes: [
            {
                path: "/admin",

                authorization: true,

                whitelist: [
                    "$node.*",
                    "users.*",
                ],

                bodyParsers: {
                    json: true
                }
            },
            {
                path: "/",

                whitelist: [
                    "posts.*",
                    "math.*",
                ],

                bodyParsers: {
                    json: true
                }
            }
        ]
    }
});
```

## Response type & status code
When the response is received from an action handler, the API gateway detects the type of response and set the `Content-Type` in the `res` headers. The status code is `200` by default. Of course you can overwrite these values, moreover, you can define custom response headers, too.

To define response headers & status code use `ctx.meta` fields:

**Available meta fields:**
* `ctx.meta.$statusCode` - set `res.statusCode`.
* `ctx.meta.$statusMessage` - set `res.statusMessage`.
* `ctx.meta.$responseType` - set `Content-Type` in header.
* `ctx.meta.$responseHeaders` - set all keys in header.
* `ctx.meta.$location` - set `Location` key in header for redirects.

**示例**
```js
module.exports = {
    name: "export",
    actions: {
        // Download response as a file in the browser
        downloadCSV(ctx) {
            ctx.meta.$responseType = "text/csv";
            ctx.meta.$responseHeaders = {
                "Content-Disposition": `attachment; filename="data-${ctx.params.id}.csv"`
            };

            return csvFileStream;
        },

        // Redirect the request
        redirectSample(ctx) {
            ctx.meta.$statusCode = 302;
            ctx.meta.$location = "/login";

            return;
        }
    }
}
```

## Authorization
You can implement authorization. Do 2 things to enable it.
1. Set `authorization: true` in your routes
2. Define the `authorize` method in service.

**Example authorization**
```js
const E = require("moleculer-web").Errors;

broker.createService({
    mixins: [ApiService],

    settings: {
        routes: [{
            // First thing
            authorization: true
        }]
    },

    methods: {
        // Second thing
        authorize(ctx, route, req, res) {
            // Read the token from header
            let auth = req.headers["authorization"];
            if (auth && auth.startsWith("Bearer")) {
                let token = auth.slice(7);

                // Check the token
                if (token == "123456") {
                    // Set the authorized user entity to `ctx.meta`
                    ctx.meta.user = { id: 1, name: "John Doe" };
                    return Promise.resolve(ctx);

                } else {
                    // Invalid token
                    return Promise.reject(new E.UnAuthorizedError(E.ERR_INVALID_TOKEN));
                }

            } else {
                // No token
                return Promise.reject(new E.UnAuthorizedError(E.ERR_NO_TOKEN));
            }
        }

    }
}
```

{% note info %}
You can find a more detailed role-based JWT authorization example in [full example](https://github.com/moleculerjs/moleculer-web/blob/master/examples/full/index.js#L239).
{% endnote %}

## Authentication
To enable the support for authentication, you need to do something similar to what is describe in the Authorization paragraph. Also in this case you have to:
1. Set `authentication: true` in your routes
2. Define your custom `authenticate` method in your service

The returned value will be set to the `ctx.meta.user` property. You can use it in your actions to get the logged in user entity.

**Example authentication**
```js
broker.createService({
    mixins: ApiGatewayService,

    settings: {
        routes: [{
            // Enable authentication
            authentication: true
        }]
    },

    methods: {
        authenticate(ctx, route, req, res) {
            let accessToken = req.query["access_token"];
            if (accessToken) {
                if (accessToken === "12345") {
                    // valid credentials. It will be set to `ctx.meta.user`
                    return Promise.resolve({ id: 1, username: "john.doe", name: "John Doe" });
                } else {
                    // invalid credentials
                    return Promise.reject();
                }
            } else {
                // anonymous user
                return Promise.resolve(null);
            }
        }
    }
});
```

## Route hooks
The `route` has before & after call hooks. You can use it to set `ctx.meta`, access `req.headers` or modify the response `data`.

```js
broker.createService({
    mixins: [ApiService],

    settings: {
        routes: [
            {
                path: "/",

                onBeforeCall(ctx, route, req, res) {
                    // Set request headers to context meta
                    ctx.meta.userAgent = req.headers["user-agent"];
                },

                onAfterCall(ctx, route, req, res, data) {
                    // Async function which return with Promise
                    return doSomething(ctx, res, data);
                }
            }
        ]
    }
});
```

> In previous versions of Moleculer Web, you couldn't manipulate the `data` in `onAfterCall`. Now you can, but you must always return the new or original `data`.


## Error handlers
You can add route-level & global-level custom error handlers.
> In handlers, you must call the `res.end`. Otherwise, the request is unhandled.

```js
broker.createService({
    mixins: [ApiService],
    settings: {

        routes: [{
            path: "/api",

            // Route error handler
            onError(req, res, err) {
                res.setHeader("Content-Type", "application/json; charset=utf-8");
                res.writeHead(500);
                res.end(JSON.stringify(err));
            }
        }],

        // Global error handler
        onError(req, res, err) {
            res.setHeader("Content-Type", "text/plain");
            res.writeHead(501);
            res.end("Global error: " + err.message);
        }       
    }
}
```

### Error formatter
API gateway implements a helper function that formats the error. You can use it to filter out the unnecessary data.

```js
broker.createService({
    mixins: [ApiService],
    methods: {
        reformatError(err) {
            // Filter out the data from the error before sending it to the client
            return _.pick(err, ["name", "message", "code", "type", "data"]);
        },
    }
}
```

## CORS headers
You can use [CORS](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) headers in Moleculer-Web service.

**用例**
```js
const svc = broker.createService({
    mixins: [ApiService],

    settings: {

        // Global CORS settings for all routes
        cors: {
            // Configures the Access-Control-Allow-Origin CORS header.
            origin: "*",
            // Configures the Access-Control-Allow-Methods CORS header. 
            methods: ["GET", "OPTIONS", "POST", "PUT", "DELETE"],
            // Configures the Access-Control-Allow-Headers CORS header.
            allowedHeaders: [],
            // Configures the Access-Control-Expose-Headers CORS header.
            exposedHeaders: [],
            // Configures the Access-Control-Allow-Credentials CORS header.
            credentials: false,
            // Configures the Access-Control-Max-Age CORS header.
            maxAge: 3600
        },

        routes: [{
            path: "/api",

            // Route CORS settings (overwrite global settings)
            cors: {
                origin: ["http://localhost:3000", "https://localhost:4000"],
                methods: ["GET", "OPTIONS", "POST"],
                credentials: true
            },
        }]
    }
});
```

## Rate limiter
The Moleculer-Web has a built-in rate limiter with a memory store.

**用例**
```js
const svc = broker.createService({
    mixins: [ApiService],

    settings: {
        rateLimit: {
            // How long to keep record of requests in memory (in milliseconds). 
            // Defaults to 60000 (1 min)
            window: 60 * 1000,

            // Max number of requests during window. Defaults to 30
            limit: 30,

            // Set rate limit headers to response. Defaults to false
            headers: true,

            // Function used to generate keys. Defaults to: 
            key: (req) => {
                return req.headers["x-forwarded-for"] ||
                    req.connection.remoteAddress ||
                    req.socket.remoteAddress ||
                    req.connection.socket.remoteAddress;
            },
            //StoreFactory: CustomStore
        }
    }
});
```

### Custom Store example
```js
class CustomStore {
    constructor(clearPeriod, opts) {
        this.hits = new Map();
        this.resetTime = Date.now() + clearPeriod;

        setInterval(() => {
            this.resetTime = Date.now() + clearPeriod;
            this.reset();
        }, clearPeriod);
    }

    /**
     * Increment the counter by key
     *
     * @param {String} key
     * @returns {Number}
     */
    inc(key) {
        let counter = this.hits.get(key) || 0;
        counter++;
        this.hits.set(key, counter);
        return counter;
    }

    /**
     * Reset all counters
     */
    reset() {
        this.hits.clear();
    }
}
```

## ETag

The `etag` option value can be `false`, `true`, `weak`, `strong`, or a custom `Function`. For full details check the [code](https://github.com/moleculerjs/moleculer-web/pull/92).

```js
const ApiGateway = require("moleculer-web");

module.exports = {
    mixins: [ApiGateway],
    settings: {
        // Service-level option
        etag: false,
        routes: [
            {
                path: "/",
                // Route-level option.
                etag: true
            }
        ]
    }
}
```

**Custom `etag` Function**
```js
module.exports = {
    mixins: [ApiGateway],
    settings: {
        // Service-level option
        etag: (body) => generateHash(body)
    }
}
```

Please note, it doesn't work with stream responses. In this case, you should generate the `etag` by yourself.

**Custom `etag` for streaming**
```js
module.exports = {
    name: "export",
    actions: {
        // Download response as a file in the browser
        downloadCSV(ctx) {
            ctx.meta.$responseType = "text/csv";
            ctx.meta.$responseHeaders = {
                "Content-Disposition": `attachment; filename="data-${ctx.params.id}.csv"`,
                "ETag": '<your etag here>'
            };
            return csvFileStream;
        }
    }
}
```

## HTTP2 Server
API Gateway provides an experimental support for HTTP2. You can turn it on with `http2: true` in service settings. **示例**
```js
const ApiGateway = require("moleculer-web");

module.exports = {
    mixins: [ApiGateway],
    settings: {
        port: 8443,

        // HTTPS server with certificate
        https: {
            key: fs.readFileSync("key.pem"),
            cert: fs.readFileSync("cert.pem")
        },

        // Use HTTP2 server
        http2: true
    }
});
```

## ExpressJS middleware usage
You can use Moleculer-Web as a middleware in an [ExpressJS](http://expressjs.com/) application.

**用例**
```js
const svc = broker.createService({
    mixins: [ApiService],

    settings: {
        server: false // Default is "true"
    }
});

// Create Express application
const app = express();

// Use ApiGateway as middleware
app.use("/api", svc.express());

// Listening
app.listen(3000);

// Start server
broker.start();
```


## Full service settings
List of all settings of Moleculer Web service:

```js
settings: {

    // Exposed port
    port: 3000,

    // Exposed IP
    ip: "0.0.0.0",

    // HTTPS server with certificate
    https: {
        key: fs.readFileSync("ssl/key.pem"),
        cert: fs.readFileSync("ssl/cert.pem")
    },

    // Used server instance. If null, it will create a new HTTP(s)(2) server
    // If false, it will start without server in middleware mode
    server: true,

    // Exposed global path prefix
    path: "/api",

    // Global-level middlewares
    use: [
        compression(),
        cookieParser()
    ],

    // Logging request parameters with 'info' level
    logRequestParams: "info",

    // Logging response data with 'debug' level
    logResponseData: "debug",

    // Use HTTP2 server (experimental)
    http2: false,

    // Override HTTP server default timeout
    httpServerTimeout: null,

    // Optimize route & alias paths (deeper first).
    optimizeOrder: true,

    // Routes
    routes: [
        {
            // Path prefix to this route  (full path: /api/admin )
            path: "/admin",

            // Whitelist of actions (array of string mask or regex)
            whitelist: [
                "users.get",
                "$node.*"
            ],

            // Call the `this.authorize` method before call the action
            authorization: true,

            // Merge parameters from querystring, request params & body 
            mergeParams: true,

            // Route-level middlewares
            use: [
                helmet(),
                passport.initialize()
            ],

            // Action aliases
            aliases: {
                "POST users": "users.create",
                "health": "$node.health"
            },

            mappingPolicy: "all",

            // Use bodyparser module
            bodyParsers: {
                json: true,
                urlencoded: { extended: true }
            }
        },
        {
            // Path prefix to this route  (full path: /api )
            path: "",

            // Whitelist of actions (array of string mask or regex)
            whitelist: [
                "posts.*",
                "file.*",
                /^math\.\w+$/
            ],

            // No authorization
            authorization: false,

            // Action aliases
            aliases: {
                "add": "math.add",
                "GET sub": "math.sub",
                "POST divide": "math.div",
                "GET greeter/:name": "test.greeter",
                "GET /": "test.hello",
                "POST upload"(req, res) {
                    this.parseUploadedFile(req, res);
                }
            },

            mappingPolicy: "restrict",

            // Use bodyparser module
            bodyParsers: {
                json: false,
                urlencoded: { extended: true }
            },

            // Calling options
            callOptions: {
                timeout: 3000,
                retries: 3,
                fallbackResponse: "Static fallback response"
            },

            // Call before `broker.call`
            onBeforeCall(ctx, route, req, res) {
                ctx.meta.userAgent = req.headers["user-agent"];
            },

            // Call after `broker.call` and before send back the response
            onAfterCall(ctx, route, req, res, data) {
                res.setHeader("X-Custom-Header", "123456");
                return data;
            },

            // Route error handler
            onError(req, res, err) {
                res.setHeader("Content-Type", "text/plain");
                res.writeHead(err.code || 500);
                res.end("Route error: " + err.message);
            }
        }
    ],

    // Folder to server assets (static files)
    assets: {
        // Root folder of assets
        folder: "./examples/www/assets",

        // Options to `server-static` module
        options: {}
    },

    // Global error handler
    onError(req, res, err) {
        res.setHeader("Content-Type", "text/plain");
        res.writeHead(err.code || 500);
        res.end("Global error: " + err.message);
    }    
}
```
## Service Methods
### `addRoute`
This service [method](services.html#Methods) (`this.addRoute(opts, toBottom = true)`) add/replace a route. For example, you can call it from your mixins to define new routes (e.g. swagger route, graphql route, etc.).

> Please note that if a route already exists this method will replace previous route configuration with a new one.

### `removeRoute`
Service method removes the route by path (`this.removeRoute("/admin")`).

## 示例
- [Simple](https://github.com/moleculerjs/moleculer-web/blob/master/examples/simple/index.js)
    - simple gateway with default settings.

- [SSL server](https://github.com/moleculerjs/moleculer-web/blob/master/examples/ssl/index.js)
    - open HTTPS server
    - whitelist handling

- [WWW with assets](https://github.com/moleculerjs/moleculer-web/blob/master/examples/www/index.js)
    - serve static files from the `assets` folder
    - whitelist
    - aliases
    - multiple body-parsers

- [Authorization](https://github.com/moleculerjs/moleculer-web/blob/master/examples/authorization/index.js)
    - simple authorization demo
    - set the authorized user to `Context.meta`

- [REST](https://github.com/moleculerjs/moleculer-web/blob/master/examples/rest/index.js)
    - simple server with RESTful aliases
    - example `posts` service with CRUD actions

- [Express](https://github.com/moleculerjs/moleculer-web/blob/master/examples/express/index.js)
    - webserver with Express
    - use moleculer-web as a middleware

- [Socket.io](https://github.com/moleculerjs/moleculer-web/blob/master/examples/socket.io/index.js)
    - start socket.io websocket server
    - call action and send back the response via websocket
    - send Moleculer events to the browser via websocket

- [Full](https://github.com/moleculerjs/moleculer-web/blob/master/examples/full/index.js)
    - SSL
    - static files
    - middlewares
    - multiple routes with different roles
    - role-based authorization with JWT
    - whitelist
    - aliases with named params
    - multiple body-parsers
    - before & after hooks
    - metrics, statistics & validation from Moleculer
    - custom error handlers

- [Webpack](https://github.com/moleculerjs/moleculer-web/blob/master/examples/webpack)
    - Webpack development environment for client-side developing
    - webpack config file
    - compression
    - static file serving

- [Webpack-Vue](https://github.com/moleculerjs/moleculer-web/blob/master/examples/webpack-vue)
    - Webpack+Vue development environment for VueJS client developing
    - webpack config file
    - Hot-replacement
    - Babel, SASS, SCSS, Vue SFC
