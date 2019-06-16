---
title: laravel5.7安装jwt-auth的正确方法完整流程
date: 2019-06-11
categories: php
tag: 
    - laravel
    - jwt-auth
---

**由于没有完整的Laravel 5.7与jwt-auth集成的指南，官网的文档陈旧不更新，所以我决定创建一个。
我假设你有一个Laravel支持auth的项目。**

#### 安装包jwt-auth
将包添加到composer.json

```php
"require": {
    ...
    "tymon/jwt-auth": "1.0.0-rc.3"
}
```

然后按命令更新composer
```php
composer update
```

### 设置配置
##### 生成密钥
```php
php artisan jwt:secret
```

##### 发布配置文件
```php
php artisan vendor:publish --provider="Tymon\JWTAuth\Providers\LaravelServiceProvider"
```

##### 打开app.php并添加以下行

在'aliases array'数组中
```php
'JWTAuth'=> Tymon\JWTAuth\Facades\JWTAuth::class，
'JWTFactory'=> Tymon\JWTAuth\Facades\JWTFactory::class，
```

##### 更新用户模型
打开User.php并为模型实现JWTSubject

```php
use Tymon\JWTAuth\Contracts\JWTSubject;
class User extends Authenticatable implements JWTSubject
```
并在模型中添加2个方法（在未完成的官方指南中阅读有关这些功能的更多信息）
```php
public function getJWTIdentifier（）{
    return $ this-> getKey（）;
}
public function getJWTCustomClaims（）{
    return [];
}
```

##### 更新配置auth.php
打开config/auth.php并将API的保护驱动程序更改为'jwt'（默认为令牌）
```php
'guards'=> [ 
    ...
    'api'=> [
        'driver'=>'jwt'，
        'provider'=>'users'，
    ]，
]，
```

##### 创建登录控制器
创建controller用于login、logout、refresh等

```php
<?php

namespace App\Http\Controllers\Auth;

use Illuminate\Support\Facades\Auth;
use App\Http\Controllers\Controller;

class JwtAuthController extends Controller
{
    /**
     * Create a new AuthController instance.
     * 要求附带email和password（数据来源users表）
     *
     * @return void
     */
    public function __construct()
    {
        $this->middleware('jwt.auth', ['except' => ['login']]);
        // 另外关于上面的中间件，官方文档写的是『auth:api』
        // 但是我推荐用 『jwt.auth』，效果是一样的，但是有更加丰富的报错信息返回
    }

    /**
     * Get a JWT via given credentials.
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function login()
    {
        $credentials = request(['email', 'password']);

        if (! $token = auth('api')->attempt($credentials)) {
            return response()->json(['error' => 'Unauthorized'], 401);
        }

        return $this->respondWithToken($token);
    }

    /**
     * Get the authenticated User.
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function me()
    {
        return response()->json(auth('api')->user());
    }

    /**
     * Log the user out (Invalidate the token).
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function logout()
    {
        auth('api')->logout();

        return response()->json(['message' => 'Successfully logged out']);
    }

    /**
     * Refresh a token.
     * 刷新token，如果开启黑名单，以前的token便会失效。
     * 值得注意的是用上面的getToken再获取一次Token并不算做刷新，两次获得的Token是并行的，即两个都可用。
     * @return \Illuminate\Http\JsonResponse
     */
    public function refresh()
    {
        return $this->respondWithToken(auth('api')->refresh());
    }

    /**
     * Get the token array structure.
     *
     * @param  string $token
     *
     * @return \Illuminate\Http\JsonResponse
     */
    protected function respondWithToken($token)
    {
        return response()->json([
            'access_token' => $token,
            'token_type' => 'bearer',
            'expires_in' => auth('api')->factory()->getTTL() * 60
        ]);
    }
}
```
##### 创建路由
打开`routes/api.php`然后添加一些路由

```php
Route::group(['prefix' => 'auth'], function () {
    Route::post('login', 'Auth\\JwtAuthController@login');
    Route::post('logout', 'Auth\\JwtAuthController@logout');
    Route::post('refresh', 'Auth\\JwtAuthController@refresh');
    Route::post('me', 'Auth\\JwtAuthController@me');
});
```
然后在标头请求中添加“Authorization：Bearer {token}”

**如果你想捕获异常
在你的app/Exceptions/Handler.php中捕获'render'函数中的错误。**

```php
use Symfony\Component\HttpKernel\Exception\UnauthorizedHttpException;

……

if ($exception instanceof UnauthorizedHttpException) {
    $preException = $exception->getPrevious();
    if ($preException instanceof
                  \Tymon\JWTAuth\Exceptions\TokenExpiredException) {
        return response()->json(['error' => 'TOKEN_EXPIRED']);
    } else if ($preException instanceof
                  \Tymon\JWTAuth\Exceptions\TokenInvalidException) {
        return response()->json(['error' => 'TOKEN_INVALID']);
    } else if ($preException instanceof
             \Tymon\JWTAuth\Exceptions\TokenBlacklistedException) {
         return response()->json(['error' => 'TOKEN_BLACKLISTED']);
   }
   if ($exception->getMessage() === 'Token not provided') {
       return response()->json(['error' => 'Token not provided']);
   }
}

```
就这样。


> 翻译自：https://medium.com/@hdcompany123/laravel-5-7-and-json-web-tokens-tymon-jwt-auth-d5af05c0659a
> 并做了适当修改,希望能够帮助到各位同学。