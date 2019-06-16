---
title: 对ID进行隐藏
date: 2019-01-11
categories: PHP
tag: Larael
---
## Hashid

不希望对外暴露有规则的数据索引，比如用户 ID 、媒体资源 ID 、商品 ID 、订单号、注册码、优惠码等，防止爬虫侵扰。那就将ID编码咯。

解决方案 [vinkla/hashids](https://github.com/ivanakimov/hashids.php) composer 懂的哈
可以自己加salt，这样每个系统的加密就不一样的，不容易被破解。
``` php
use Hashids\Hashids;

$hashids = new Hashids();

$hashids->encode(1);
```
Laravel 更加有优雅的解决方案哟 [Laravel Hashid](https://learnku.com/courses/laravel-package/hash-data-id-vinklahashids/1945）


