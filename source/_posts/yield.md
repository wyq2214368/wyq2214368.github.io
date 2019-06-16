---
title: yield 生成器 读取超大文件
date: 2018-02-01 
categories: PHP
tag: yield
---
生成器函数的核心是yield关键字。它最简单的调用形式看起来像一个return申明，不同之处在于普通return会返回值并终止函数的执行，而yield会返回一个值给循环调用此生成器的代码并且只是暂停执行生成器函数。

### 读取超大文件

``` php
function readTxt($file)
{
    # code...
    $handle = fopen($file, 'rb');

    while (feof($handle)===false) {
        # code...
        yield fgets($handle);
    }

    fclose($handle);
}

```

### 形成斐波那数列

``` php
function fb($n){
	$a = 0;$b=1;
	while($n >0){
		yield $b;
		list($b,$a) = [$a + $b,$b];
		$n--;
	}
}
//如果用递归实现必须要cache节点数据,不然就是以指数增长的计算。
$map   = [];
function fbd($n){
	global $map;
	if($n == 1)return 1;
	if($n == 2)return 2;
	echo 'aaaaaaa'.PHP_EOL;
	if(isset($map[$n]) ){
		return $map[$n];
	}else{
		$data = f($n-1) + f($n-2);
		$map[$n] = $data;
		return $data;
	}
}
```