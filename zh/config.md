<div class='article-menu'>
  <ul>
    <li>
      <a href="#overview">读取配置</a> <ul>
        <li>
          <a href="#factory">工厂</a>
        </li>
        <li>
          <a href="#native-arrays">本机数组</a>
        </li>
        <li>
          <a href="#file-adapter">文件适配器</a>
        </li>
        <li>
          <a href="#ini-files">读取 INI 文件</a>
        </li>
        <li>
          <a href="#merging">合并的配置</a>
        </li>
        <li>
          <a href="#nested-configuration">嵌套的配置</a>
        </li>
        <li>
          <a href="#injecting-into-di">注射配置依赖项</a>
        </li>
      </ul>
    </li>
  </ul>
</div>

<a name='overview'></a>

# 读取配置

`Phalcon\Config` 到应用程序中使用的 PHP 对象是组件，用于转换配置文件的不同格式 （使用适配器）。

Values can be obtained from `Phalcon\Config` as follows:

```php
<?php

use Phalcon\Config;

$config = new Config(
    [
        'test' => [
            'parent' => [
                'property'  => 1,
                'property2' => 'yeah',
            ],
        ],  
    ]
);

echo $config->get('test')->get('parent')->get('property');  // displays 1
echo $config->test->parent->property;                       // displays 1
echo $config->path('test.parent.property');                 // displays 1
```

<a name='factory'></a>

## 工厂

Loads Config Adapter class using `adapter` option, if no extension is provided it will be added to `filePath`.

```php
<?php

use Phalcon\Config\Factory;

$options = [
    'filePath' => 'path/config',
    'adapter'  => 'php',
 ];

$config = Factory::load($options);
```

<a name='native-arrays'></a>

## 本机数组

The first example shows how to convert native arrays into `Phalcon\Config` objects. 此选项提供了最佳性能，因为在此请求时没有读取文件。

```php
<?php

use Phalcon\Config;

$settings = [
    'database' => [
        'adapter'  => 'Mysql',
        'host'     => 'localhost',
        'username' => 'scott',
        'password' => 'cheetah',
        'dbname'   => 'test_db'
    ],
     'app' => [
        'controllersDir' => '../app/controllers/',
        'modelsDir'      => '../app/models/',
        'viewsDir'       => '../app/views/'
    ],
    'mysetting' => 'the-value'
];

$config = new Config($settings);

echo $config->app->controllersDir, "\n";
echo $config->database->username, "\n";
echo $config->mysetting, "\n";
```

如果你想要更好地组织你的项目你可以在另一个文件中保存该数组，然后阅读它。

```php
<?php

use Phalcon\Config;

require 'config/config.php';

$config = new Config($settings);
```

<a name='file-adapter'></a>

## 文件适配器

可用的适配器是：

| 类                                | 描述                                                                                               |
| -------------------------------- | ------------------------------------------------------------------------------------------------ |
| `Phalcon\Config\Adapter\Ini`  | Uses INI files to store settings. Internally the adapter uses the PHP function `parse_ini_file`. |
| `Phalcon\Config\Adapter\Json` | 使用 JSON 文件来存储设置。                                                                                 |
| `Phalcon\Config\Adapter\Php`  | Uses PHP multidimensional arrays to store settings. This adapter offers the best performance.    |
| `Phalcon\Config\Adapter\Yaml` | 使用 YAML 文件来存储设置。                                                                                 |

<a name='ini-files'></a>

## 读取 INI 文件

Ini 文件是常见的方式来存储设置。 `Phalcon\Config` 使用优化的 PHP 函数 `parse_ini_file` 来读取这些文件。 Files sections are parsed into sub-settings for easy access.

```ini
[database]
adapter  = Mysql
host     = localhost
username = scott
password = cheetah
dbname   = test_db

[phalcon]
controllersDir = '../app/controllers/'
modelsDir      = '../app/models/'
viewsDir       = '../app/views/'

[models]
metadata.adapter  = 'Memory'
```

您可以读取文件，如下所示：

```php
<?php

use Phalcon\Config\Adapter\Ini as ConfigIni;

$config = new ConfigIni('path/config.ini');

echo $config->phalcon->controllersDir, "\n";
echo $config->database->username, "\n";
echo $config->models->metadata->adapter, "\n";
```

<a name='merging'></a>

## 合并的配置

`Phalcon\Config` can recursively merge the properties of one configuration object into another. New properties are added and existing properties are updated.

```php
<?php

use Phalcon\Config;

$config = new Config(
    [
        'database' => [
            'host'   => 'localhost',
            'dbname' => 'test_db',
        ],
        'debug' => 1,
    ]
);

$config2 = new Config(
    [
        'database' => [
            'dbname'   => 'production_db',
            'username' => 'scott',
            'password' => 'secret',
        ],
        'logging' => 1,
    ]
);

$config->merge($config2);

print_r($config);
```

上面的代码产生以下内容：

```bash
Phalcon\Config Object
(
    [database] => Phalcon\Config Object
        (
            [host] => localhost
            [dbname]   => production_db
            [username] => scott
            [password] => secret
        )
    [debug] => 1
    [logging] => 1
)
```

在[Phalcon Incubator](https://github.com/phalcon/incubator)中有更多可用的适配器可用于配置组件

<a name='nested-configuration'></a>

## 嵌套的配置

您可以使用`Phalcon\Config::path`方法轻松访问嵌套的配置值。 这种方法允许获取值，而不考虑路径的某些部分不存在。 让我们看看一个例子：

```php
<?php

use Phalcon\Config;

$config = new Config(
   [
        'phalcon' => [
            'baseuri' => '/phalcon/'
        ],
        'models' => [
            'metadata' => 'memory'
        ],
        'database' => [
            'adapter'  => 'mysql',
            'host'     => 'localhost',
            'username' => 'user',
            'password' => 'passwd',
            'name'     => 'demo'
        ],
        'test' => [
            'parent' => [
                'property' => 1,
                'property2' => 'yeah'
            ],
        ],
   ]
);

// Using dot as delimiter
$config->path('test.parent.property2');    // yeah
$config->path('database.host', null, '.'); // localhost

$config->path('test.parent'); // Phalcon\Config

// Using slash as delimiter. A default value may also be specified and
// will be returned if the configuration option does not exist.
$config->path('test/parent/property3', 'no', '/'); // no

Config::setPathDelimiter('/');
$config->path('test/parent/property2'); // yeah
```

The following example shows how to create usefull facade to access nested configuration values:

```php
<?php

use Phalcon\Di;
use Phalcon\Config;

/**
 * @return mixed|Config
 */
function config() {
    $args = func_get_args();
    $config = Di::getDefault()->getShared(__FUNCTION__);

    if (empty($args)) {
       return $config;
    }

    return call_user_func_array([$config, 'path'], $args);
}
```

<a name='injecting-into-di'></a>

## 注射配置依赖项

You can inject your configuration to the controller allowing us to use `Phalcon\Config` inside `Phalcon\Mvc\Controller`. To be able to do that, you have to add it as a service in the Dependency Injector container. Add following code inside your bootstrap file:

```php
<?php

use Phalcon\Di\FactoryDefault;
use Phalcon\Config;

// Create a DI
$di = new FactoryDefault();

$di->set(
    'config',
    function () {
        $configData = require 'config/config.php';

        return new Config($configData);
    }
);
```

现在在您的控制器可以访问您的配置通过使用依赖注入功能使用名称 `config` 像下面的代码：

```php
<?php

use Phalcon\Mvc\Controller;

class MyController extends Controller
{
    private function getDatabaseName()
    {
        return $this->config->database->dbname;
    }
}
```