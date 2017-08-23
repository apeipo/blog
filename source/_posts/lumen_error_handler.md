---
title: Lumen和Laravel错误处理机制修改
date: 2017-07-12 21:50:42
tags: [PHP,LARAVEL,LUMEN]
category: [PHP]
---
在使用Laravel或者Lumen时会碰到这种情况，如果php的代码中产生了Notice或者Warning，会导致Lumen跳到错误页，日志中会打印一个很长很长的stack trace，如下图：

```php
$app->get('/test', function () use ($app) {
	$arr = [];
	print($arr['name']);
});
#请求/test 产生的日志
[2017-07-12 11:55:54] lumen.ERROR: ErrorException: Undefined index: name in /home/vagrant/Code/featurestream/routes/web.php:16
Stack trace:
#0 /home/vagrant/Code/featurestream/routes/web.php(16): Laravel\Lumen\Application->Laravel\Lumen\Concerns\{closure}(8, 'Undefined index...', '/home/vagrant/C...', 16, Array)
#1 [internal function]: Closure->{closure}()
#2 /home/vagrant/Code/featurestream/vendor/illuminate/container/BoundMethod.php(29): call_user_func_array(Object(Closure), Array)
....
```

上面的日志只截取了一小部分，实际运行时最简单的api请求Lumen的stack trace会有30层左右，Laravel的会有60层..
另外一个重要的问题是，这种处理机制会让一些小错误把整个请求搞挂，代码中到处加`if(isset($arr['xx']))` 或者设置默认值。
对于这个问题，Laravel和Lumen设计的初衷是好的：**所有的PHP错误都应该被处理，包括Notice和Warning**。就是太严格了，有时候我们并不需要代码有这么严格的检查，出现Notice或者Warning时，只需要打印个模块日志或者有PHP日志就行。
先看下Lumen中的错误是如何处理的（Laravel中也差不多，不再单独讲）。

## 错误处理流程
### 错误Handler
先看Lumen的入口：`bootstrap/app.php`，有一段错误处理相关的代码：

```php
$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);
```
singleton是往app（Lumen的服务容器）里注入实例的方法，这里实例化了一个ExceptionHandler类，实例为Handler。后面的代码如果有从$app里取ExceptionHandler的实例的话，会返回Handler这个类的实例。
Handler类在App\Exceptions目录下，代码如下。比较简单，只包含两个方法，再去看父类(`Laravel\Lumen\Exceptions\Handler`)的方法逻辑会发现，**report方法负责打印日志（也就是上文那个长长的trace），render方法会根据错误类型的不同构建错误页面**。

```php
    public function report(Exception $e)
    {
        parent::report($e);
    }

    public function render($request, Exception $e)
    {
        return parent::render($request, $e);
    }
```

错误的处理逻辑找到了，如何触发进入这个逻辑的呢？
### 错误触发
正常情况下，PHP产生Notice或者Warning是不会抛出Exception的，会产生Exception肯定是框架内部做了更改。
`bootstrap/app.php`中没找到设置错误处理的地方，接着往下看Lumen框架的容器类Application（Lumen的核心类和入口），目录：`vendor/laravel/lumen-framework/src/Application.php`。
可以看到在其构造函数中调用了一个registerErrorHandling方法，方法代码：

```php
    /**
     * Set the error handling for the application.
     *
     * @return void
     */
    protected function registerErrorHandling()
    {
        error_reporting(-1);

        set_error_handler(function ($level, $message, $file = '', $line = 0) {
            if (error_reporting() & $level) {
                throw new ErrorException($message, 0, $level, $file, $line);
            }
        });

        set_exception_handler(function ($e) {
            $this->handleUncaughtException($e);
        });

        register_shutdown_function(function () {
            $this->handleShutdown();
        });
    }
```

`set_error_handler`是PHP设置错误处理的方法。registerErrorHandling下用`error_reporting(-1)`把PHP的报错开关都打开，这样所有级别的错误都会触发`error_handler`。在`error_handler`中抛出了一个ErrorException异常。
至此，我们知道ErrorException这个异常是怎么产生的了，知道异常会交由谁来处理了（上一节中的Handler类）。不过还有个疑问，ErrorException和Handler是怎么关联起来的？
### ErrorException和Handler
`set_error_handler`设置的方法中会抛出异常，那肯定存在针对异常的try catch块。
从请求路口`public/index.php`往下看，
-->`bootstrap/app.php` -->`Application.php -> run` -->`Application.php -> dispatch`
在dispatch方法中发现了try catch的逻辑，在catch到Exception后调用了Handler的report和render方法。

```php
public function dispatch($request = null)
{
   list($method, $pathInfo) = $this->parseIncomingRequest($request);
   try {
			...
   } catch (Exception $e) {
       return $this->prepareResponse($this->sendExceptionToHandler($e));
   } catch (Throwable $e) {
       return $this->prepareResponse($this->sendExceptionToHandler($e));
   }
}
protected function sendExceptionToHandler($e)
{
   $handler = $this->resolveExceptionHandler();

   if ($e instanceof Error) {
       $e = new FatalThrowableError($e);
   }

   $handler->report($e);

   return $handler->render($this->make('request'), $e);
}    
```
## 解决
知道了错误从触发到结束的整个流程，再来看怎么解决.
### 方法1.屏蔽错误
简单粗暴的方法，既然框架用来`error_reporting(-1)`打开了所有错误开关，那我们再用`error_reporting`把我们不关心的错误给屏蔽了。把下面代码加在`bootstrap/app.php`中，**注意得加在Applition实例化之后**。

```php
error_reporting(E_ALL ^ E_NOTICE ^ E_WARNING);
```
这种方法的弊端显而易见，我们只是不想让E_NOTICE搞挂请求，但是这类错误还是得关注和修复的。。
### 方法2.修改Handler
既然是Handler负责错误处理，那我们修改Handler(`App\Exceptions\Handler`)的逻辑就行。

```php
    public function report(Exception $e)
    {
        //parent::report($e);
        Log::warning($e->getMessage(), ['file' => $e->getFile(), 'line' => $e->getLine()]);
    }

    public function render($request, Exception $e)
    {
        //return parent::render($request, $e);
        header('Content-type: application/json');
        echo json_encode(['errmsg' => $e->getMessage()]);
        return ;
    }
```

如上代码，将错误处理逻辑进行更改，只打印简单的日志，并且也不跳到错误页了。
不过，这种方法还是有问题：

1. 代码产生的NOTICE和WARNING还是会中断执行流程（因为会抛出异常）。比如一个请求中执行A->B->C三个方法，如果B中产生了一个NOTICE，整个请求还是会被中断，C不会被执行。
2. 如果一个请求原本不是返回json，是返回一个view，则上面的render方法就不适用了。

### 方法3.在业务逻辑顶层中catch异常
原理类似方法2，只不过把异常的处理从Handler中移到了业务逻辑里（比如Controller中）。
这个方法的问题和方法2一样，NOTICE还是会中断请求...

### 方法4.修改`error_handler`，不抛出异常
罪魁祸首就在于那个`set_error_handler`注册的处理方法遇到PHP错误就会抛出异常，那我们修改他就行。
`set_error_handler`所在的registerErrorHandling方法在框架的源代码中，最好不要直接修改，我们可以在它注册完之后再重新注册一个覆盖它。可以加在`bootstrap/app.php`中，Application实例化之后。代码如下：

```php
set_error_handler(function ($level, $message, $file = '', $line = 0) {
    if ($level == E_NOTICE || $level == E_WARNING) {
        Log::warning("PHP NOTICE or WARNING; MSG:[$message]", ['file' => $file, 'line' => $line]);
        return;
    }
    if (error_reporting() & $level) {
        throw new ErrorException($message, 0, $level, $file, $line);
    }
});
```

注意，**代码中因为使用了Log这个Facades，因此必须放在$app->withFacades()之后**
至此，NOTICE和WARNING不会产生烦人的日志，也不会搞挂请求，并且也保留了有效的提示信息。：）



