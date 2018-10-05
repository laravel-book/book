> 随着前面 《02. Kernel Handle解析》 和 《03. ServiceProvider Register 解析》 的结束，我们接下来要分析的便是 启动服务提供者 这个步骤。

## 代码
```php

 <?php 
  
 namespace Illuminate\Foundation\Bootstrap; 
  
 use Illuminate\Contracts\Foundation\Application; 
  
 class BootProviders 
 { 
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
 } 
```

同 `ServiceProvider Register ` 一样的，在 BootProviders 中也是调用了 `Illuminate\Foundation\Application::boot()`
```php
     /** 
      * Boot the application's service providers. 
      * 
      * @return void 
      */ 
     public function boot() 
     { 
         if ($this->booted) { 
             return; 
         } 
  
         // Once the application has booted we will also fire some "booted" callbacks 
         // for any listeners that need to do work after this initial booting gets 
         // finished. This is useful when ordering the boot-up processes we run. 
         $this->fireAppCallbacks($this->bootingCallbacks); 
  
         array_walk($this->serviceProviders, function ($p) { 
             $this->bootProvider($p); 
         }); 
  
         $this->booted = true; 
  
         $this->fireAppCallbacks($this->bootedCallbacks); 
     } 
```

首先判断是否 boot 过。 
 
然后通过触发 $bootingCallbacks 钩子
```php
     /** 
      * Call the booting callbacks for the application. 
      * 
      * @param  array  $callbacks 
      * @return void 
      */ 
     protected function fireAppCallbacks(array $callbacks) 
     { 
         foreach ($callbacks as $callback) { 
             call_user_func($callback, $this); 
         } 
     } 
```
> $bootingCallbacks 是来自 [02. HTTP Kernel Handle解析](https://github.com/xiaohuilam/laravel/issues/2#L710) 登记的闭包，主要是服务于声明了 `$defer = true` 的服务提供者。
 
然后依次遍历 `$this->serviceProviders` 执行 `Illuminate\Foundation\Application::bootProvider()`
```php

     /** 
      * Boot the given service provider. 
      * 
      * @param  \Illuminate\Support\ServiceProvider  $provider 
      * @return mixed 
      */ 
     protected function bootProvider(ServiceProvider $provider) 
     { 
         if (method_exists($provider, 'boot')) { 
             return $this->call([$provider, 'boot']); 
         } 
     } 
```

其实就是运行一遍 ServiceProvider 的 boot 方法 

接着将 `Illuminate\Foundation\Application::$boot` 设置为 true 
 
最后触发 bootedCallbacks 钩子
