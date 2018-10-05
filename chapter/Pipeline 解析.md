> Laravel 的管道机制，是其中间件 (middleware) 得以被序列执行的基础

# 管道机制
在开始之前我们先帖一段管理的应用代码，位于 [02. HTTP Kernel Handle解析](https://github.com/xiaohuilam/laravel/issues/2#L148)
[laravel/vendor/laravel/framework/src/Illuminate/Foundation/Http/Kernel.php Lines 148 to 151 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/Http/Kernel.php#L148-L151)
```php
         return (new Pipeline($this->app)) 
                     ->send($request) 
                     ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware) 
                     ->then($this->dispatchToRouter()); 
```
 
语义化得出的分析为 
携带 `send` 请求对象，经过中间件的处理，然后进入路由。 
 
## 代码解析

1. send 携带方法
```php
     /** 
      * Set the object being sent through the pipeline. 
      * 
      * @param  mixed  $passable 
      * @return $this 
      */ 
     public function send($passable) 
     { 
         $this->passable = $passable; 
  
         return $this; 
     } 
```
好像没啥出奇的，就是传入并储存一个对象，返回 `$this`

2. through 经过方法
```php
     /** 
      * Set the array of pipes. 
      * 
      * @param  array|mixed  $pipes 
      * @return $this 
      */ 
     public function through($pipes) 
     { 
         $this->pipes = is_array($pipes) ? $pipes : func_get_args(); 
  
         return $this; 
     } 
```
好像也没啥关键的实现，传入“被管道”的集合，返回 `$this`

3. then 异步方法
```php
     /** 
      * Run the pipeline with a final destination callback. 
      * 
      * @param  \Closure  $destination 
      * @return mixed 
      */ 
     public function then(Closure $destination) 
     { 
         $pipeline = array_reduce( 
             array_reverse($this->pipes), $this->carry(), $this->prepareDestination($destination) 
         ); 
  
         return $pipeline($this->passable); 
     } 
```

之前的都只是做登记一下就GG的作用，这个 `then` 才是根本大法，
使用 `array_reduce` 处理得到 `$pipeline`。这个 `array_reduce` 是何方神圣呢？我们查文档得到:
Iteratively reduce the array to a single value using a callback function（中文意思就是 使用回调函数，迭代地将数组减少为单个值）。 

但在走到 array_reduce 之前，调用了 carry 方法  (在 5.3 及以前版本，carry 方法其实为 `getSlice` 方法，而过程几乎一样 [来源](https://github.com/laravel/framework/blob/5.3/src/Illuminate/Pipeline/Pipeline.php#L96))，其内部逻辑为：

```php

     /** 
      * Get a Closure that represents a slice of the application onion. 
      * 
      * @return \Closure 
      */ 
     protected function carry() 
     { 
         return function ($stack, $pipe) { 
             return function ($passable) use ($stack, $pipe) { 
                 if (is_callable($pipe)) { 
                     // If the pipe is an instance of a Closure, we will just call it directly but 
                     // otherwise we'll resolve the pipes out of the container and call it with 
                     // the appropriate method and arguments, returning the results back out. 
                     return $pipe($passable, $stack); 
                 } elseif (! is_object($pipe)) { 
                     list($name, $parameters) = $this->parsePipeString($pipe); 
  
                     // If the pipe is a string we will parse the string and resolve the class out 
                     // of the dependency injection container. We can then build a callable and 
                     // execute the pipe function giving in the parameters that are required. 
                     $pipe = $this->getContainer()->make($name); 
  
                     $parameters = array_merge([$passable, $stack], $parameters); 
                 } else { 
                     // If the pipe is already an object we'll just make a callable and pass it to 
                     // the pipe as-is. There is no need to do any extra parsing and formatting 
                     // since the object we're given was already a fully instantiated object. 
                     $parameters = [$passable, $stack]; 
                 } 
  
                 $response = method_exists($pipe, $this->method) 
                                 ? $pipe->{$this->method}(...$parameters) 
                                 : $pipe(...$parameters); 
  
                 return $response instanceof Responsable 
                             ? $response->toResponse($this->container->make(Request::class)) 
                             : $response; 
             }; 
         }; 
     } 
```

 `Laravel` 的这个 carry 方法返回的是一个闭包，执行返回的闭包所返回的又是一个闭包，只不过前一个返回是 array_reduce 批量执行时拿到的，后面是处理原始对象（请求）时，处理中间件时候拿到的。
* 闭包（前一个）的第一个参数 `$stack` 为 null 或者闭包对象，第二个参数 `$pipe` 为从1开始的调用顺序。
*  闭包（前一个）的返回值，为管道中被执行的逻辑，也就是 middleware 的 handle。

具体 handle 代码我们举一个例子:
```php

     /** 
      * Handle an incoming request. 
      * 
      * @param  \Illuminate\Http\Request  $request 
      * @param  \Closure  $next 
      * @return mixed 
      * 
      * @throws \Illuminate\Session\TokenMismatchException 
      */ 
     public function handle($request, Closure $next) 
     { 
         if ( 
             $this->isReading($request) || 
             $this->runningUnitTests() || 
             $this->inExceptArray($request) || 
             $this->tokensMatch($request) 
         ) { 
             return tap($next($request), function ($response) use ($request) { 
                 if ($this->addHttpCookie) { 
                     $this->addCookieToResponse($request, $response); 
                 } 
             }); 
         } 
  
         throw new TokenMismatchException; 
     } 
```

其中的 `$next($request)` 表示执行下一个中间件, $next 指的就是下面这个闭包
```php
             return function ($passable) use ($stack, $pipe) { 
                 if (is_callable($pipe)) { 
                     // If the pipe is an instance of a Closure, we will just call it directly but 
                     // otherwise we'll resolve the pipes out of the container and call it with 
                     // the appropriate method and arguments, returning the results back out. 
                     return $pipe($passable, $stack); 
                 } elseif (! is_object($pipe)) { 
                     list($name, $parameters) = $this->parsePipeString($pipe); 
  
                     // If the pipe is a string we will parse the string and resolve the class out 
                     // of the dependency injection container. We can then build a callable and 
                     // execute the pipe function giving in the parameters that are required. 
                     $pipe = $this->getContainer()->make($name); 
  
                     $parameters = array_merge([$passable, $stack], $parameters); 
                 } else { 
                     // If the pipe is already an object we'll just make a callable and pass it to 
                     // the pipe as-is. There is no need to do any extra parsing and formatting 
                     // since the object we're given was already a fully instantiated object. 
                     $parameters = [$passable, $stack]; 
                 } 
  
                 $response = method_exists($pipe, $this->method) 
                                 ? $pipe->{$this->method}(...$parameters) 
                                 : $pipe(...$parameters); 
  
                 return $response instanceof Responsable 
                             ? $response->toResponse($this->container->make(Request::class)) 
                             : $response; 
             }; 
```

关于中间件执行的流程，是不是有眉目了？
![image](https://user-images.githubusercontent.com/6964962/46106844-6124d500-c20c-11e8-8c09-d2e63cd238eb.png)

等上面执行完成，Laravel 就要执行这个闭包了
[laravel/vendor/laravel/framework/src/Illuminate/Foundation/Http/Kernel.php Lines 166 to 178 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/Http/Kernel.php#L166-L178)
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

也就是执行路由的 dispatch 逻辑（包含被调用的逻辑一共 82 行代码）。
[laravel/vendor/laravel/framework/src/Illuminate/Routing/Router.php Lines 601 to 682 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Routing/Router.php#L601-L682)
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
```

你可能会因为这里又有中间件而诧异
[laravel/vendor/laravel/framework/src/Illuminate/Routing/Router.php Lines 674 to 681 in d081c91](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Routing/Router.php#L674-L681)
```php

         return (new Pipeline($this->container)) 
                         ->send($request) 
                         ->through($middleware) 
                         ->then(function ($request) use ($route) { 
                             return $this->prepareResponse( 
                                 $request, $route->run() 
                             ); 
                         }); 
```

其实这里的中间件和前面的不一样，前面的是在kernel 中注册的中间件。这里的中间件是在路由绑定中注册的中间件。是两部分。

于是乎，Laravel 鬼使神差的实现了中间件的管道执行机制。
 
> 接下来，请查看 [09. 容器的依赖注入机制](https://github.com/xiaohuilam/laravel/issues/9) 了解 `$route->run() ` 背后的故事。
