# Kernel Handle

`App\Http\Kernel` 继承自 `Illuminate\Foundation\Http\Kernel` 类，所以本文章的分析主要集中在 [app/Http/Kernel.php](https://github.com/xiaohuilam/laravel/blob/master/app/Http/Kernel.php) 和 [vendor/laravel/framework/src/Illuminate/Foundation/Http/Kernel.php](https://github.com/laravel/framework/blob/5.7/src/Illuminate/Foundation/Http/Kernel.php) 两个类中

## __construct 解析

因为 `App\Http\Kernel` 没有 __construct 方法，所以穿透到了 `Illuminate\Foundation\Http\Kernel` 的 __construct：
```php
     /** 
      * Create a new HTTP kernel instance. 
      * 
      * @param  \Illuminate\Contracts\Foundation\Application  $app 
      * @param  \Illuminate\Routing\Router  $router 
      * @return void 
      */ 
     public function __construct(Application $app, Router $router) 
     { 
         $this->app = $app; 
         $this->router = $router; 
  
         $router->middlewarePriority = $this->middlewarePriority; 
  
         foreach ($this->middlewareGroups as $key => $middleware) { 
             $router->middlewareGroup($key, $middleware); 
         } 
  
         foreach ($this->routeMiddleware as $key => $middleware) { 
             $router->aliasMiddleware($key, $middleware); 
         } 
     } 
```

首先是此方法传入参数的 ` __construct(Application $app, Router $router)` 声明，这里恰好使用了 Laravel 容器的依赖注入(Dependency Injection，Inversion of Control的一种)设计。具体的在本篇不讲述，可参见本篇末尾的单独分析的链接的文章。 
 
执行到 __construct 时， `Illuminate\Foundation\Application` 容器会将 `Illuminate\Foundation\Application`（即应用容器自身）和 `Illuminate\Routing\Router` 注入到方法内。然后逻辑代码将两个对象赋给 `App\Http\Kernel` 的 `$this` 属性中。
 
然后将  `Illuminate\Foundation\Http\Kernel` 的 $middlewarePriority 属性
```php
     /** 
      * The priority-sorted list of middleware. 
      * 
      * Forces the listed middleware to always be in the given order. 
      * 
      * @var array 
      */ 
     protected $middlewarePriority = [ 
         \Illuminate\Session\Middleware\StartSession::class, 
         \Illuminate\View\Middleware\ShareErrorsFromSession::class, 
         \Illuminate\Auth\Middleware\Authenticate::class, 
         \Illuminate\Session\Middleware\AuthenticateSession::class, 
         \Illuminate\Routing\Middleware\SubstituteBindings::class, 
         \Illuminate\Auth\Middleware\Authorize::class, 
     ]; 
```

赋值给 `Illuminate\Routing\Router` 的 $middlewarePriority 属性

根据其注释，此属性是强制对 middleward 中间件执行顺序进行排序的作用。

在后面将 `App\Http\Kernel` 中声明的 `$middlewareGroup`
```php
     /** 
      * The application's route middleware groups. 
      * 
      * @var array 
      */ 
     protected $middlewareGroups = [ 
         'web' => [ 
             \App\Http\Middleware\EncryptCookies::class, 
             \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class, 
             \Illuminate\Session\Middleware\StartSession::class, 
             // \Illuminate\Session\Middleware\AuthenticateSession::class, 
             \Illuminate\View\Middleware\ShareErrorsFromSession::class, 
             \App\Http\Middleware\VerifyCsrfToken::class, 
             \Illuminate\Routing\Middleware\SubstituteBindings::class, 
         ], 
  
         'api' => [ 
             'throttle:60,1', 
             'bindings', 
         ], 
     ]; 
```

在 `Illuminate\Foundation\Http\Kernel` 的 96-98行，调用 `Illuminate\Routing\Router` 的 middlewareGroup 方法，存到 $router 的 $middlewareGroup 中
```php
     /** 
      * Register a group of middleware. 
      * 
      * @param  string  $name 
      * @param  array  $middleware 
      * @return $this 
      */ 
     public function middlewareGroup($name, array $middleware) 
     { 
         $this->middlewareGroups[$name] = $middleware; 
  
         return $this; 
     } 
```

紧接着，将
```php
     /** 
      * The application's route middleware. 
      * 
      * These middleware may be assigned to groups or used individually. 
      * 
      * @var array 
      */ 
     protected $routeMiddleware = [ 
         'auth' => \App\Http\Middleware\Authenticate::class, 
         'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class, 
         'bindings' => \Illuminate\Routing\Middleware\SubstituteBindings::class, 
         'cache.headers' => \Illuminate\Http\Middleware\SetCacheHeaders::class, 
         'can' => \Illuminate\Auth\Middleware\Authorize::class, 
         'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class, 
         'signed' => \Illuminate\Routing\Middleware\ValidateSignature::class, 
         'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class, 
         'verified' => \Illuminate\Auth\Middleware\EnsureEmailIsVerified::class, 
     ]; 
```

的中间件，调用 `Illuminate\Routing\Router` 的 aliasMiddleware 的方法，绑定到 $router 的 $middleware 中
```php
     /** 
      * Register a short-hand name for a middleware. 
      * 
      * @param  string  $name 
      * @param  string  $class 
      * @return $this 
      */ 
     public function aliasMiddleware($name, $class) 
     { 
         $this->middleware[$name] = $class; 
  
         return $this; 
     } 
```

至此， `Kernel::__construct()` 解析完毕。

## handle 解析

```php
     /** 
      * Handle an incoming HTTP request. 
      * 
      * @param  \Illuminate\Http\Request  $request 
      * @return \Illuminate\Http\Response 
      */ 
     public function handle($request) 
     { 
         try { 
             $request->enableHttpMethodParameterOverride(); 
  
             $response = $this->sendRequestThroughRouter($request); 
         } catch (Exception $e) { 
             $this->reportException($e); 
  
             $response = $this->renderException($request, $e); 
         } catch (Throwable $e) { 
             $this->reportException($e = new FatalThrowableError($e)); 
  
             $response = $this->renderException($request, $e); 
         } 
  
         $this->app['events']->dispatch( 
             new Events\RequestHandled($request, $response) 
         ); 
  
         return $response; 
     } 
```

调用到 `Symfony\Component\HttpFoundation\Request` 的 enableHttpMethodParameterOverride 方法
> `Illuminate\Http\Request` 继承自 `Symfony\Component\HttpFoundation\Request`  
> 并且 `Illuminate\Http\Request` 未覆盖 enableHttpMethodParameterOverride

```php
     /** 
      * Enables support for the _method request parameter to determine the intended HTTP method. 
      * 
      * Be warned that enabling this feature might lead to CSRF issues in your code. 
      * Check that you are using CSRF tokens when required. 
      * If the HTTP method parameter override is enabled, an html-form with method "POST" can be altered 
      * and used to send a "PUT" or "DELETE" request via the _method request parameter. 
      * If these methods are not protected against CSRF, this presents a possible vulnerability. 
      * 
      * The HTTP method can only be overridden when the real HTTP method is POST. 
      */ 
     public static function enableHttpMethodParameterOverride() 
     { 
         self::$httpMethodParameterOverride = true; 
     }
```


然后，调用 `Illuminate\Foundation\Http\Kernel` 的 sendRequestThroughRouter
```php
     /** 
      * Send the given request through the middleware / router. 
      * 
      * @param  \Illuminate\Http\Request  $request 
      * @return \Illuminate\Http\Response 
      */ 
     protected function sendRequestThroughRouter($request) 
     { 
         $this->app->instance('request', $request); 
  
         Facade::clearResolvedInstance('request'); 
  
         $this->bootstrap(); 
  
         return (new Pipeline($this->app)) 
                     ->send($request) 
                     ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware) 
                     ->then($this->dispatchToRouter()); 
     } 
```

通过第142行，将 $request 注入进 `Illuminate\Foundation\Application` 容器。 
 
144行，将门面类中的 `request` 数据清理掉
```php
     /** 
      * Clear a resolved facade instance. 
      * 
      * @param  string  $name 
      * @return void 
      */ 
     public static function clearResolvedInstance($name) 
     { 
         unset(static::$resolvedInstance[$name]); 
     }
```

## Bootstrap 解析

然后146行，调用  `Illuminate\Foundation\Http\Kernel` 的 bootstrap 方法
```php

     /** 
      * Bootstrap the application for HTTP requests. 
      * 
      * @return void 
      */ 
     public function bootstrap() 
     { 
         if (! $this->app->hasBeenBootstrapped()) { 
             $this->app->bootstrapWith($this->bootstrappers()); 
         } 
     } 
```

第161执行到容器的 hasBeenBootstrapped 方法
```php

     /** 
      * Determine if the application has been bootstrapped before. 
      * 
      * @return bool 
      */ 
     public function hasBeenBootstrapped() 
     { 
         return $this->hasBeenBootstrapped; 
     }
```

第162实际得到这个数组
```php
     /** 
      * The bootstrap classes for the application. 
      * 
      * @var array 
      */ 
     protected $bootstrappers = [ 
         \Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables::class, 
         \Illuminate\Foundation\Bootstrap\LoadConfiguration::class, 
         \Illuminate\Foundation\Bootstrap\HandleExceptions::class, 
         \Illuminate\Foundation\Bootstrap\RegisterFacades::class, 
         \Illuminate\Foundation\Bootstrap\RegisterProviders::class, 
         \Illuminate\Foundation\Bootstrap\BootProviders::class, 
     ]; 
```

然后将数组做为参数，执行容器的 `bootstrapWith` 方法
```php
     /** 
      * Run the given array of bootstrap classes. 
      * 
      * @param  array  $bootstrappers 
      * @return void 
      */ 
     public function bootstrapWith(array $bootstrappers) 
     { 
         $this->hasBeenBootstrapped = true; 
  
         foreach ($bootstrappers as $bootstrapper) { 
             $this['events']->fire('bootstrapping: '.$bootstrapper, [$this]); 
  
             $this->make($bootstrapper)->bootstrap($this); 
  
             $this['events']->fire('bootstrapped: '.$bootstrapper, [$this]); 
         } 
     }
```

特别留意206行的 `make` 调用
> 关于容器 `make` 方法的细节
> 请查阅 [10. 容器的 singleton 和 bind 的实现](https://github.com/xiaohuilam/laravel/issues/10) 的 “揭开 Container::make() 神秘的面纱” 段落

```php
     /** 
      * Resolve the given type from the container. 
      * 
      * (Overriding Container::make) 
      * 
      * @param  string  $abstract 
      * @param  array  $parameters 
      * @return mixed 
      */ 
     public function make($abstract, array $parameters = []) 
     { 
         $abstract = $this->getAlias($abstract); 
  
         if (isset($this->deferredServices[$abstract]) && ! isset($this->instances[$abstract])) { 
             $this->loadDeferredProvider($abstract); 
         } 
  
         return parent::make($abstract, $parameters); 
     } 
```

关于 loadDeferredProvider 的逻辑会最终执行到
```php
     /** 
      * Register a deferred provider and service. 
      * 
      * @param  string  $provider 
      * @param  string|null  $service 
      * @return void 
      */ 
     public function registerDeferredProvider($provider, $service = null) 
     { 
         // Once the provider that provides the deferred service has been registered we 
         // will remove it from our local list of the deferred services with related 
         // providers so that this container does not try to resolve it out again. 
         if ($service) { 
             unset($this->deferredServices[$service]); 
         } 
  
         $this->register($instance = new $provider($this)); 
  
         if (! $this->booted) { 
             $this->booting(function () use ($instance) { 
                 $this->bootProvider($instance); 
             }); 
         } 
     } 
```

第710行的 booting 方法，是登记一个闭包 (并不会马上执行这个闭包)， 然后这个服务提供者在 `boot()` 阶段的 `$this->fireAppCallbacks($this->bootingCallbacks)` 才会真正被创建。 关联阅读请见 [04. ServiceProvider Boot 解析](https://github.com/xiaohuilam/laravel/issues/4)

然后结果就是依次执行
* `\Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables`
* `\Illuminate\Foundation\Bootstrap\LoadConfiguration`
* `\Illuminate\Foundation\Bootstrap\HandleExceptions`
* `\Illuminate\Foundation\Bootstrap\RegisterFacades`
* `\Illuminate\Foundation\Bootstrap\RegisterProviders`
* `\Illuminate\Foundation\Bootstrap\BootProviders`

这些 Bootstrap 类的 bootstrap 方法

* `\Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables`
```php
     /** 
      * Bootstrap the given application. 
      * 
      * @param  \Illuminate\Contracts\Foundation\Application  $app 
      * @return void 
      */ 
     public function bootstrap(Application $app) 
     { 
         if ($app->configurationIsCached()) { 
             return; 
         } 
  
         $this->checkForSpecificEnvironmentFile($app); 
  
         try { 
             (new Dotenv($app->environmentPath(), $app->environmentFile()))->load(); 
         } catch (InvalidPathException $e) { 
             // 
         } catch (InvalidFileException $e) { 
             echo 'The environment file is invalid: '.$e->getMessage(); 
             die(1); 
         } 
     } 
```

* `\Illuminate\Foundation\Bootstrap\LoadConfiguration`
```php
     /** 
      * Bootstrap the given application. 
      * 
      * @param  \Illuminate\Contracts\Foundation\Application  $app 
      * @return void 
      */ 
     public function bootstrap(Application $app) 
     { 
         $items = []; 
  
         // First we will see if we have a cache configuration file. If we do, we'll load 
         // the configuration items from that file so that it is very quick. Otherwise 
         // we will need to spin through every configuration file and load them all. 
         if (file_exists($cached = $app->getCachedConfigPath())) { 
             $items = require $cached; 
  
             $loadedFromCache = true; 
         } 
  
         // Next we will spin through all of the configuration files in the configuration 
         // directory and load each one into the repository. This will make all of the 
         // options available to the developer for use in various parts of this app. 
         $app->instance('config', $config = new Repository($items)); 
  
         if (! isset($loadedFromCache)) { 
             $this->loadConfigurationFiles($app, $config); 
         } 
  
         // Finally, we will set the application's environment based on the configuration 
         // values that were loaded. We will pass a callback which will be used to get 
         // the environment in a web context where an "--env" switch is not present. 
         $app->detectEnvironment(function () use ($config) { 
             return $config->get('app.env', 'production'); 
         }); 
  
         date_default_timezone_set($config->get('app.timezone', 'UTC')); 
  
         mb_internal_encoding('UTF-8'); 
     } 
```

* `\Illuminate\Foundation\Bootstrap\HandleExceptions`
```php

     /** 
      * Bootstrap the given application. 
      * 
      * @param  \Illuminate\Contracts\Foundation\Application  $app 
      * @return void 
      */ 
     public function bootstrap(Application $app) 
     { 
         $this->app = $app; 
  
         error_reporting(-1); 
  
         set_error_handler([$this, 'handleError']); 
  
         set_exception_handler([$this, 'handleException']); 
  
         register_shutdown_function([$this, 'handleShutdown']); 
  
         if (! $app->environment('testing')) { 
             ini_set('display_errors', 'Off'); 
         } 
     }
```

* `\Illuminate\Foundation\Bootstrap\RegisterFacades`
```php
     /** 
      * Bootstrap the given application. 
      * 
      * @param  \Illuminate\Contracts\Foundation\Application  $app 
      * @return void 
      */ 
     public function bootstrap(Application $app) 
     { 
         Facade::clearResolvedInstances(); 
  
         Facade::setFacadeApplication($app); 
  
         AliasLoader::getInstance(array_merge( 
             $app->make('config')->get('app.aliases', []), 
             $app->make(PackageManifest::class)->aliases() 
         ))->register(); 
     } 
```

* `\Illuminate\Foundation\Bootstrap\RegisterProviders`
```php

     /** 
      * Bootstrap the given application. 
      * 
      * @param  \Illuminate\Contracts\Foundation\Application  $app 
      * @return void 
      */ 
     public function bootstrap(Application $app) 
     { 
         $app->registerConfiguredProviders(); 
     } 
```

* `\Illuminate\Foundation\Bootstrap\BootProviders`
```php

     /** 
      * Bootstrap the given application. 
      * 
      * @param  \Illuminate\Contracts\Foundation\Application  $app 
      * @return void 
      */ 
     public function bootstrap(Application $app) 
     { 
         $app->boot(); 
     } 
```

上面列出的清单中，分别作用为


| 类 | 作用 |
| --- | --- |
| `\Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables` | 加载环境变量 |
| `\Illuminate\Foundation\Bootstrap\LoadConfiguration` | 加载config |
| `\Illuminate\Foundation\Bootstrap\HandleExceptions` | 错误处理者 |
| `\Illuminate\Foundation\Bootstrap\RegisterFacades` | 注册门面类 |
| `\Illuminate\Foundation\Bootstrap\RegisterProviders` | 注册服务提供者|
| `\Illuminate\Foundation\Bootstrap\BootProviders` | 启动服务提供者 |


其中最后两项在 laravel 请求的生命周期中是至关重要的，我们将在后续文章中重点讲解。 
 
在 `bootstrap` 阶段结束后，`Kernel::sendRequestThroughRouter` 后面带 `pipeline` 关键字的代码就是管道。
> 关于管道请查阅 [05. Pipeline 解析](https://github.com/xiaohuilam/laravel/issues/5)

```php
         return (new Pipeline($this->app)) 
                     ->send($request) 
                     ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware) 
                     ->then($this->dispatchToRouter()); 
```

在进入管道前， 调用了 `dispatchToRouter` 返回一个闭包对象
```php

     /** 
      * Get the route dispatcher callback. 
      * 
      * @return \Closure 
      */ 
     protected function dispatchToRouter() 
     { 
         return function ($request) { 
             $this->app->instance('request', $request); 
  
             return $this->router->dispatch($request); 
         }; 
     } 
```

匹配路由的逻辑清晰可见
```php

     /** 
      * Dispatch the request to the application. 
      * 
      * @param  \Illuminate\Http\Request  $request 
      * @return \Illuminate\Http\Response|\Illuminate\Http\JsonResponse 
      */ 
     public function dispatch(Request $request) 
     { 
         $this->currentRequest = $request; 
  
         return $this->dispatchToRoute($request); 
     } 
  
     /** 
      * Dispatch the request to a route and return the response. 
      * 
      * @param  \Illuminate\Http\Request  $request 
      * @return mixed 
      */ 
     public function dispatchToRoute(Request $request) 
     { 
         return $this->runRoute($request, $this->findRoute($request)); 
     } 
  
     /** 
      * Find the route matching a given request. 
      * 
      * @param  \Illuminate\Http\Request  $request 
      * @return \Illuminate\Routing\Route 
      */ 
     protected function findRoute($request) 
     { 
         $this->current = $route = $this->routes->match($request); 
  
         $this->container->instance(Route::class, $route); 
  
         return $route; 
     } 
  
     /** 
      * Return the response for the given route. 
      * 
      * @param  \Illuminate\Http\Request  $request 
      * @param  \Illuminate\Routing\Route  $route 
      * @return mixed 
      */ 
     protected function runRoute(Request $request, Route $route) 
     { 
         $request->setRouteResolver(function () use ($route) { 
             return $route; 
         }); 
  
         $this->events->dispatch(new Events\RouteMatched($route, $request)); 
  
         return $this->prepareResponse($request, 
             $this->runRouteWithinStack($route, $request) 
         ); 
     } 
  
     /** 
      * Run the given route within a Stack "onion" instance. 
      * 
      * @param  \Illuminate\Routing\Route  $route 
      * @param  \Illuminate\Http\Request  $request 
      * @return mixed 
      */ 
     protected function runRouteWithinStack(Route $route, Request $request) 
     { 
         $shouldSkipMiddleware = $this->container->bound('middleware.disable') && 
                                 $this->container->make('middleware.disable') === true; 
  
         $middleware = $shouldSkipMiddleware ? [] : $this->gatherRouteMiddleware($route); 
  
         return (new Pipeline($this->container)) 
                         ->send($request) 
                         ->through($middleware) 
                         ->then(function ($request) use ($route) { 
                             return $this->prepareResponse( 
                                 $request, $route->run() 
                             ); 
                         }); 
     } 
```

> 如果一路抽丝剥茧，我们便能找到 Router 调用 controller 的逻辑了。 请见[06. RouteServiceProvider 详解](https://github.com/xiaohuilam/laravel/issues/6) 最后段落。
