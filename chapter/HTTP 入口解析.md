# HTTP 入口解析

## public/index.php 入口

在 `public/index.php` 中第 24 行
```
 require __DIR__.'/../vendor/autoload.php'; 
```


逻辑为，引入 `../vendor/autoload.php`，即 `composer` 产生的 `vendor/autoload.php`。 

接着第 38 行
```
 $app = require_once __DIR__.'/../bootstrap/app.php'; 
```

这里面做的事情不少，具体的我们看 `bootstrap/app.php` 的代码。

## bootstrap/app.php 实例化容器

在第
```
 $app = new Illuminate\Foundation\Application( 
     realpath(__DIR__.'/../') 
 );
 ```


这是 `Laravel` 框架内为数不多的直接 `new` 出来的对象，也就是框架的应用对象， 
此对象继承于 `Illuminate\Container\Container` （容器），
所以， `Laravel` 框架中的容器对象一般就是指的 `Illuminate\Foundation\Application` 对象（因在实际运行过程中，不会直接 `new Illuminate\Container\Container` 的）。  

在第 [29-42行]
```
 $app->singleton( 
     Illuminate\Contracts\Http\Kernel::class, 
     App\Http\Kernel::class 
 ); 
  
 $app->singleton( 
     Illuminate\Contracts\Console\Kernel::class, 
     App\Console\Kernel::class 
 ); 
  
 $app->singleton( 
     Illuminate\Contracts\Debug\ExceptionHandler::class, 
     App\Exceptions\Handler::class 
 ); 
 ```
 
这里的singleton是 `public/index.php` 中，用 `Illuminate\Contracts\Http\Kernel` 能注入后取出 `App\Http\Kernel` 的关键。
> 关于 `singleton()` 的解析，请见 [10. 容器的 singleton 和 bind 的实现](https://github.com/xiaohuilam/laravel/issues/10)

第 55行 
```
 return $app; 
```
 
在前面 `public/index.php` 中的
```
 $app = require_once __DIR__.'/../bootstrap/app.php'; 
```
拿到了 `return` 的 `$app` 。


## public/index.php 构建 + 处理

在 `public/index.php` 第 52 行
https://github.com/xiaohuilam/laravel/blob/7028b17ed8bf35ee2f1269c0f9c985b411cb4469/public/index.php#L52

这里调用了 `Illuminate\Foundation\Application` 的 make 方法（具体解析请见本篇结尾的链接），获得了 http 处理器 ( `$kernel` )

然后第 54-56 行
```
 $kernel = $app->make(Illuminate\Contracts\Http\Kernel::class); 
```
先用 `Illuminate\Http\Request::capture()` 抓出一个 `Illuminate\Http\Request` HTTP请求对象

## Request::capture 请求捕获

`Illuminate\Http\Request::capture()` 的代码为
```
 $response = $kernel->handle( 
     $request = Illuminate\Http\Request::capture() 
 ); 
```
 
其中调用了 `Symfony\Component\HttpFoundation\Request::createFromGlobals()`
```
     public static function createFromGlobals() 
     { 
         $request = self::createRequestFromFactory($_GET, $_POST, array(), $_COOKIE, $_FILES, $_SERVER); 
  
         if (0 === strpos($request->headers->get('CONTENT_TYPE'), 'application/x-www-form-urlencoded') 
             && \in_array(strtoupper($request->server->get('REQUEST_METHOD', 'GET')), array('PUT', 'DELETE', 'PATCH')) 
         ) { 
             parse_str($request->getContent(), $data); 
             $request->request = new ParameterBag($data); 
         } 
  
         return $request; 
     } 
```

其作用是讲 HTTP 请求过来的参数，按照 `key => value` 格式生成 `Symfony\Component\HttpFoundation\ParameterBag` 对象。
然后将包含 `ParameterBad` 的 `$request` 对象交给 `Illuminate\Http\Request::createFromBase()`
```
     public static function createFromBase(SymfonyRequest $request) 
     { 
         if ($request instanceof static) { 
             return $request; 
         } 
  
         $content = $request->content; 
  
         $request = (new static)->duplicate( 
             $request->query->all(), $request->request->all(), $request->attributes->all(), 
             $request->cookies->all(), $request->files->all(), $request->server->all() 
         ); 
  
         $request->content = $content; 
  
         $request->request = $request->getInputSource(); 
  
         return $request; 
     } 
```

`(new static)->duplicate()` 最终执行了
```
     public function duplicate(array $query = null, array $request = null, array $attributes = null, array $cookies = null, array $files = null, array $server = null) 
     { 
         $dup = clone $this; 
         if (null !== $query) { 
             $dup->query = new ParameterBag($query); 
         } 
         if (null !== $request) { 
             $dup->request = new ParameterBag($request); 
         } 
         if (null !== $attributes) { 
             $dup->attributes = new ParameterBag($attributes); 
         } 
         if (null !== $cookies) { 
             $dup->cookies = new ParameterBag($cookies); 
         } 
         if (null !== $files) { 
             $dup->files = new FileBag($files); 
         } 
         if (null !== $server) { 
             $dup->server = new ServerBag($server); 
             $dup->headers = new HeaderBag($dup->server->getHeaders()); 
         } 
         $dup->languages = null; 
         $dup->charsets = null; 
         $dup->encodings = null; 
         $dup->acceptableContentTypes = null; 
         $dup->pathInfo = null; 
         $dup->requestUri = null; 
         $dup->baseUrl = null; 
         $dup->basePath = null; 
         $dup->method = null; 
         $dup->format = null; 
  
         if (!$dup->get('_format') && $this->get('_format')) { 
             $dup->attributes->set('_format', $this->get('_format')); 
         } 
  
         if (!$dup->getRequestFormat(null)) { 
             $dup->setRequestFormat($this->getRequestFormat(null)); 
         } 
  
         return $dup; 
     } 
```

其将重要的 http 请求头、请求参数、请求文件、cookie 等信息赋给相应属性。

在后面的
```
         $request->request = $request->getInputSource(); 
```

`Request::getInputSource()` 的代码是
```
     protected function getInputSource() 
     { 
         if ($this->isJson()) { 
             return $this->json(); 
         } 
  
         return in_array($this->getRealMethod(), ['GET', 'HEAD']) ? $this->query : $this->request; 
     } 
```

作用是调用 `$request->input()`
> GET、HEAD 请求时，能取到 GET 参数，
> POST 时，能取到 POST 参数。

至此，`Request::capture()` 就执行完了。

## public/index.php 运行逻辑

然后交给 `App\Http\Kernel` 的 `handle` 方法进行处理，获得 $response (`Illuminate\Http\Request` 对象)。
> Laravel 框架的设计绝大可扩展部分（如服务提供者、中间件）都是在这个阶段处理和触发的，
> 所以我单独把 [02. HTTP Kernel Handle 解析](https://github.com/xiaohuilam/laravel/issues/2) 拆出去了。

最后，在第 58-60行
```
 $response->send(); 
  
 $kernel->terminate($request, $response); 
```

调用 Illuminate\Http\Response 的 send 方法，将响应的状态码/头/内容返回给客户端(浏览器)，具体过程可见本篇末尾的引用。
 
 
 
---
 * `Illuminate\Foundation\Application` 的 `singleton`、`bind` 方法和 `make` 解析请见 [10. 容器的 singleton 和 bind 的实现](https://github.com/xiaohuilam/laravel/issues/10)
 * `App\Http\Kernel` 的 handle 方法解析请见 [#02. Kernel Handle解析](https://github.com/xiaohuilam/laravel/issues/2)
 * `App\Http\Kernel` 的 terminate 方法解析请见 [TODO]
