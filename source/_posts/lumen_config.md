---
title: Lumen自定义配置
date: 2017-07-06 23:42:22
tags: [PHP,LUMEN,LARAVEL]
category: 后端
---

先从一个问题说起，Lumen在使用Redis时，文档中有一段说明：
>If you have not called $app->withEloquent() in your bootstrap/app.php file, then you should call app->configure('database'); in the bootstrap/app.php file to ensure the Redis database configuration is properly loaded.

如果没有调用`$app->withEloquent()`的话，需要调用`$app->configure('database')`来加载redis的配置。为什么需要这么做呢？
### redis使用的配置
Redis使用的配置比较容易查看，在安装`illuminate/redis`后查看`vendor/illuminate/redis/RedisServiceProvider.php`，可以看到该Service使用的配置是`database.redis`
**注意区分CacheService使用的配置，Cache使用的配置是config/cache.php**
### database.php在什么时候加载？
按照文档的说法，需要调用`$app->withEloquent()` 或者 `$app->configure('database')`。
withEloquent做了什么？

```php
#withEloquent调用了make，创建db模块
public function withEloquent()
{
   $this->make('db');
}
#make是Appliation提供的注册模块实例的方法
public function make($abstract)
{
	#先从this->alias中获取模块对应的别名
	$abstract = $this->getAlias($abstract);
	#根据名称在availableBindings中查找对应的注册方法进行初始化
	if (array_key_exists($abstract, $this->availableBindings) &&
	  ! array_key_exists($this->availableBindings[$abstract], $this->ranServiceBinders)) {
	  $this->{$method = $this->availableBindings[$abstract]}();
	  $this->ranServiceBinders[$method] = true;
	}
	return parent::make($abstract);
}
#availableBindings定义了模块和其对应的初始化方法
 public $availableBindings = [
	...
   'db' => 'registerDatabaseBindings',
   'config' => 'registerConfigBindings',
   ...
];
#registerDatabaseBindings方法中调用loadComponent初始化db模块，并注册到Application中，
protected function registerDatabaseBindings()
{
   $this->singleton('db', function () {
       return $this->loadComponent(
           'database', [
               'Illuminate\Database\DatabaseServiceProvider',
               'Illuminate\Pagination\PaginationServiceProvider',
           ], 'db'
       );
   });
}
#loadComponent会调用configure加载配置
public function loadComponent($config, $providers, $return = null)
{
   $this->configure($config);
   foreach ((array) $providers as $provider) {
       $this->register($provider);
   }
   return $this->make($return ?: $config);
}

```

所以，如果调用了withEloquent，则database就已经被加载进来，不要再用configure注册了。
## 配置加载
再来看Lumen的配置加载方式
### Laravel的配置加载方式
比较简单，在boostraps中调用LoadConfiguration，扫描config目录下的所有文件，使用Illuminate\Contracts\Config\Repository（Config实际使用的类）保存配置。
如果开启了配置cache，则会加载bootstrap/cache/config.php。
### Lumen的配置加载方式
从上一节可以看到，Lumen的配置加载是按照模块的需要，最终调用configure方法进行加载。configure方法的调用代码如下。
从代码中可以看出来，configure会先从**"项目根目录/config"**目录下加载配置，如果不存在则从**"Application.php所在目录/../config"**加载。

```php
#配置加载方法
public function configure($name)
{
   if (isset($this->loadedConfigurations[$name])) {
       return;
   }
   $this->loadedConfigurations[$name] = true;
	  #获取配置路径
   $path = $this->getConfigurationPath($name);
   if ($path) {
   		#生成config模块的实例
       $this->make('config')->set($name, require $path);
   }
}
#从上一节中的availableBindings数组可以看出，config模块实例初始化方法为registerConfigBindings
protected function registerConfigBindings()
{
   $this->singleton('config', function () {
   		#Illuminate\Config\Repository as ConfigRepository;
       return new ConfigRepository;
   });
}
#配置路径获取方法
public function getConfigurationPath($name = null)
{
   if (! $name) {
       $appConfigDir = $this->basePath('config').'/';

       if (file_exists($appConfigDir)) {
           return $appConfigDir;
       } elseif (file_exists($path = __DIR__.'/../config/')) {
           return $path;
       }
   } else {
       $appConfigPath = $this->basePath('config').'/'.$name.'.php';

       if (file_exists($appConfigPath)) {
           return $appConfigPath;
       } elseif (file_exists($path = __DIR__.'/../config/'.$name.'.php')) {
           return $path;
       }
   }
}
```
### 总结
从Lumen的配置加载可以看出来，**Lumen没有像Laravel那样自动加载config下的所有配置文件**（从这里也可以看出，Lumen按需加载的原则）。因此，如果我们需要增加自定义的配置文件，需要在bootstrap/app.php中或者其他地方调用`$app->configure('xxx.php')`

另外，从configure获取路径的方式也会发现，可以将`vendor/laravel/lumen-framework/src/../config`文件夹拷贝到项目根目录，以方便配置的修改。


