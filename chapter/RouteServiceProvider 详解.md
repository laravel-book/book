# RouteServiceProvider 详解
> 在 `RouteServiceProvider::boot` 阶段，所有路由都只是执行登记，匹配的逻辑是在 [HTTP Kernel Handle解析](HTTP Kernel Handle 解析.md) 的 dispatchToRouter 阶段。 
 
我们先找到 `RouteServiceProvider.php` 
 
在 [`config/app.php`](https://github.com/xiaohuilam/laravel/blob/6d9215c0a48aa68ef65d83c3ab8d1ba3c4e23d39/config/app.php#L161) 的 `providers` 中定义的 RouteServiceProvider 其实是:
```php
         App\Providers\RouteServiceProvider::class, 
```

而这个类其实继承自 [`Illuminate\Foundation\Support\Providers\RouteServiceProvider`](https://github.com/xiaohuilam/laravel/blob/6d9215c0a48aa68ef65d83c3ab8d1ba3c4e23d39/app/Providers/RouteServiceProvider.php#L6-L9)
```php
 use Illuminate\Foundation\Support\Providers\RouteServiceProvider as ServiceProvider; 
  
 class RouteServiceProvider extends ServiceProvider 
 { 
```

我们分析 [`Illuminate\Foundation\Support\Providers\RouteServiceProvider`](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/Support/Providers/RouteServiceProvider.php#L1-L104) 的代码

## register() 阶段
其 [`register`](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/Support/Providers/RouteServiceProvider.php#L81-L89) 方法是空的
```php
     /** 
      * Register the service provider. 
      * 
      * @return void 
      */ 
     public function register() 
     { 
         // 
     } 
```

## boot() 阶段
而在 [`boot`](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/Support/Providers/RouteServiceProvider.php#L24-L43) 阶段，几乎运行了 `RouteServiceProvider` 中的所有方法
```php
     /** 
      * Bootstrap any application services. 
      * 
      * @return void 
      */ 
     public function boot() 
     { 
         $this->setRootControllerNamespace(); 
  
         if ($this->app->routesAreCached()) { 
             $this->loadCachedRoutes(); 
         } else { 
             $this->loadRoutes(); 
  
             $this->app->booted(function () { 
                 $this->app['router']->getRoutes()->refreshNameLookups(); 
                 $this->app['router']->getRoutes()->refreshActionLookups(); 
             }); 
         } 
     } 
```

### 一. [setRootControllerNamespace()](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/Support/Providers/RouteServiceProvider.php#L45-L55)
```php
     /** 
      * Set the root controller namespace for the application. 
      * 
      * @return void 
      */ 
     protected function setRootControllerNamespace() 
     { 
         if (! is_null($this->namespace)) { 
             $this->app[UrlGenerator::class]->setRootControllerNamespace($this->namespace); 
         } 
     } 
```

### 二. 判断路由是否缓存过
[`Illuminate\Foundation\Application::routesAreCached`](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/Application.php#L895-L903)，
其实就是判断文件 `bootstrap/cache/routes.php` 是否存在
laravel/vendor/laravel/framework/src/Illuminate/Foundation/Application.php 
   
[Lines 895 to 903 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/Application.php#L895-L903)
```php
     /** 
      * Determine if the application routes are cached. 
      * 
      * @return bool 
      */ 
     public function routesAreCached() 
     { 
         return $this['files']->exists($this->getCachedRoutesPath()); 
     } 
```

[Lines 905 to 913 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/Application.php#L905-L913)
```php
     /** 
      * Get the path to the routes cache file. 
      * 
      * @return string 
      */ 
     public function getCachedRoutesPath() 
     { 
         return $this->bootstrapPath().'/cache/routes.php'; 
     } 
```

  存在则直接加载
[Lines 57 to 67 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/Support/Providers/RouteServiceProvider.php#L57-L67)
```php
     /** 
      * Load the cached routes for the application. 
      * 
      * @return void 
      */ 
     protected function loadCachedRoutes() 
     { 
         $this->app->booted(function () { 
             require $this->app->getCachedRoutesPath(); 
         }); 
     } 
```

### 三. 如果没有缓存，则遍历路由
[laravel/vendor/laravel/framework/src/Illuminate/Foundation/Support/Providers/RouteServiceProvider.php Lines 69 to 79 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/Support/Providers/RouteServiceProvider.php#L69-L79)
```php
     /** 
      * Load the application routes. 
      * 
      * @return void 
      */ 
     protected function loadRoutes() 
     { 
         if (method_exists($this, 'map')) { 
             $this->app->call([$this, 'map']); 
         } 
     } 
```

`map` 方法在这里
[laravel/app/Providers/RouteServiceProvider.php Lines 31 to 43 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/app/Providers/RouteServiceProvider.php#L31-L43)
```php
     /** 
      * Define the routes for the application. 
      * 
      * @return void 
      */ 
     public function map() 
     { 
         $this->mapApiRoutes(); 
  
         $this->mapWebRoutes(); 
  
         // 
     } 
```

[`mapApiRoutes` 和 `mapWebRoutes`](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/app/Providers/RouteServiceProvider.php#L45-L72)
```php

     /** 
      * Define the "web" routes for the application. 
      * 
      * These routes all receive session state, CSRF protection, etc. 
      * 
      * @return void 
      */ 
     protected function mapWebRoutes() 
     { 
         Route::middleware('web') 
              ->namespace($this->namespace) 
              ->group(base_path('routes/web.php')); 
     } 
  
     /** 
      * Define the "api" routes for the application. 
      * 
      * These routes are typically stateless. 
      * 
      * @return void 
      */ 
     protected function mapApiRoutes() 
     { 
         Route::prefix('api') 
              ->middleware('api') 
              ->namespace($this->namespace) 
              ->group(base_path('routes/api.php')); 
     } 
```

  这里的 `mapWebRoutes` 和 `mapApiRoutes` 是分别将 routes/web.php 和 routes/api.php 用 `Route` 门面类的 group 加载了一遍。 `Route::group` 实质运行到的是 `Illuminate\Routing\Router::group` (关于门面类的文章请见 [TODO])，代码为
  [laravel/vendor/laravel/framework/src/Illuminate/Routing/Router.php Lines 356 to 416 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Routing/Router.php#L356-L416)
```php
     /** 
      * Create a route group with shared attributes. 
      * 
      * @param  array  $attributes 
      * @param  \Closure|string  $routes 
      * @return void 
      */ 
     public function group(array $attributes, $routes) 
     { 
         $this->updateGroupStack($attributes); 
  
         // Once we have updated the group stack, we'll load the provided routes and 
         // merge in the group's attributes when the routes are created. After we 
         // have created the routes, we will pop the attributes off the stack. 
         $this->loadRoutes($routes); 
  
         array_pop($this->groupStack); 
     } 
  
     /** 
      * Update the group stack with the given attributes. 
      * 
      * @param  array  $attributes 
      * @return void 
      */ 
     protected function updateGroupStack(array $attributes) 
     { 
         if (! empty($this->groupStack)) { 
             $attributes = RouteGroup::merge($attributes, end($this->groupStack)); 
         } 
  
         $this->groupStack[] = $attributes; 
     } 
  
     /** 
      * Merge the given array with the last group stack. 
      * 
      * @param  array  $new 
      * @return array 
      */ 
     public function mergeWithLastGroup($new) 
     { 
         return RouteGroup::merge($new, end($this->groupStack)); 
     } 
  
     /** 
      * Load the provided routes. 
      * 
      * @param  \Closure|string  $routes 
      * @return void 
      */ 
     protected function loadRoutes($routes) 
     { 
         if ($routes instanceof Closure) { 
             $routes($this); 
         } else { 
             $router = $this; 
  
             require $routes; 
         } 
     } 
```

  上面的逻辑是一层层剥开 `Route::group` 去执行里面的 `Route::get` / `Route::post` ...
  [laravel/vendor/laravel/framework/src/Illuminate/Routing/Router.php Lines 133 to 214 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Routing/Router.php#L133-L214)
```php
      * Register a new GET route with the router. 
      * 
      * @param  string  $uri 
      * @param  \Closure|array|string|null  $action 
      * @return \Illuminate\Routing\Route 
      */ 
     public function get($uri, $action = null) 
     { 
         return $this->addRoute(['GET', 'HEAD'], $uri, $action); 
     } 
  
     /** 
      * Register a new POST route with the router. 
      * 
      * @param  string  $uri 
      * @param  \Closure|array|string|null  $action 
      * @return \Illuminate\Routing\Route 
      */ 
     public function post($uri, $action = null) 
     { 
         return $this->addRoute('POST', $uri, $action); 
     } 
  
     /** 
      * Register a new PUT route with the router. 
      * 
      * @param  string  $uri 
      * @param  \Closure|array|string|null  $action 
      * @return \Illuminate\Routing\Route 
      */ 
     public function put($uri, $action = null) 
     { 
         return $this->addRoute('PUT', $uri, $action); 
     } 
  
     /** 
      * Register a new PATCH route with the router. 
      * 
      * @param  string  $uri 
      * @param  \Closure|array|string|null  $action 
      * @return \Illuminate\Routing\Route 
      */ 
     public function patch($uri, $action = null) 
     { 
         return $this->addRoute('PATCH', $uri, $action); 
     } 
  
     /** 
      * Register a new DELETE route with the router. 
      * 
      * @param  string  $uri 
      * @param  \Closure|array|string|null  $action 
      * @return \Illuminate\Routing\Route 
      */ 
     public function delete($uri, $action = null) 
     { 
         return $this->addRoute('DELETE', $uri, $action); 
     } 
  
     /** 
      * Register a new OPTIONS route with the router. 
      * 
      * @param  string  $uri 
      * @param  \Closure|array|string|null  $action 
      * @return \Illuminate\Routing\Route 
      */ 
     public function options($uri, $action = null) 
     { 
         return $this->addRoute('OPTIONS', $uri, $action); 
     } 
  
     /** 
      * Register a new route responding to all verbs. 
      * 
      * @param  string  $uri 
      * @param  \Closure|array|string|null  $action 
      * @return \Illuminate\Routing\Route 
      */ 
     public function any($uri, $action = null) 
     { 
         return $this->addRoute(self::$verbs, $uri, $action); 
     } 
```

  不难发现，不管是 GET/POST... 还是 any，都只是调用了 [`addRoute`](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Routing/Router.php#L434-L445) 将这个路由的属性登记了一下：
```php
     /** 
      * Add a route to the underlying route collection. 
      * 
      * @param  array|string  $methods 
      * @param  string  $uri 
      * @param  \Closure|array|string|null  $action 
      * @return \Illuminate\Routing\Route 
      */ 
     public function addRoute($methods, $uri, $action) 
     { 
         return $this->routes->add($this->createRoute($methods, $uri, $action)); 
     } 
```

  `$this->routes` 是一个 `\Illuminate\Routing\RouteCollection` 的集合类，`add` 由这个方法生成的路由
  [laravel/vendor/laravel/framework/src/Illuminate/Routing/Router.php Lines 447 to 478 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Routing/Router.php#L447-L478)
```php

     /** 
      * Create a new route instance. 
      * 
      * @param  array|string  $methods 
      * @param  string  $uri 
      * @param  mixed  $action 
      * @return \Illuminate\Routing\Route 
      */ 
     protected function createRoute($methods, $uri, $action) 
     { 
         // If the route is routing to a controller we will parse the route action into 
         // an acceptable array format before registering it and creating this route 
         // instance itself. We need to build the Closure that will call this out. 
         if ($this->actionReferencesController($action)) { 
             $action = $this->convertToControllerAction($action); 
         } 
  
         $route = $this->newRoute( 
             $methods, $this->prefix($uri), $action 
         ); 
  
         // If we have groups that need to be merged, we will merge them now after this 
         // route has already been created and is ready to go. After we're done with 
         // the merge we will be ready to return the route back out to the caller. 
         if ($this->hasGroupStack()) { 
             $this->mergeGroupAttributesIntoRoute($route); 
         } 
  
         $this->addWhereClausesToRoute($route); 
  
         return $route; 
     } 
```

### 四. dispatch() -> runRoute() 阶段
路由登记完成后，就是 Kernel 触发管道层层剥洋葱调用中间件最后触发路由 dispatch （当然，这已经运行到 RouteServiceProvider 的外面了）。
> 剥洋葱的过程请查阅 [05. Pipeline 解析](https://github.com/xiaohuilam/laravel/issues/5)

  
[laravel/vendor/laravel/framework/src/Illuminate/Routing/Router.php Lines 660 to 682 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Routing/Router.php#L660-L682)
```php

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

第679行的 `$route->run` 是至关重要的，调用到了 controller  (本质其实是使用 [09. 容器的依赖注入机制](https://github.com/xiaohuilam/laravel/issues/9) 将路由方法所依赖参数解析出来，并运行)
[laravel/vendor/laravel/framework/src/Illuminate/Routing/Route.php Lines 158 to 176 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Routing/Route.php#L158-L176)
```php

     /** 
      * Run the route action and return the response. 
      * 
      * @return mixed 
      */ 
     public function run() 
     { 
         $this->container = $this->container ?: new Container; 
  
         try { 
             if ($this->isControllerAction()) { 
                 return $this->runController(); 
             } 
  
             return $this->runCallable(); 
         } catch (HttpResponseException $e) { 
             return $e->getResponse(); 
         } 
     } 
```
