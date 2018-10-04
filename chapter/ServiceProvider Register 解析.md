# ServiceProvider Register 解析

> 前面 [《02. Kernel Handle解析》](https://github.com/xiaohuilam/laravel/issues/2) 结尾，我们留下了 `注册服务提供者` 和 `启动服务提供者` 是两个比较重要的步骤的悬念，接下来两篇文章，我们来分别解析这两大流程。

## 代码
```php
 <?php 
  
 namespace Illuminate\Foundation\Bootstrap; 
  
 use Illuminate\Contracts\Foundation\Application; 
  
 class RegisterProviders 
 { 
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
 } 
```

### 解析
整个 RegisterProviders 都是粮衣，其没有任何功能功能代码，而是调用到了 `Illuminate\Foundation\ Application` 的 `registerConfiguredProviders`
> 注意运行时实际并没有调用 Illuminate\Contracts\Foundation\Application 的 registerConfiguredProviders，而是调用了其具体实现类 Illuminate\Foundation\ Application 的 registerConfiguredProviders。不要被上面代码中的注解表象所迷惑

```php
     /** 
      * Register all of the configured providers. 
      * 
      * @return void 
      */ 
     public function registerConfiguredProviders() 
     { 
         $providers = Collection::make($this->config['app.providers']) 
                         ->partition(function ($provider) { 
                             return Str::startsWith($provider, 'Illuminate\\'); 
                         }); 
  
         $providers->splice(1, 0, [$this->make(PackageManifest::class)->providers()]); 
  
         (new ProviderRepository($this, new Filesystem, $this->getCachedServicesPath())) 
                     ->load($providers->collapse()->toArray()); 
     } 
```

上面第 540-543 行是将 [`config/app.php` 中配置的 `providers`](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/config/app.php#L122-L163)
```php
     'providers' => [ 
  
         /* 
          * Laravel Framework Service Providers... 
          */ 
         Illuminate\Auth\AuthServiceProvider::class, 
         Illuminate\Broadcasting\BroadcastServiceProvider::class, 
         Illuminate\Bus\BusServiceProvider::class, 
         Illuminate\Cache\CacheServiceProvider::class, 
         Illuminate\Foundation\Providers\ConsoleSupportServiceProvider::class, 
         Illuminate\Cookie\CookieServiceProvider::class, 
         Illuminate\Database\DatabaseServiceProvider::class, 
         Illuminate\Encryption\EncryptionServiceProvider::class, 
         Illuminate\Filesystem\FilesystemServiceProvider::class, 
         Illuminate\Foundation\Providers\FoundationServiceProvider::class, 
         Illuminate\Hashing\HashServiceProvider::class, 
         Illuminate\Mail\MailServiceProvider::class, 
         Illuminate\Notifications\NotificationServiceProvider::class, 
         Illuminate\Pagination\PaginationServiceProvider::class, 
         Illuminate\Pipeline\PipelineServiceProvider::class, 
         Illuminate\Queue\QueueServiceProvider::class, 
         Illuminate\Redis\RedisServiceProvider::class, 
         Illuminate\Auth\Passwords\PasswordResetServiceProvider::class, 
         Illuminate\Session\SessionServiceProvider::class, 
         Illuminate\Translation\TranslationServiceProvider::class, 
         Illuminate\Validation\ValidationServiceProvider::class, 
         Illuminate\View\ViewServiceProvider::class, 
  
         /* 
          * Package Service Providers... 
          */ 
  
         /* 
          * Application Service Providers... 
          */ 
         App\Providers\AppServiceProvider::class, 
         App\Providers\AuthServiceProvider::class, 
         // App\Providers\BroadcastServiceProvider::class, 
         App\Providers\EventServiceProvider::class, 
         App\Providers\RouteServiceProvider::class, 
  
     ],
```
读取出来，判断是否为 `Illuminate\` 开头，归为两组：
```php
Illuminate\Support\Collection {#2825
     all: [
       Illuminate\Support\Collection {#2822
         all: [
            "Illuminate\Auth\AuthServiceProvider",
            "Illuminate\Broadcasting\BroadcastServiceProvider",
            "Illuminate\Bus\BusServiceProvider",
            "Illuminate\Cache\CacheServiceProvider",
            "Illuminate\Foundation\Providers\ConsoleSupportServiceProvider",
            "Illuminate\Cookie\CookieServiceProvider",
            "Illuminate\Database\DatabaseServiceProvider",
            "Illuminate\Encryption\EncryptionServiceProvider",
            "Illuminate\Filesystem\FilesystemServiceProvider",
            "Illuminate\Foundation\Providers\FoundationServiceProvider",
            "Illuminate\Hashing\HashServiceProvider",
            "Illuminate\Mail\MailServiceProvider",
            "Illuminate\Notifications\NotificationServiceProvider",
            "Illuminate\Pagination\PaginationServiceProvider",
            "Illuminate\Pipeline\PipelineServiceProvider",
            "Illuminate\Queue\QueueServiceProvider",
            "Illuminate\Redis\RedisServiceProvider",
            "Illuminate\Auth\Passwords\PasswordResetServiceProvider",
            "Illuminate\Session\SessionServiceProvider",
            "Illuminate\Translation\TranslationServiceProvider",
            "Illuminate\Validation\ValidationServiceProvider",
            "Illuminate\View\ViewServiceProvider",
         ],
       },
       Illuminate\Support\Collection {#2823
         all: [
            "22" => "App\Providers\AppServiceProvider",
            "22" => "App\Providers\AuthServiceProvider",
            "22" => "App\Providers\EventServiceProvider",
            "22" => "App\Providers\RouteServiceProvider",
         ],
       },
     ],
   }
```

接着 `$providers->splice(1, 0, [$this->make(PackageManifest::class)->providers()])` 这一行会得到如下结果
```php
Illuminate\Support\Collection {#33
  #items: array:3 [
    0 => Illuminate\Support\Collection {#21
      #items: array:22 [
        0 => "Illuminate\Auth\AuthServiceProvider"
        1 => "Illuminate\Broadcasting\BroadcastServiceProvider"
        2 => "Illuminate\Bus\BusServiceProvider"
        3 => "Illuminate\Cache\CacheServiceProvider"
        4 => "Illuminate\Foundation\Providers\ConsoleSupportServiceProvider"
        5 => "Illuminate\Cookie\CookieServiceProvider"
        6 => "Illuminate\Database\DatabaseServiceProvider"
        7 => "Illuminate\Encryption\EncryptionServiceProvider"
        8 => "Illuminate\Filesystem\FilesystemServiceProvider"
        9 => "Illuminate\Foundation\Providers\FoundationServiceProvider"
        10 => "Illuminate\Hashing\HashServiceProvider"
        11 => "Illuminate\Mail\MailServiceProvider"
        12 => "Illuminate\Notifications\NotificationServiceProvider"
        13 => "Illuminate\Pagination\PaginationServiceProvider"
        14 => "Illuminate\Pipeline\PipelineServiceProvider"
        15 => "Illuminate\Queue\QueueServiceProvider"
        16 => "Illuminate\Redis\RedisServiceProvider"
        17 => "Illuminate\Auth\Passwords\PasswordResetServiceProvider"
        18 => "Illuminate\Session\SessionServiceProvider"
        19 => "Illuminate\Translation\TranslationServiceProvider"
        20 => "Illuminate\Validation\ValidationServiceProvider"
        21 => "Illuminate\View\ViewServiceProvider"
      ]
    }
    1 => array:5 [
      0 => "BeyondCode\DumpServer\DumpServerServiceProvider"
      1 => "Fideloper\Proxy\TrustedProxyServiceProvider"
      2 => "Laravel\Tinker\TinkerServiceProvider"
      3 => "Carbon\Laravel\ServiceProvider"
      4 => "NunoMaduro\Collision\Adapters\Laravel\CollisionServiceProvider"
    ]
    2 => Illuminate\Support\Collection {#31
      #items: array:4 [
        22 => "App\Providers\AppServiceProvider"
        23 => "App\Providers\AuthServiceProvider"
        24 => "App\Providers\EventServiceProvider"
        25 => "App\Providers\RouteServiceProvider"
      ]
    }
  ]
}
```

最后一句代码
```php
(new ProviderRepository($this, new Filesystem, $this->getCachedServicesPath())) 
                     ->load($providers->collapse()->toArray()); 
```
第一步是将加载缓存好的 `bootstrap/cache/services.php`
```php
     /** 
      * Get the path to the cached services.php file. 
      * 
      * @return string 
      */ 
     public function getCachedServicesPath() 
     { 
         return $this->bootstrapPath().'/cache/services.php'; 
     } 
```

其实就是执行 [`ProviderRepository::load()`](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/ProviderRepository.php#L47-L79) $providers-> collapse() 得到的数组 (这个数组是前面 `splice` 拆分后再并入的)。
```php
     /** 
      * Register the application service providers. 
      * 
      * @param  array  $providers 
      * @return void 
      */ 
     public function load(array $providers) 
     { 
         $manifest = $this->loadManifest(); 
  
         // First we will load the service manifest, which contains information on all 
         // service providers registered with the application and which services it 
         // provides. This is used to know which services are "deferred" loaders. 
         if ($this->shouldRecompile($manifest, $providers)) { 
             $manifest = $this->compileManifest($providers); 
         } 
  
         // Next, we will register events to load the providers for each of the events 
         // that it has requested. This allows the service provider to defer itself 
         // while still getting automatically loaded when a certain event occurs. 
         foreach ($manifest['when'] as $provider => $events) { 
             $this->registerLoadEvents($provider, $events); 
         } 
  
         // We will go ahead and register all of the eagerly loaded providers with the 
         // application so their services can be registered with the application as 
         // a provided service. Then we will set the deferred service list on it. 
         foreach ($manifest['eager'] as $provider) { 
             $this->app->register($provider); 
         } 
  
         $this->app->addDeferredServices($manifest['deferred']); 
     } 
```
 
 
这句 loadManifest 即为加载前面传入的 `bootstrap/cache/services.php`
```php
     /** 
      * Load the service provider manifest JSON file. 
      * 
      * @return array|null 
      */ 
     public function loadManifest() 
     { 
         // The service manifest is a file containing a JSON representation of every 
         // service provided by the application and whether its provider is using 
         // deferred loading or should be eagerly loaded on each request to us. 
         if ($this->files->exists($this->manifestPath)) { 
             $manifest = $this->files->getRequire($this->manifestPath); 
  
             if ($manifest) { 
                 return array_merge(['when' => []], $manifest); 
             } 
         } 
     } 
```

第60行调用到的的 `shouldRecompile` 逻辑为判断是否 `provider` 不符合，代码
```php
     /** 
      * Determine if the manifest should be compiled. 
      * 
      * @param  array  $manifest 
      * @param  array  $providers 
      * @return bool 
      */ 
     public function shouldRecompile($manifest, $providers) 
     { 
         return is_null($manifest) || $manifest['providers'] != $providers; 
     } 
```

如果需要重新编译此服务提供者缓存的 `bootstrap/cache/services.php` ，把数据 `$manifest` 提取到 `bootstrap/cache/services.php` ，具体过程为：
```php
     /** 
      * Compile the application service manifest file. 
      * 
      * @param  array  $providers 
      * @return array 
      */ 
     protected function compileManifest($providers) 
     { 
         // The service manifest should contain a list of all of the providers for 
         // the application so we can compare it on each request to the service 
         // and determine if the manifest should be recompiled or is current. 
         $manifest = $this->freshManifest($providers); 
  
         foreach ($providers as $provider) { 
             $instance = $this->createProvider($provider); 
  
             // When recompiling the service manifest, we will spin through each of the 
             // providers and check if it's a deferred provider or not. If so we'll 
             // add it's provided services to the manifest and note the provider. 
             if ($instance->isDeferred()) { 
                 foreach ($instance->provides() as $service) { 
                     $manifest['deferred'][$service] = $provider; 
                 } 
  
                 $manifest['when'][$provider] = $instance->when(); 
             } 
  
             // If the service providers are not deferred, we will simply add it to an 
             // array of eagerly loaded providers that will get registered on every 
             // request to this application instead of "lazy" loading every time. 
             else { 
                 $manifest['eager'][] = $provider; 
             } 
         } 
  
         return $this->writeManifest($manifest); 
     } 
```

```php
     /** 
      * Write the service manifest file to disk. 
      * 
      * @param  array  $manifest 
      * @return array 
      * 
      * @throws \Exception 
      */ 
     public function writeManifest($manifest) 
     { 
         if (! is_writable(dirname($this->manifestPath))) { 
             throw new Exception('The bootstrap/cache directory must be present and writable.'); 
         } 
  
         $this->files->put( 
             $this->manifestPath, '<?php return '.var_export($manifest, true).';' 
         ); 
  
         return array_merge(['when' => []], $manifest); 
     } 
```

到这时，ProviderRepository::load() 的服务提供者缓存的判断和执行流程就结束了。我们回到 ProviderRepository::load() 的后面流程，接下来就是
```php
         // Next, we will register events to load the providers for each of the events 
         // that it has requested. This allows the service provider to defer itself 
         // while still getting automatically loaded when a certain event occurs. 
         foreach ($manifest['when'] as $provider => $events) { 
             $this->registerLoadEvents($provider, $events); 
         } 
```

没错，触发 [`registerLoadEvents`](https://github.com/xiaohuilam/laravel/blob/d081c918b7e582ec5b3f94316f44834466cec37d/vendor/laravel/framework/src/Illuminate/Foundation/ProviderRepository.php#L112-L128) 方法了。感谢 laravel 代码做到了足够语义化，我们猜到了这里就是触发服务提供者注册事件的。具体流程为：
```php

     /** 
      * Register the load events for the given provider. 
      * 
      * @param  string  $provider 
      * @param  array  $events 
      * @return void 
      */ 
     protected function registerLoadEvents($provider, array $events) 
     { 
         if (count($events) < 1) { 
             return; 
         } 
  
         $this->app->make('events')->listen($events, function () use ($provider) { 
             $this->app->register($provider); 
         }); 
     } 
```

* 若需关注更多关于 Laravel 事件触发机制，请关注 [TODO]

再继续后面，就是 ProviderRepository::load() 触发容器的 `register` 的流程了
```php
         // We will go ahead and register all of the eagerly loaded providers with the 
         // application so their services can be registered with the application as 
         // a provided service. Then we will set the deferred service list on it. 
         foreach ($manifest['eager'] as $provider) { 
             $this->app->register($provider); 
         } 
```

`Illuminate\Foundation\Container\Appplication` 容器的 `register` 方法代码为
```php
     /** 
      * Register a service provider with the application. 
      * 
      * @param  \Illuminate\Support\ServiceProvider|string  $provider 
      * @param  bool   $force 
      * @return \Illuminate\Support\ServiceProvider 
      */ 
     public function register($provider, $force = false) 
     { 
         if (($registered = $this->getProvider($provider)) && ! $force) { 
             return $registered; 
         } 
  
         // If the given "provider" is a string, we will resolve it, passing in the 
         // application instance automatically for the developer. This is simply 
         // a more convenient way of specifying your service provider classes. 
         if (is_string($provider)) { 
             $provider = $this->resolveProvider($provider); 
         } 
  
         if (method_exists($provider, 'register')) { 
             $provider->register(); 
         } 
  
         // If there are bindings / singletons set as properties on the provider we 
         // will spin through them and register them with the application, which 
         // serves as a convenience layer while registering a lot of bindings. 
         if (property_exists($provider, 'bindings')) { 
             foreach ($provider->bindings as $key => $value) { 
                 $this->bind($key, $value); 
             } 
         } 
  
         if (property_exists($provider, 'singletons')) { 
             foreach ($provider->singletons as $key => $value) { 
                 $this->singleton($key, $value); 
             } 
         } 
  
         $this->markAsRegistered($provider); 
  
         // If the application has already booted, we will call this boot method on 
         // the provider class so it has an opportunity to do its boot logic and 
         // will be ready for any usage by this developer's application logic. 
         if ($this->booted) { 
             $this->bootProvider($provider); 
         } 
  
         return $provider; 
     } 
```


步骤 
1. 先尝试取 `getProvider` 这个服务，如果能取到证明注册过了，除非要求强硬方式 `$force` 就不再注册 
2. 解析出 `$provier` 为具体的 ServiceProvider 对象 
3. 判断 `$provider` 有无 `register` 方法，有则运行 
4. 判断 `$provider`有无定义 `$bindings` 属性，若有，则依次 `$key`，`$value` 为参，进入 `Illuminate\Foundation\Container\Appplication::bind($key, $value)` 执行 
5. 判断 `$provider` 有无定义 `$singletons` 属性，若有，则依次 `$key`，`$value` 为参，进入 `Illuminate\Foundation\Container\Appplication::singleton($key, $value)` 执行 
6. 标记这个服务提供者已被注册过了 `Illuminate\Foundation\Container\Appplication::$loadedProviders[ServiceProvider::class] = true` 
7. 如果服务提供者已经被启动过了 ($booted == true), 调用 `bootProvider($provider)` 
