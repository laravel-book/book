# HTTP 入口解析

## public/index.php 入口

在 `public/index.php` 中第 24 行
https://github.com/xiaohuilam/laravel/blob/7028b17ed8bf35ee2f1269c0f9c985b411cb4469/public/index.php#L24


逻辑为，引入 `../vendor/autoload.php`，即 `composer` 产生的 `vendor/autoload.php`。 

接着第 38 行
https://github.com/xiaohuilam/laravel/blob/7028b17ed8bf35ee2f1269c0f9c985b411cb4469/public/index.php#L38

这里面做的事情不少，具体的我们看 `bootstrap/app.php` 的代码。

## bootstrap/app.php 实例化容器

在第 [14-16行]https://github.com/xiaohuilam/laravel/blob/7028b17ed8bf35ee2f1269c0f9c985b411cb4469/bootstrap/app.php#L14-L16


这是 `Laravel` 框架内为数不多的直接 `new` 出来的对象，也就是框架的应用对象， 
此对象继承于 `Illuminate\Container\Container` （容器），
所以， `Laravel` 框架中的容器对象一般就是指的 `Illuminate\Foundation\Application` 对象（因在实际运行过程中，不会直接 `new Illuminate\Container\Container` 的）。  

在第 [29-42行]
https://github.com/xiaohuilam/laravel/blob/7028b17ed8bf35ee2f1269c0f9c985b411cb4469/bootstrap/app.php#L29-L42

这里的singleton是 `public/index.php` 中，用 `Illuminate\Contracts\Http\Kernel` 能注入后取出 `App\Http\Kernel` 的关键。
> 关于 `singleton()` 的解析，请见 [10. 容器的 singleton 和 bind 的实现](https://github.com/xiaohuilam/laravel/issues/10)

第 55行 https://github.com/xiaohuilam/laravel/blob/7028b17ed8bf35ee2f1269c0f9c985b411cb4469/bootstrap/app.php#L55
 
在前面 `public/index.php` 中的
https://github.com/xiaohuilam/laravel/blob/7028b17ed8bf35ee2f1269c0f9c985b411cb4469/public/index.php#L38 拿到了 `return` 的 `$app` 。


## public/index.php 构建 + 处理

在 `public/index.php` 第 52 行
https://github.com/xiaohuilam/laravel/blob/7028b17ed8bf35ee2f1269c0f9c985b411cb4469/public/index.php#L52

这里调用了 `Illuminate\Foundation\Application` 的 make 方法（具体解析请见本篇结尾的链接），获得了 http 处理器 ( `$kernel` )

然后第 54-56 行
https://github.com/xiaohuilam/laravel/blob/7028b17ed8bf35ee2f1269c0f9c985b411cb4469/public/index.php#L54-L56
先用 `Illuminate\Http\Request::capture()` 抓出一个 `Illuminate\Http\Request` HTTP请求对象

## Request::capture 请求捕获

`Illuminate\Http\Request::capture()` 的代码为
https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Http/Request.php#L55-L60

其中调用了 `Symfony\Component\HttpFoundation\Request::createFromGlobals()`
https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/symfony/http-foundation/Request.php#L279-L291
其作用是讲 HTTP 请求过来的参数，按照 `key => value` 格式生成 `Symfony\Component\HttpFoundation\ParameterBag` 对象。
然后将包含 `ParameterBad` 的 `$request` 对象交给 `Illuminate\Http\Request::createFromBase()`
https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Http/Request.php#L404-L422

`(new static)->duplicate()` 最终执行了
https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/symfony/http-foundation/Request.php#L427-L469
其将重要的 http 请求头、请求参数、请求文件、cookie 等信息赋给相应属性。

在后面的
https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Http/Request.php#L419
`Request::getInputSource()` 的代码是
https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Http/Request.php#L351-L358

作用是调用 `$request->input()`
> GET、HEAD 请求时，能取到 GET 参数，
> POST 时，能取到 POST 参数。

至此，`Request::capture()` 就执行完了。

## public/index.php 运行逻辑

然后交给 `App\Http\Kernel` 的 `handle` 方法进行处理，获得 $response (`Illuminate\Http\Request` 对象)。
> Laravel 框架的设计绝大可扩展部分（如服务提供者、中间件）都是在这个阶段处理和触发的，
> 所以我单独把 [02. HTTP Kernel Handle 解析](https://github.com/xiaohuilam/laravel/issues/2) 拆出去了。

最后，在第 58-60行
https://github.com/xiaohuilam/laravel/blob/7028b17ed8bf35ee2f1269c0f9c985b411cb4469/public/index.php#L58-L60

调用 Illuminate\Http\Response 的 send 方法，将响应的状态码/头/内容返回给客户端(浏览器)，具体过程可见本篇末尾的引用。
 
 
 
---
 * `Illuminate\Foundation\Application` 的 `singleton`、`bind` 方法和 `make` 解析请见 [10. 容器的 singleton 和 bind 的实现](https://github.com/xiaohuilam/laravel/issues/10)
 * `App\Http\Kernel` 的 handle 方法解析请见 [#02. Kernel Handle解析](https://github.com/xiaohuilam/laravel/issues/2)
 * `App\Http\Kernel` 的 terminate 方法解析请见 [TODO]
