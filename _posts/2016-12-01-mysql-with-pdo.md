---
layout: post
title: PDO操作MySQL的安全性问题
tags: [PHP, MySQL, 注入]
comment: false
---

在网上搜到关于PDO的信息大多都是如何使用它来防止注入这样的安全问题出现，导致以前一直觉得只要是使用PDO的预处理来操作数据库的话就不会再有注入的出现。直到前些时间看了微擎的源码后发现并不是这样的，再好的东西用的不好还是会有问题出现。也难怪它被挖了这么多的注入。。。  

## 一些基本的用法  
- PDO::query() — 执行一条语句，并返回一个PDOStatement对象
- PDO::exec() — 执行一条 SQL 语句，并返回受影响的行数
- PDO::errorInfo() — 获取跟上一次语句句柄操作相关的扩展错误信息
- PDO::prepare() — 预处理一条SQL语句，并返回一个PDOStatement对象
- PDOStatement::execute() — 执行一条预处理语句

结合点代码来看  
```php
<?php
	define( 'host', 'localhost' );
    define( 'user', 'root' );
    define( 'password', 'root' );
    define( 'dbname', 'mytest' );

    function connect(){
        try {
            $PDO = new PDO( 'mysql:host=' . host . ';dbname=' . dbname, user, password );
        }catch (PDOException $e) {
            echo 'Error connecting to MySQL: ' . $e->getMessage();
        }

        return $PDO;
    }

    $pdo = connect();
    /*
    $res = $pdo->query("SELECT * FROM admin;");
    返回一个PDOStatement对象，可通过fetchobject()来获取数据
    */
    /*
    $res = $pdo->exec("DELETE from admin WHERE id=3 or id=4;");
    返回影响的行数，这里的话就是2了
    */

    //$sql = "SELECT * FROM admin WHERE id=? or id=?";
    $sql = "SELECT * FROM admin WHERE id=:id1 or id=:id2";
    //上面是两种不同的参数标记方式

    $stmt = $pdo->prepare($sql);

    //$res = $stmt->execute(array(1, 2));
    $res = $stmt->execute(array('id1'=>1, 'id2'=>2));
    //上面是对应不同参数标记方式的绑定PHP变量的方法，也可以用bindParam()来绑定变量
    
    var_dump($res);
    echo '<br>';
    echo '<br>errorinfo:<br>';
    var_dump($stmt->errorinfo());
    echo '<br><br>data:<br>';
    var_dump($stmt->fetchobject());
?>
```  
## 问题的出现  
正常的预处理的话是不会发生问题的，但当预处理的句子受用户的输入控制的时候问题就出现了。预处理句子可控的话就跟平时的注入没啥区别了，联合查询或是PDO支持多语句的执行都可以，要是调用了errorinfo的话还可以构造报错的句子来进行注入。  

有的时候代码是用了用户的输入来做为预处理语句里的参数标记，这种情况预处理的句子就是可控的了。看个微擎实际的例子。  

framework/class/db.class.php中的implode函数  

```php
private function implode($params, $glue = ',') {
        $result = array('fields' => ' 1 ', 'params' => array());
        $split = '';
        $suffix = '';
        $allow_operator = array('>', '<', '<>', '!=', '>=', '<=', '+=', '-=', 'LIKE', 'like');
        if (in_array(strtolower($glue), array('and', 'or'))) {
            $suffix = '__';
        }
        if (!is_array($params)) {
            $result['fields'] = $params;
            return $result;
        }
        if (is_array($params)) {
            $result['fields'] = '';
            foreach ($params as $fields => $value) {
                $operator = '';
                if (strpos($fields, ' ') !== FALSE) {
                    list($fields, $operator) = explode(' ', $fields, 2);
                    if (!in_array($operator, $allow_operator)) {
                        $operator = '';
                    }
                }
                if (empty($operator)) {
                    $fields = trim($fields);
                    if (is_array($value)) {
                        $operator = 'IN';
                    } else {
                        $operator = '=';
                    }
                } elseif ($operator == '+=') {
                    $operator = " = `$fields` + ";
                } elseif ($operator == '-=') {
                    $operator = " = `$fields` - ";
                }
                if (is_array($value)) {
                    $insql = array();
                    foreach ($value as $k => $v) {
                        $insql[] = ":{$suffix}{$fields}_{$k}";
                        $result['params'][":{$suffix}{$fields}_{$k}"] = is_null($v) ? '' : $v;
                    }
                    $result['fields'] .= $split . "`$fields` {$operator} (".implode(",", $insql).")";
                    $split = ' ' . $glue . ' ';
                } else {
                    $result['fields'] .= $split . "`$fields` {$operator}  :{$suffix}$fields";
                    $split = ' ' . $glue . ' ';
                    $result['params'][":{$suffix}$fields"] = is_null($value) ? '' : $value;
                }
            }
        }
    return $result;
}
```  
要是传入的参数是一个数组的话，它会拿数组的名字和下标结合成一个预处理语句的参数标记。拿一个之前的漏洞点来试下，加个die(var_dump())来看预处理的句子  
![1.jpg](https://ooo.0o0.ooo/2016/12/04/5843b98de9b81.jpg)
参数标记还要成功绑定才不会直接是错误不执行预处理的句子，结合一个之前的注入点这里的一个利用方法就是  
![2.jpg](https://ooo.0o0.ooo/2016/12/04/5843bb1e853b3.jpg)
这里用报错语句的原因是代码默认是有调用errorinfo函数的。再贴张当时在官方demo测试的图  
![3.jpg](https://ooo.0o0.ooo/2016/12/04/5843bbce6bd77.jpg)

## 后记
这个是之前在先知提交的，后来发现重了，而且这么久了官方也一直没修。就当是记录PDO的一些知识了，还有就是这文章本来两天前就应该写好了的，结果傻逼的误删了一个卷再加上一些其它的事就拖到了今天了  - -

