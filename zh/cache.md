<div class='article-menu'>
  <ul>
    <li>
      <a href="#overview">使用高速缓存提高性能</a> 
      <ul>
        <li>
          <a href="#implementation">When to implement cache?</a>
        </li>
        <li>
          <a href="#caching-behavior">缓存行为</a>
        </li>
        <li>
          <a href="#factory">工厂</a>
        </li>
        <li>
          <a href="#output-fragments">缓存的视图片段</a>
        </li>
        <li>
          <a href="#arbitrary-data">任意的数据缓存</a> 
          <ul>
            <li>
              <a href="#backend-file-example">文件后端示例</a>
            </li>
            <li>
              <a href="#backend-memcached-example">Memcached 后端示例</a>
            </li>
          </ul>
        </li>
        <li>
          <a href="#read">查询缓存</a>
        </li>
        <li>
          <a href="#delete">从缓存中删除数据</a>
        </li>
        <li>
          <a href="#exists">Checking cache existence</a>
        </li>
        <li>
          <a href="#lifetime">生命周期</a>
        </li>
        <li>
          <a href="#multi-level">多级缓存</a>
        </li>
        <li>
          <a href="#adapters-frontend">前端适配器</a> 
          <ul>
            <li>
              <a href="#adapters-frontend-custom">Implementing your own Frontend adapters</a>
            </li>
          </ul>
        </li>
        <li>
          <a href="#adapters-backend">后端适配器</a> 
          <ul>
            <li>
              <a href="#adapters-backend-factory">工厂</a>
            </li>
            <li>
              <a href="#adapters-backend-custom">Implementing your own Backend adapters</a>
            </li>
            <li>
              <a href="#adapters-backend-file">文件后端选项</a>
            </li>
            <li>
              <a href="#adapters-backend-libmemcached">Libmemcached 后端选项</a>
            </li>
            <li>
              <a href="#adapters-backend-memcache">Memcache 后端选项</a>
            </li>
            <li>
              <a href="#adapters-backend-apc">APC 后端选项</a>
            </li>
            <li>
              <a href="#adapters-backend-apcu">APCU 后端选项</a>
            </li>
            <li>
              <a href="#adapters-backend-mongo">Mongo后端选项</a>
            </li>
            <li>
              <a href="#adapters-backend-xcache">XCache 后端选项</a>
            </li>
            <li>
              <a href="#adapters-backend-redis">Redis后端选项</a>
            </li>
          </ul>
        </li>
      </ul>
    </li>
  </ul>
</div>

<a name='overview'></a>

# Improving Performance with Cache

Phacon提供 `Phalcon\Cache` 类允许更快地访问常用或已处理的数据。 `Phalcon\Cache` 是用 C 编写的实现更高的性能和降低开销，当从后端获取项目。 此类使用前端和后端组件的内部的结构。 前端组件作为输入的源或接口后, 端组件提供对类的存储选项。

<a name='implementation'></a>

## 什么时候使用缓存

Although this component is very fast, implementing it in cases that are not needed could lead to a loss of performance rather than gain. We recommend you check this cases before using a cache:

* 你使复杂的运算，每次都返回相同的结果 （不经常更改）
* 你正在使用大量的助手和生成的输出几乎都是一样
* 你正在不断地访问数据库中的数据，这些数据很少更改

<div class='alert alert-warning'>
    <p>
        <strong>注意</strong>即使在执行缓存之后, 应在一段时间检查您的缓存的命中率。 这能很容易做到，尤其是 Memcache 或 Apc，与后端提供的相关工具。
    </p>
</div>

<a name='caching-behavior'></a>

## 缓存行为

缓存的过程分为 2 个部分：

* <0Frontend</strong>： 这一部分是负责检查，如果密钥已过期，并且执行更多转换对数据存储之前和之后从后端-检索
* **Backend**： 这部分是负责沟通，写/读前端所需的数据。

<a name='factory'></a>

## 工厂

实例化前端或后端适配器可以通过两种方式实现：

传统的方式

```php
<?php

use Phalcon\Cache\Backend\File as BackFile;
use Phalcon\Cache\Frontend\Data as FrontData;

// Create an Output frontend. Cache the files for 2 days
$frontCache = new FrontData(
    [
        'lifetime' => 172800,
    ]
);

// Create the component that will cache from the 'Output' to a 'File' backend
// Set the cache file directory - it's important to keep the '/' at the end of
// the value for the folder
$cache = new BackFile(
    $frontCache,
    [
        'cacheDir' => '../app/cache/',
    ]
);
```

或使用工厂对象，如下所示：

```php
<?php

use Phalcon\Cache\Frontend\Factory as FFactory;
use Phalcon\Cache\Backend\Factory as BFactory;

 $options = [
     'lifetime' => 172800,
     'adapter'  => 'data',
 ];
 $frontendCache = FFactory::load($options);


$options = [
    'cacheDir' => '../app/cache/',
    'prefix'   => 'app-data',
    'frontend' => $frontendCache,
    'adapter'  => 'file',
];

$backendCache = BFactory::load($options);
```

<a name='output-fragments'></a>

## 缓存的视图片段

输出片段是一块的 HTML 或文本，缓存是原样退回。 从 `ob_ *` 函数或 PHP 输出，这样它可以保存在缓存中，将自动捕获输出。 下面的示例演示这种用法。 它接收由 PHP 生成的输出，并将其存储到一个文件。 该文件的内容是每 172,800 秒刷新 （2 天）。

The implementation of this caching mechanism allows us to gain performance by not executing the helper `Phalcon\Tag::linkTo()` call whenever this piece of code is called.

```php
<?php

use Phalcon\Tag;
use Phalcon\Cache\Backend\File as BackFile;
use Phalcon\Cache\Frontend\Output as FrontOutput;

// Create an Output frontend. Cache the files for 2 days
$frontCache = new FrontOutput(
    [
        'lifetime' => 172800,
    ]
);

// Create the component that will cache from the 'Output' to a 'File' backend
// Set the cache file directory - it's important to keep the '/' at the end of
// the value for the folder
$cache = new BackFile(
    $frontCache,
    [
        'cacheDir' => '../app/cache/',
    ]
);

// Get/Set the cache file to ../app/cache/my-cache.html
$content = $cache->start('my-cache.html');

// If $content is null then the content will be generated for the cache
if ($content === null) {
    // Print date and time
    echo date('r');

    // Generate a link to the sign-up action
    echo Tag::linkTo(
        [
            'user/signup',
            'Sign Up',
            'class' => 'signup-button',
        ]
    );

    // Store the output into the cache file
    $cache->save();
} else {
    // Echo the cached output
    echo $content;
}
```

<div class='alert alert-warning'>
    <p>
        <strong>注意</strong>在上面的示例中，我们的代码保持不变，回显输出给用户，像之前它做的那样。 我们缓存组件透明地捕获该输出并将其存储在缓存文件中 （当生成缓存时） 或它将其发送回用户预编译从以前的调用，从而避免昂贵的操作。
    </p>
</div>

<a name='arbitrary-data'></a>

## 任意的数据缓存

Caching just data is equally important for your application. Caching can reduce database load by reusing commonly used (but not updated) data, thus speeding up your application.

<a name='backend-file-example'></a>

### 文件后端示例

缓存的适配器之`File`。 此适配器的唯一的重点区域是位置的将存储缓存文件的位置。 这是由 `cacheDir` 选项，其中 *必须* 有一个反斜杠在结束了它的控制。

```php
<?php

use Phalcon\Cache\Backend\File as BackFile;
use Phalcon\Cache\Frontend\Data as FrontData;

// Cache the files for 2 days using a Data frontend
$frontCache = new FrontData(
    [
        'lifetime' => 172800,
    ]
);

// Create the component that will cache 'Data' to a 'File' backend
// Set the cache file directory - important to keep the `/` at the end of
// the value for the folder
$cache = new BackFile(
    $frontCache,
    [
        'cacheDir' => '../app/cache/',
    ]
);

$cacheKey = 'robots_order_id.cache';

// Try to get cached records
$robots = $cache->get($cacheKey);

if ($robots === null) {
    // $robots is null because of cache expiration or data does not exist
    // Make the database call and populate the variable
    $robots = Robots::find(
        [
            'order' => 'id',
        ]
    );

    // Store it in the cache
    $cache->save($cacheKey, $robots);
}

// Use $robots :)
foreach ($robots as $robot) {
   echo $robot->name, '\n';
}
```

<a name='backend-memcached-example'></a>

### Memcached 后端示例

上面的例子会略有改变 （尤其是在职权配置） 当我们使用 Memcached 后端。

```php
<?php

use Phalcon\Cache\Frontend\Data as FrontData;
use Phalcon\Cache\Backend\Libmemcached as BackMemCached;

// Cache data for one hour
$frontCache = new FrontData(
    [
        'lifetime' => 3600,
    ]
);

// Create the component that will cache 'Data' to a 'Memcached' backend
// Memcached connection settings
$cache = new BackMemCached(
    $frontCache,
    [
        'servers' => [
            [
                'host'   => '127.0.0.1',
                'port'   => '11211',
                'weight' => '1',
            ]
        ]
    ]
);

$cacheKey = 'robots_order_id.cache';

// Try to get cached records
$robots = $cache->get($cacheKey);

if ($robots === null) {
    // $robots is null because of cache expiration or data does not exist
    // Make the database call and populate the variable
    $robots = Robots::find(
        [
            'order' => 'id',
        ]
    );

    // Store it in the cache
    $cache->save($cacheKey, $robots);
}

// Use $robots :)
foreach ($robots as $robot) {
   echo $robot->name, '\n';
}
```

<div class='alert alert-warning'>
    <p>
        <strong>注意</strong><code>Save （）</code> 调用此方法将返回一个布尔值，该值指示成功 (<code>true</code>) 或失败 (<code>false</code>)。 根据您使用的后端，需要看看相关的日志，以确定故障。
    </p>
</div>

<a name='read'></a>

## 查询缓存

The elements added to the cache are uniquely identified by a key. 在文件的后端，关键是实际的文件名。 要从缓存中检索数据，我们只需要调用它使用的唯一键。 如果不存在的键，get 方法将返回 null。

```php
<?php

// Retrieve products by key 'myProducts'
$products = $cache->get('myProducts');
```

如果你想要知道哪些键存储在缓存中你可以调用 `queryKeys` 方法：

```php
<?php

// Query all keys used in the cache
$keys = $cache->queryKeys();

foreach ($keys as $key) {
    $data = $cache->get($key);

    echo 'Key=', $key, ' Data=', $data;
}

// Query keys in the cache that begins with 'my-prefix'
$keys = $cache->queryKeys('my-prefix');
```

<a name='delete'></a>

## 从缓存中删除数据

There are times where you will need to forcibly invalidate a cache entry (due to an update in the cached data). The only requirement is to know the key that the data have been stored with.

```php
<?php

// Delete an item with a specific key
$cache->delete('someKey');

$keys = $cache->queryKeys();

// 从缓存中删除这些数据
foreach ($keys as $key) {
    $cache->delete($key);
}
```

<a name='exists'></a>

## 检查缓存是否存在

它是可能要检查是否缓存中已存在具有给定的键：

```php
<?php

if ($cache->exists('someKey')) {
    echo $cache->get('someKey');
} else {
    echo 'Cache does not exists!';
}
```

<a name='lifetime'></a>

## 生命周期

A `lifetime` is a time in seconds that a cache could live without expire. By default, all the created caches use the lifetime set in the frontend creation. 在创建或从缓存中数据的检索，您可以设置特定的生存期：

设置生存期时检索：

```php
<?php

$cacheKey = 'my.cache';

// Setting the cache when getting a result
$robots = $cache->get($cacheKey, 3600);

if ($robots === null) {
    $robots = 'some robots';

    // Store it in the cache
    $cache->save($cacheKey, $robots);
}
```

在保存时设置生存期：

```php
<?php

$cacheKey = 'my.cache';

$robots = $cache->get($cacheKey);

if ($robots === null) {
    $robots = 'some robots';

    // Setting the cache when saving data
    $cache->save($cacheKey, $robots, 3600);
}
```

<a name='multi-level'></a>

## 多级缓存

高速缓存组件，此功能允许开发人员来实现多级缓存。 这一新功能是十分有用的因为你可以将相同的数据保存在几个缓存位置具有不同的生存期，首先阅读从一个更快的适配器和结束与最慢的一个，直到数据过期：

```php
<?php

use Phalcon\Cache\Multiple;
use Phalcon\Cache\Backend\Apc as ApcCache;
use Phalcon\Cache\Backend\File as FileCache;
use Phalcon\Cache\Frontend\Data as DataFrontend;
use Phalcon\Cache\Backend\Memcache as MemcacheCache;

$ultraFastFrontend = new DataFrontend(
    [
        'lifetime' => 3600,
    ]
);

$fastFrontend = new DataFrontend(
    [
        'lifetime' => 86400,
    ]
);

$slowFrontend = new DataFrontend(
    [
        'lifetime' => 604800,
    ]
);

// Backends are registered from the fastest to the slower
$cache = new Multiple(
    [
        new ApcCache(
            $ultraFastFrontend,
            [
                'prefix' => 'cache',
            ]
        ),
        new MemcacheCache(
            $fastFrontend,
            [
                'prefix' => 'cache',
                'host'   => 'localhost',
                'port'   => '11211',
            ]
        ),
        new FileCache(
            $slowFrontend,
            [
                'prefix'   => 'cache',
                'cacheDir' => '../app/cache/',
            ]
        ),
    ]
);

// Save, saves in every backend
$cache->save('my-key', $data);
```

<a name='adapters-frontend'></a>

## 前端适配器

使用可用的前端适配器接口或输入的源到缓存中：

| 适配器                                  | 描述                                                                                                                                                             |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Phalcon\Cache\Frontend\Output`   | 从标准的 PHP 输出中读取输入的数据。                                                                                                                                           |
| `Phalcon\Cache\Frontend\Data`     | It's used to cache any kind of PHP data (big arrays, objects, text, etc). Data is serialized before stored in the backend.                                     |
| `Phalcon\Cache\Frontend\Base64`   | It's used to cache binary data. The data is serialized using `base64_encode` before be stored in the backend.                                                  |
| `Phalcon\Cache\Frontend\Json`     | Data is encoded in JSON before be stored in the backend. Decoded after be retrieved. This frontend is useful to share data with other languages or frameworks. |
| `Phalcon\Cache\Frontend\Igbinary` | It's used to cache any kind of PHP data (big arrays, objects, text, etc). Data is serialized using `Igbinary` before be stored in the backend.                 |
| `Phalcon\Cache\Frontend\None`     | It's used to cache any kind of PHP data without serializing them.                                                                                              |

<a name='adapters-frontend-custom'></a>

### Implementing your own Frontend adapters

The `Phalcon\Cache\FrontendInterface` interface must be implemented in order to create your own frontend adapters or extend the existing ones.

<a name='adapters-backend'></a>

## Backend Adapters

The backend adapters available to store cache data are:

| Adapter                                 | Description                                          | Info                                      | Required Extensions                                |
| --------------------------------------- | ---------------------------------------------------- | ----------------------------------------- | -------------------------------------------------- |
| `Phalcon\Cache\Backend\Apc`          | Stores data to the Alternative PHP Cache (APC).      | [APC](http://php.net/apc)                 | [APC](http://pecl.php.net/package/APC)             |
| `Phalcon\Cache\Backend\Apcu`         | Stores data to the APCu (APC without opcode caching) | [APCu](http://php.net/apcu)               | [APCu](http://pecl.php.net/package/APCu)           |
| `Phalcon\Cache\Backend\File`         | Stores data to local plain files.                    |                                           |                                                    |
| `Phalcon\Cache\Backend\Libmemcached` | Stores data to a memcached server.                   | [Memcached](http://www.php.net/memcached) | [Memcached](http://pecl.php.net/package/memcached) |
| `Phalcon\Cache\Backend\Memcache`     | Stores data to a memcached server.                   | [Memcache](http://www.php.net/memcache)   | [Memcache](http://pecl.php.net/package/memcache)   |
| `Phalcon\Cache\Backend\Mongo`        | Stores data to Mongo Database.                       | [MongoDB](http://mongodb.org/)            | [Mongo](http://mongodb.org/)                       |
| `Phalcon\Cache\Backend\Redis`        | Stores data in Redis.                                | [Redis](http://redis.io/)                 | [Redis](http://pecl.php.net/package/redis)         |
| `Phalcon\Cache\Backend\Xcache`       | Stores data in XCache.                               | [XCache](http://xcache.lighttpd.net/)     | [XCache](http://pecl.php.net/package/xcache)       |

<div class='alert alert-warning'>
    <p>
        <strong>NOTE</strong> In PHP 7 to use phalcon <code>apc</code> based adapter classes you needed to install <code>apcu</code> and <code>apcu_bc</code> package from pecl. Now in Phalcon 3.2.0 you can switch your <code>*\Apc</code> classes to <code>*\Apcu</code> and remove <code>apcu_bc</code>. Keep in mind that in Phalcon 4 we will most likely remove all `*\Apc` classes.
    </p>
</div>

<a name='adapters-backend-factory'></a>

### Factory

There are many backend adapters (see [Backend Adapters](#adapters-backend)). The one you use will depend on the needs of your application. The following example loads the Backend Cache Adapter class using `adapter` option, if frontend will be provided as array it will call Frontend Cache Factory

```php
<?php

use Phalcon\Cache\Backend\Factory;
use Phalcon\Cache\Frontend\Data;

$options = [
    'prefix'   => 'app-data',
    'frontend' => new Data(),
    'adapter'  => 'apc',
];
$backendCache = Factory::load($options);
```

<a name='adapters-backend-custom'></a>

### Implementing your own Backend adapters

The `Phalcon\Cache\BackendInterface` interface must be implemented in order to create your own backend adapters or extend the existing ones.

<a name='adapters-backend-file'></a>

### 文件后端选项

This backend will store cached content into files in the local server. The available options for this backend are:

| Option     | Description                                                 |
| ---------- | ----------------------------------------------------------- |
| `prefix`   | A prefix that is automatically prepended to the cache keys. |
| `cacheDir` | A writable directory on which cached files will be placed.  |

<a name='adapters-backend-libmemcached'></a>

### Libmemcached 后端选项

This backend will store cached content on a memcached server. Per default persistent memcached connection pools are used. The available options for this backend are:

**General options**

| Option          | Description                                                                                                        |
| --------------- | ------------------------------------------------------------------------------------------------------------------ |
| `statsKey`      | Used to tracking of cached keys.                                                                                   |
| `prefix`        | A prefix that is automatically prepended to the cache keys.                                                        |
| `persistent_id` | To create an instance that persists between requests, use `persistent_id` to specify a unique ID for the instance. |

**Servers options**

| Option   | Description                                                                                                 |
| -------- | ----------------------------------------------------------------------------------------------------------- |
| `host`   | The `memcached` host.                                                                                       |
| `port`   | The `memcached` port.                                                                                       |
| `weight` | The weight parameter effects the consistent hashing used to determine which server to read/write keys from. |

**Client options**

Used for setting Memcached options. See [Memcached::setOptions](http://php.net/manual/en/memcached.setoptions.php) for more.

**Example**

```php
<?php
use Phalcon\Cache\Backend\Libmemcached;
use Phalcon\Cache\Frontend\Data as FrontData;

// Cache data for 2 days
$frontCache = new FrontData(
    [
        'lifetime' => 172800,
    ]
);

// Create the Cache setting memcached connection options
$cache = new Libmemcached(
    $frontCache,
    [
        'servers' => [
            [
                'host'   => '127.0.0.1',
                'port'   => 11211,
                'weight' => 1,
            ],
        ],
        'client' => [
            \Memcached::OPT_HASH       => \Memcached::HASH_MD5,
            \Memcached::OPT_PREFIX_KEY => 'prefix.',
        ],
        'persistent_id' => 'my_app_cache',
    ]
);
```

<a name='adapters-backend-memcache'></a>

### Memcache 后端选项

This backend will store cached content on a memcached server. The available options for this backend are:

| Option       | Description                                                 |
| ------------ | ----------------------------------------------------------- |
| `prefix`     | A prefix that is automatically prepended to the cache keys. |
| `host`       | The memcached host.                                         |
| `port`       | The memcached port.                                         |
| `persistent` | Create a persistent connection to memcached?                |

<a name='adapters-backend-apc'></a>

### APC 后端选项

This backend will store cached content on Alternative PHP Cache ([APC](http://php.net/apc)). The available options for this backend are:

| Option   | Description                                                 |
| -------- | ----------------------------------------------------------- |
| `prefix` | A prefix that is automatically prepended to the cache keys. |

<a name='adapters-backend-apcu'></a>

### APCU 后端选项

This backend will store cached content on Alternative PHP Cache ([APCU](http://php.net/apcu)). The available options for this backend are:

| Option   | Description                                                 |
| -------- | ----------------------------------------------------------- |
| `prefix` | A prefix that is automatically prepended to the cache keys. |

<a name='adapters-backend-mongo'></a>

### Mongo后端选项

This backend will store cached content on a MongoDB server ([MongoDB](http://mongodb.org/)). The available options for this backend are:

| Option       | Description                                                 |
| ------------ | ----------------------------------------------------------- |
| `prefix`     | A prefix that is automatically prepended to the cache keys. |
| `server`     | A MongoDB connection string.                                |
| `db`         | Mongo database name.                                        |
| `collection` | Mongo collection in the database.                           |

<a name='adapters-backend-xcache'></a>

### XCache 后端选项

This backend will store cached content on XCache ([XCache](http://xcache.lighttpd.net/)). The available options for this backend are:

| Option   | Description                                                 |
| -------- | ----------------------------------------------------------- |
| `prefix` | A prefix that is automatically prepended to the cache keys. |

<a name='adapters-backend-redis'></a>

### Redis后端选项

This backend will store cached content on a Redis server ([Redis](http://redis.io/)). The available options for this backend are:

| Option       | Description                                                    |
| ------------ | -------------------------------------------------------------- |
| `prefix`     | A prefix that is automatically prepended to the cache keys.    |
| `host`       | Redis host.                                                    |
| `port`       | Redis port.                                                    |
| `auth`       | Password to authenticate to a password-protected Redis server. |
| `persistent` | Create a persistent connection to Redis.                       |
| `index`      | The index of the Redis database to use.                        |

There are more adapters available for this components in the [Phalcon Incubator](https://github.com/phalcon/incubator)