# ThinkPHP5 Db类源码分析

## 结构  

ThinkPHP5 Db类是由Db .php（用户入口）  ， Connection (连接器) ， Builder(SQL构造器)，Query（查询器）四部分构成，用户对数据库的所有操作均通过Db .php完成

## 模块详解 

### 1. 用户入口 Db.php

用户入口是用户和数据库操作类的桥梁，作用和index.php入口文件类似，在这个类中，主要完成了调用驱动类和连接器Connection完成连接，然后通过连接器进行下一步操作

先看看Db.php的connect方法

```PHP
/**
     * 数据库初始化，并取得数据库类实例
     * @access public
     * @param  mixed       $config 连接配置
     * @param  bool|string $name   连接标识 true 强制重新连接
     * @return Connection
     * @throws Exception
     */
    public static function connect($config = [], $name = false)
    {
        if (false === $name) {
            $name = md5(serialize($config));
        }

        if (true === $name || !isset(self::$instance[$name])) {
            // 解析连接参数 支持数组和字符串
            $options = self::parseConfig($config);

            if (empty($options['type'])) {
                throw new \InvalidArgumentException('Undefined db type');
            }

            $class = false !== strpos($options['type'], '\\') ?
            $options['type'] :
            '\\think\\db\\connector\\' . ucwords($options['type']);

            // 记录初始化信息
            if (App::$debug) {
                Log::record('[ DB ] INIT ' . $options['type'], 'info');
            }

            if (true === $name) {
                $name = md5(serialize($config));
            }

            self::$instance[$name] = new $class($options);
        }

        return self::$instance[$name];
    }
```

`$name`变量以md5的形式作为同一配置文件的连接标识，当传入的`$name`变量为`True`或类中不存在`$instance[$name]`属性时，将重新建立连接，并将数据库的连接保存在`$instance[$name]`中，调用完成后，返回值为实例化数据库的连接。在实例化连接类的时候，是根据配置文件的要求实例化不同的数据库连接类。



再来看看`__callStatic()`方法

```php
/**
     * 调用驱动类的方法
     * @access public
     * @param  string $method 方法名
     * @param  array  $params 参数
     * @return mixed
     */
    public static function __callStatic($method, $params)
    {
        return call_user_func_array([self::connect(), $method], $params);
    }
```

理解这个方法的核心在于了解`__callStatic()`和`call_user_func_array()`这两个方法

`__callStatic()`是php中的一个魔术方法，当静态调用类中不存在的方法时，`__callStatic()`方法就会执行，同时将调用的方法名和参数分别作为两个参数传入`__callStatic()`方法

`call_user_func_array()`方法：调用回调函数，并把一个数组参数作为回调函数的参数，它不仅可以调用一个函数，同时还可以调用一个类方法，第一个参数传入一个数组，数组元素分别为类和方法名即可，分析`connect`方法时我们已经知道该方法返回值时一个已经实例化的Connection`类

> call_user_func_array — 调用回调函数，并把一个数组参数作为回调函数的参数

以下列操作未来，看看最终发生了什么

```PHP
Db::Query("SELECT * FROM tablename")

    call_user_func_array([self::connect(), $method], $params)
    #等价于 
    $conn = new Connection()
    $conn -> Query("SELECT * FROM tablename")

```

由于Db类中并没有`Query`方法，因此执行了Db类的`__callStatic`方法,`call_user_func_array`方法实际是执行了`Connection`类中的`Query`方法（这里忽略驱动类，实际是现执行驱动类方法，驱动类继承了`Connection`）

到了这一步，程序已经由Db.php进入到`Connection`类中了，下面我们再来看看`Connection`模块



### 2. 连接器 Connection

连接器  顾名思义 连接就是它的核心能力，连接器主要用于与数据库的交互，如创建数据库的连接，记录连接的相关信息，发送sql语句到数据库执行（Query模块的功能也是依赖连接器Connection的query和execute方法实现的），事务等功能，再ThinkPHP中，连接器由Connection基类和数据库驱动类共同组成

首先看看`connect`方法

```php
 /**
   * 连接数据库方法
   * @access public
   * @param array         $config  连接参数
   * @param integer       $linkNum 连接序号
   * @param array|bool    $autoConnection 是否自动连接主数据库（用于分布式）
   * @return PDO
   * @throws Exception
   */
public function connect(array $config = [], $linkNum = 0, $autoConnection = false)
{
    if (!isset($this->links[$linkNum])) {
        if (!$config) {
            $config = $this->config;
        } else {
            $config = array_merge($this->config, $config);
        }
        // 连接参数
        if (isset($config['params']) && is_array($config['params'])) {
            $params = $config['params'] + $this->params;
        } else {
            $params = $this->params;
        }
        // 记录当前字段属性大小写设置
        $this->attrCase = $params[PDO::ATTR_CASE];

        // 数据返回类型
        if (isset($config['result_type'])) {
            $this->fetchType = $config['result_type'];
        }
        try {
            if (empty($config['dsn'])) {
                $config['dsn'] = $this->parseDsn($config);
            }
            if ($config['debug']) {
                $startTime = microtime(true);
            }
            $this->links[$linkNum] = new PDO($config['dsn'], $config['username'], $config['password'], $params);
            if ($config['debug']) {
                // 记录数据库连接信息
                Log::record('[ DB ] CONNECT:[ UseTime:' . number_format(microtime(true) - $startTime, 6) . 's ] ' . $config['dsn'], 'sql');
            }
        } catch (\PDOException $e) {
            if ($autoConnection) {
                Log::record($e->getMessage(), 'error');
                return $this->connect($autoConnection, $linkNum);
            } else {
                throw $e;
            }
        }
    }
    return $this->links[$linkNum];
}
```

`connect`方法有三个默认参数`$config , $linknum ,$autoConnection`分别为数据库连接配置信息，连接序号，以及分布式数据库是否连主库。如果`$linkNum`不存在，将 实例化一个PDO对象连接到数据库，并将连接或者说PDO对象保存在`$this->link[$linkNum]`中，`connect`方法的返回值`$this->links[$linkNum]`就是新创建或者已存在的连接对象



initConnect()方法  连接器在执行Query或execute方法时，均会调用初始化连接方法，该方法的核心在于将connect方法建立的PDO对象保存在linkID属性


```php
/**
  * 初始化数据库连接
  * @access protected
  * @param boolean $master 是否主服务器
  * @return void
  */
protected function initConnect($master = true)
{
    if (!empty($this->config['deploy'])) {
        // 采用分布式数据库
        if ($master || $this->transTimes) {
            if (!$this->linkWrite) {
                $this->linkWrite = $this->multiConnect(true);
            }
            $this->linkID = $this->linkWrite;
        } else {
            if (!$this->linkRead) {
                $this->linkRead = $this->multiConnect(false);
            }
            $this->linkID = $this->linkRead;
        }
    } elseif (!$this->linkID) {
        // 默认单数据库
        $this->linkID = $this->connect();
    }
}
```



`query`方法

```php
/**
     * 执行查询 返回数据集
     * @access public
     * @param string        $sql sql指令
     * @param array         $bind 参数绑定
     * @param bool          $master 是否在主服务器读操作
     * @param bool          $pdo 是否返回PDO对象
     * @return mixed
     * @throws PDOException
     * @throws \Exception
     */
public function query($sql, $bind = [], $master = false, $pdo = false)
{
    
    $this->initConnect($master);
    if (!$this->linkID) {
        return false;
    }

    // 记录SQL语句
    $this->queryStr = $sql;
    if ($bind) {
        $this->bind = $bind;
    }

    Db::$queryTimes++;
    try {
        // 调试开始
        $this->debug(true);

        // 预处理
        $this->PDOStatement = $this->linkID->prepare($sql);

        // 是否为存储过程调用
        $procedure = in_array(strtolower(substr(trim($sql), 0, 4)), ['call', 'exec']);
        // 参数绑定
        if ($procedure) {
            $this->bindParam($bind);
        } else {
            $this->bindValue($bind);
        }
        // 执行查询
        $this->PDOStatement->execute();
        // 调试结束
        $this->debug(false, '', $master);
        // 返回结果集
        return $this->getResult($pdo, $procedure);
    } catch (\PDOException $e) {
        if ($this->isBreak($e)) {
            return $this->close()->query($sql, $bind, $master, $pdo);
        }
        throw new PDOException($e, $this->config, $this->getLastsql());
    } catch (\Throwable $e) {
        if ($this->isBreak($e)) {
            return $this->close()->query($sql, $bind, $master, $pdo);
        }
        throw $e;
    } catch (\Exception $e) {
        if ($this->isBreak($e)) {
            return $this->close()->query($sql, $bind, $master, $pdo);
        }
        throw $e;
    }
}
```

方法内部第一行 `$this->initConnect($master);`作用在于判断是否需要连接主服务器，同时将数据库连接对象(PDO对象)保存在`linkID`属性中，再将SQL语句保存在`queryStr`属性中，如果需要进行参数绑定，再将绑定参数保存在`bind`属性中。

`$querryTimes`是Db类的一个静态属性，用于保存Db类的查询次数。

`$this->PDOStatement = $this->linkID->prepare($sql)`使用PDO对象的`prepare()` $^{注1}$方法对sql语句进行预处理并保存在`PDOStatement`属性中。

`$procedure = in_array(strtolower(substr(trim($sql), 0, 4)), ['call', 'exec'])` $^{注2} $ 通过去除左部空白字符，字符串截取及全部转换为小写等对sql字符串的处理，获取sql语句真正开始的4的字符，判断是否为call或exec 如果是，则为存储过程调用，对存储过程和非存储过程的调用使用不同的参数绑定方法，然后执行查询返回查询的结果集

> 注1：`PDO::prepare()` — 准备要执行的SQL语句并返回一个 `PDOStatement`  对象(PHP 5 >= 5.1.0, PECL pdo >= 0.1.0)
>
> 注2：`strtolower() `函数把字符串转换为小写

`execute`方法

```php
    /**
     * 执行语句
     * @access public
     * @param  string        $sql sql指令
     * @param  array         $bind 参数绑定
     * @param  Query         $query 查询对象
     * @return int
     * @throws PDOException
     * @throws \Exception
     */
    public function execute($sql, $bind = [], Query $query = null)
    {
        $this->initConnect(true);
        if (!$this->linkID) {
            return false;
        }

        // 记录SQL语句
        $this->queryStr = $sql;
        if ($bind) {
            $this->bind = $bind;
        }

        Db::$executeTimes++;
        try {
            // 调试开始
            $this->debug(true);

            // 预处理
            $this->PDOStatement = $this->linkID->prepare($sql);

            // 是否为存储过程调用
            $procedure = in_array(strtolower(substr(trim($sql), 0, 4)), ['call', 'exec']);
            // 参数绑定
            if ($procedure) {
                $this->bindParam($bind);
            } else {
                $this->bindValue($bind);
            }
            // 执行语句
            $this->PDOStatement->execute();
            // 调试结束
            $this->debug(false, '', true);

            if ($query && !empty($this->config['deploy']) && !empty($this->config['read_master'])) {
                $query->readMaster();
            }

            $this->numRows = $this->PDOStatement->rowCount();
            return $this->numRows;
        } catch (\PDOException $e) {
            if ($this->isBreak($e)) {
                return $this->close()->execute($sql, $bind, $query);
            }
            throw new PDOException($e, $this->config, $this->getLastsql());
        } catch (\Throwable $e) {
            if ($this->isBreak($e)) {
                return $this->close()->execute($sql, $bind, $query);
            }
            throw $e;
        } catch (\Exception $e) {
            if ($this->isBreak($e)) {
                return $this->close()->execute($sql, $bind, $query);
            }
            throw $e;
        }
    }
```

`execute`方法的功能和`query`方法相似，略有不同，query方法在分布式数据库下，默认读取的时从服务器数据，可以设置读取主服务器数据，execute方法只能操作主服务器数据，query的返回值是查询的结果集，而execute方法的返回值则是操作影响的行数，如果通过execute方法执行查询语句，返回的将是查询到的结果行数。

`query`方法更适合执行读取操作，即查询语句，而`execute`方法更适合执行写入操作，即插入，更新，删除等



__call方法

```php
    /**
     * 调用Query类的查询方法
     * @access public
     * @param string    $method 方法名称
     * @param array     $args 调用参数
     * @return mixed
     */
    public function __call($method, $args)
    {
        return call_user_func_array([$this->getQuery(), $method], $args);
    }
```

连接器Connection调用查询器Query的方法，是通过`__call`方法实现的，`__call`也是PHP中的一个魔术方法,功能与`__callStatic`类似

> __call 该方法在调用的方法不存在时会自动调用



Connection 方法另一大功能是对事务的操作，这里先不展开分析，以后有机会再单独做深入分析

### 3. 查询器 Query

查询器侧重于对数据的操作。查询器对对数据进行一定的预处理，然后调用SQL生成器Builder生成SQL语句，再通过连接器Connection与数据库交互。ThinkPHP DB类的链式操作的主要方法都在Query类中实现，包括聚集函数，LIMIT  GROUP BY   ORDER BY等操作

对查询器的分析 我们使用Db类进行查询的实例进行展开

```php 
# 实例1  查询一条数据
Db::table('think_user')->where('id',1)->find();
```

`Db::table('think_user')` Db类中并没有table方法，前面的分析中我们已经知道，Db类中有一个`__callStatic`的魔术方法，实现在Db类中调用Connection类的方法，而Connection类中同样有一个`__call`的魔术方法，实现对`Query`类方法的调用。实际的执行过程是这样的


* Db类中没有table方法，因此触发__callStatic方法
```php
Db :: __callStatic("table",["think_user"]) 
```
*  __callStatic方法的 self::connect()实例化了 Connection 
```php
$conn = new Connection()
```
*  __callStatice方法内部的call_user_func_array方法要执行Connection的table方法
```php
$conn -> table("think_user")
```
*  Connection方法中同样没有table方法,执行 __call魔术方法
```php
$conn -> __call("table",['think_user'])
```
*  __call通过$this->getQuery() 实例化了Query 
```php
$conn -> getQuery()
$query = new Query()
```
*  __call 方法内部的call_user_func_array方法执行Query的table方法
```php
$query -> table("think_user")
```
* table方法指定当前要查询的数据表名，保存在Query对象 option['table']属性中,然后返回$this即$query 这也是链式调用的实现方法
```php
$query -> option['table'] = "think_user"   
```
* table方法执行完毕，继续执行where方法 ，where方法指定AND查询条件
```php
$query -> where("id",1)
```
*  where方法内部使用`func_get_args`方法获取参数列表,保存在`$param`变量中，where的第一个参数是字段名，使用		`array_shift`方法删除`$param`的第一个元素
   
```php
$param = func_get_args();
array_shift($param)
```

* 使用`parseWhereExp`方法分析条件表达式

```php
$query -> parseWhereExp('AND', "id", 1, null, $param)
```

* `parseWhereExp`方法对查询条件进行一系列的分析处理后，将查询条件保存到Query对象的option属性中，之后where方法同样返回`$this`即Query对象

```php
$query -> options['multi']["AND"]["ID"] = ["eq",1]
```

* `where`方法执行完毕，接下来执行find方法

```php
$query -> find()
```

* 在find方法内部，调用parseExpress方法分析条件表达式获取条件参数保存到`$options`变量

```php
$options = $query -> parseExpress()
```

* 调用buider属性也就是Buider对象的select方法组装select SQL

```PHP
$sql = $query -> bulider -> select($options)
```

* 调用query方法执行查询

```php
$query -> query($sql)
```

到这里整个查询结果就基本结束了，当然实际的情况比上述描述还要复杂，比如parseWhereExp方法内部的处理过程，这里只是选择最核心的路线进行分析



parseWhereExp方法分析

```PHP
/**
 * 分析查询表达式
 * @access public
 * @param string                $logic     查询逻辑 and or xor
 * @param string|array|\Closure $field     查询字段
 * @param mixed                 $op        查询表达式
 * @param mixed                 $condition 查询条件
 * @param array                 $param     查询参数
 * @param  bool                 $strict    严格模式
 * @return void
 */
protected function parseWhereExp($logic, $field, $op, $condition, $param = [], $strict = false)
{
    $logic = strtoupper($logic);
    if ($field instanceof \Closure) {
        $this->options['where'][$logic][] = is_string($op) ? [$op, $field] : $field;
        return;
    }

    if (is_string($field) && !empty($this->options['via']) && !strpos($field, '.')) {
        $field = $this->options['via'] . '.' . $field;
    }

    if ($field instanceof Expression) {
        return $this->whereRaw($field, is_array($op) ? $op : []);
    } elseif ($strict) {
        // 使用严格模式查询
        $where[$field] = [$op, $condition];

        // 记录一个字段多次查询条件
        $this->options['multi'][$logic][$field][] = $where[$field];
    } elseif (is_string($field) && preg_match('/[,=\>\<\'\"\(\s]/', $field)) {
        $where[] = ['exp', $this->raw($field)];
        if (is_array($op)) {
            // 参数绑定
            $this->bind($op);
        }
    } elseif (is_null($op) && is_null($condition)) {
        if (is_array($field)) {
            // 数组批量查询
            $where = $field;
            foreach ($where as $k => $val) {
                $this->options['multi'][$logic][$k][] = $val;
            }
        } elseif ($field && is_string($field)) {
            // 字符串查询
            $where[$field]                            = ['null', ''];
            $this->options['multi'][$logic][$field][] = $where[$field];
        }
    } elseif (is_array($op)) {
        $where[$field] = $param;
    } elseif (in_array(strtolower($op), ['null', 'notnull', 'not null'])) {
        // null查询
        $where[$field] = [$op, ''];

        $this->options['multi'][$logic][$field][] = $where[$field];
    } elseif (is_null($condition)) {
        // 字段相等查询
        $where[$field] = ['eq', $op];

        $this->options['multi'][$logic][$field][] = $where[$field];
    } else {
        if ('exp' == strtolower($op)) {
            $where[$field] = ['exp', $this->raw($condition)];
            // 参数绑定
            if (isset($param[2]) && is_array($param[2])) {
                $this->bind($param[2]);
            }
        } else {
            $where[$field] = [$op, $condition];
        }
        // 记录一个字段多次查询条件
        $this->options['multi'][$logic][$field][] = $where[$field];
    }

    if (!empty($where)) {
        if (!isset($this->options['where'][$logic])) {
            $this->options['where'][$logic] = [];
        }
        if (is_string($field) && $this->checkMultiField($field, $logic)) {
            $where[$field] = $this->options['multi'][$logic][$field];
        } elseif (is_array($field)) {
            foreach ($field as $key => $val) {
                if ($this->checkMultiField($key, $logic)) {
                    $where[$key] = $this->options['multi'][$logic][$key];
                }
            }
        }
        $this->options['where'][$logic] = array_merge($this->options['where'][$logic], $where);
    }
}
```

以官网的三种示例逐一分析：

表达式查询：

> 新版的表达式查询采用全新的方式，查询表达式的使用格式：
>
> ```php
> Db::table('think_user')
>     ->where('id','>',1)
>     ->where('name','thinkphp')
>     ->select(); 
> ```
>
> 更多的表达式查询语法，可以参考[查询语法](https://www.kancloud.cn/manual/thinkphp5/135182)部分。

在上方所列代码中，出现了两种表达式 `"id",">",1`,和`"name","thinkphp"`这两种表达式等价于`id>1`和`name='thinkphp'`

那么`parseWhereExp`接收到这样的条件表达式，是如何进行处理的呢？

首先看where方法在调用parseWhereExp方法时传入了什么参数

```PHP
public function where($field, $op = null, $condition = null)
{
    $param = func_get_args();
    array_shift($param);
    $this->parseWhereExp('AND', $field, $op, $condition, $param);
    return $this;
}


where('id','>',1)
# 内部执行	
$this->parseWhereExp('AND', "id", ">", 1, ["id", ">", 1]);   
# 传入 parseWhereExp 方法的形参值如下
$logic = "AND", 
$field = "id" , 
$op   	   = ">" , 
$condition = "1" , 
$param = ["id", ">", 1], 
$strict = false
```

下面再来看parseWhereExp方法内部的执行过程

```PHP
# 将logic 全部转为大写
$logic = strtoupper($logic); 
# 处理后logic值如下
$logic = "AND"

# 判断field是否为\Closure 类的实例， field  = 'id'  是字符串，不属于\Closure  条件内部逻辑不执行
if ($field instanceof \Closure)

# 判断 field 是否为字符串 ，options['via'] 不为空 ，field 中 包含.   三个条件需要同时成立，此处$field='id'仅满足第一个条件，本次查询没有设置表别名 if判断内部逻辑不执行
if (is_string($field) && !empty($this->options['via']) && !strpos($field, '.')) {
    $field = $this->options['via'] . '.' . $field;
}

# 判断 $field 是否属于Expression的实例 判断不通过  内部逻辑不执行
if ($field instanceof Expression){

# 判断 $strict 是否为真，（如果为真代表开启严格查询模式） $strict = false  判断不通过 内部逻辑不执行
} elseif ($strict) {
   
# 判断   $field是否为字符串 且是否包含  
} elseif (is_string($field) && preg_match('/[,=\>\<\'\"\(\s]/', $field)) {
    


```











