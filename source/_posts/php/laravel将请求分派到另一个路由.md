---
title: laravel将请求分派到另一个路由
date: 2019-06-25
categories: php
tag: laravel
---

> 参考laravel手册 https://laravel.com/api/5.7/Illuminate/Routing/Router.html

最近有个业务场景是这样：
```php
//在管理模块中，要根据模块名动态跳转到该模块的controller来执行configs方法
//有类似如下两条路由
Route::get('manage/{module}', 'Common\\ManageController@configs');
Route::get('module1/config', 'Common\\Module1Controller@configs')->name('articles.configs');
```
访问A路由，要动态的跳转B路由，即 

```php
Class ManageController
{

	...
	
	public function configs($module)
	{
		return redirect()->route("$module.configs");
	}
}

Class Module1Controller
{

	...
	
	public function configs(Request $request)
	{
		// do something
	}
}
```

但这样会有302重定向，本来想通过nginx的location + rewrite来重写地址，来取出302的跳转。后来想了想觉得laravel这么优雅肯定有相关函数可以用。

于是查阅了[learnku的laravel文档](https://learnku.com/docs/laravel/5.7/routing/2253)，但没有找到，google查阅前几页也没有提到解决方案，然后自己查了查laravel原始文档，发现Router类提供了responseWithRoute方法，于是将Controller修改如下：
```php
Class ManageController
{

	...
	
	public function configs($module)
	{
		request()->merge(['module' => $module]); //将参数值添加到request中
        return Route::respondWithRoute("$module.configs");
	}
}

```
更多可用方法可进一步查阅文档。

特此记个笔记，希望对同学们有帮助～