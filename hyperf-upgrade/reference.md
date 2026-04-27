# Hyperf 升级详细参考

## ⚠️ 重要提示

**升级前必读**：
- 务必在测试环境完整验证后再升级到生产环境
- 建议逐步升级：3.0 → 3.1 → 3.2，不要跨版本跳跃
- 备份所有配置文件和自定义代码
- 仔细阅读每个版本的完整 Changelog

## Logger 配置结构迁移

### 3.1 配置（旧）

```php
// config/autoload/logger.php
return [
    'default' => [
        'driver' => 'daily',
        'path' => BASE_PATH . '/runtime/logs/hyperf.log',
        'level' => env('LOG_LEVEL', 'debug'),
        'days' => 7,
    ],
];
```

### 3.2 配置（新）

```php
// config/autoload/logger.php
return [
    'default' => 'default',  // 指向默认 channel
    'channels' => [
        'default' => [
            'driver' => 'daily',
            'path' => BASE_PATH . '/runtime/logs/hyperf.log',
            'level' => env('LOG_LEVEL', 'debug'),
            'days' => 7,
        ],
        
        // 单文件日志
        'single' => [
            'driver' => 'single',
            'path' => BASE_PATH . '/runtime/logs/single.log',
            'level' => 'debug',
        ],
        
        // 自定义日志通道
        'error' => [
            'driver' => 'daily',
            'path' => BASE_PATH . '/runtime/logs/error.log',
            'level' => 'error',
            'days' => 30,
        ],
        
        // Slack 日志（需要安装 monolog/slack）
        'slack' => [
            'driver' => 'slack',
            'url' => env('LOG_SLACK_WEBHOOK_URL'),
            'username' => 'Hyperf Log',
            'emoji' => ':boom:',
            'level' => 'critical',
        ],
    ],
];
```

### 使用示例

```php
namespace App\Service;

use Psr\Log\LoggerInterface;
use Hyperf\Logger\LoggerFactory;

class LogService
{
    private LoggerInterface $logger;
    private LoggerInterface $errorLogger;

    public function __construct(LoggerFactory $loggerFactory)
    {
        // 使用默认日志
        $this->logger = $loggerFactory->get('default');
        
        // 使用自定义日志通道
        $this->errorLogger = $loggerFactory->get('error');
    }

    public function doSomething(): void
    {
        $this->logger->info('操作成功');
        $this->errorLogger->error('发生错误');
    }
}
```

## Cache 配置结构迁移

### 3.1 配置（旧）

```php
// config/autoload/cache.php
use Hyperf\Cache\Driver\RedisDriver;
use Hyperf\Codec\Packer\PhpSerializerPacker;

return [
    'default' => [
        'driver' => RedisDriver::class,
        'packer' => PhpSerializerPacker::class,
        'prefix' => 'c:',
    ],
];
```

### 3.2 配置（新）

```php
// config/autoload/cache.php
use Hyperf\Cache\Driver\RedisDriver;
use Hyperf\Cache\Driver\FileDriver;
use Hyperf\Codec\Packer\PhpSerializerPacker;
use Hyperf\Codec\Packer\JsonPacker;

return [
    'default' => env('CACHE_DRIVER', 'default'),  // 指向默认 store
    'stores' => [
        'default' => [
            'driver' => RedisDriver::class,
            'packer' => PhpSerializerPacker::class,
            'prefix' => 'c:',
        ],
        
        // 文件缓存
        'file' => [
            'driver' => FileDriver::class,
            'packer' => PhpSerializerPacker::class,
            'path' => BASE_PATH . '/runtime/cache',
        ],
        
        // JSON 序列化缓存
        'json' => [
            'driver' => RedisDriver::class,
            'packer' => JsonPacker::class,
            'prefix' => 'j:',
        ],
        
        // 不同前缀的 Redis 缓存
        'session' => [
            'driver' => RedisDriver::class,
            'packer' => PhpSerializerPacker::class,
            'prefix' => 'session:',
        ],
    ],
];
```

### 使用示例

```php
namespace App\Service;

use Hyperf\Cache\CacheManager;
use Psr\SimpleCache\CacheInterface;

class CacheService
{
    private CacheInterface $defaultCache;
    private CacheInterface $sessionCache;

    public function __construct(CacheManager $cacheManager)
    {
        // 使用默认缓存
        $this->defaultCache = $cacheManager->getCache('default');
        
        // 使用 session 缓存
        $this->sessionCache = $cacheManager->getCache('session');
    }

    public function getUser(int $userId): ?array
    {
        $key = "user:{$userId}";
        
        // 从缓存获取
        $user = $this->defaultCache->get($key);
        
        if ($user === null) {
            // 从数据库查询
            $user = $this->fetchUserFromDb($userId);
            
            // 存入缓存，有效期 3600 秒
            $this->defaultCache->set($key, $user, 3600);
        }
        
        return $user;
    }

    public function setSession(string $sessionId, array $data): void
    {
        $this->sessionCache->set($sessionId, $data, 7200);
    }
}
```

## Request/Response Headers 行为变更

### 3.1 之前（合并模式）

```php
// v3.1 之前的行为
$request->withHeaders(['foo' => 1])->withHeaders(['bar' => 2]);
// 结果：['foo' => [1], 'bar' => [2]]  ✅ 合并

$request->withHeader('foo', 1)->withHeader('foo', 2);
// 结果：['foo' => [2]]  ✅ 替换单个 header
```

### 3.1 之后（替换模式）

```php
// v3.1 之后的行为
$request->withHeaders(['foo' => 1])->withHeaders(['bar' => 2]);
// 结果：['bar' => [2]]  ❌ 完全替换，丢失 foo

// 正确的做法
$request->setHeaders(['foo' => 1])->setHeaders(['bar' => 2]);
// 结果：['bar' => [2]]  ⚠️ 同样是替换

// 如果需要保留多个 headers
$request->withHeader('foo', 1)->withHeader('bar', 2);
// 结果：['foo' => [1], 'bar' => [2]]  ✅ 正确

// 或者使用 add 系列方法
$request->addHeader('foo', 1)->addHeader('foo', 2);
// 结果：['foo' => [1, 2]]  ✅ 添加而非替换
```

### 完整示例

```php
namespace App\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\RequestHandlerInterface;

class CustomMiddleware
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        $response = $handler->handle($request);
        
        // ✅ 推荐方式：逐个设置
        $response = $response
            ->withHeader('X-Custom-Header', 'value1')
            ->withHeader('X-Another-Header', 'value2');
        
        // ✅ 或者使用 setHeaders（一次性设置所有）
        $response = $response->withHeaders([
            'X-Custom-Header' => 'value1',
            'X-Another-Header' => 'value2',
        ]);
        
        // ❌ 避免：多次调用 withHeaders 会覆盖
        // $response = $response->withHeaders(['foo' => 1]);
        // $response = $response->withHeaders(['bar' => 2]);  // foo 会丢失
        
        return $response;
    }
}
```

## Async Queue 任务类迁移

### 3.1 及之前

```php
namespace App\Job;

use Hyperf\AsyncQueue\Job;

class ProcessOrderJob extends Job
{
    protected int $orderId;

    public function __construct(int $orderId)
    {
        $this->orderId = $orderId;
    }

    public function handle(): void
    {
        // 处理订单逻辑
        echo "Processing order: {$this->orderId}\n";
    }

    // 指定队列名称
    public function getQueueName(): string
    {
        return 'order_queue';
    }
}
```

### 3.2 之后

```php
namespace App\Job;

use Hyperf\AsyncQueue\Job;

class ProcessOrderJob extends Job
{
    protected int $orderId;

    public function __construct(int $orderId)
    {
        $this->orderId = $orderId;
    }

    public function handle(): void
    {
        // 处理订单逻辑
        echo "Processing order: {$this->orderId}\n";
    }

    // 方法名改为 getPoolName
    public function getPoolName(): string
    {
        return 'order_queue';
    }
}
```

### 自动注册消费者（3.2 新特性）

```php
// config/autoload/async_queue.php
return [
    'default' => [
        'driver' => Hyperf\AsyncQueue\Driver\RedisDriver::class,
        'redis' => [
            'pool' => 'default',
        ],
        'channel' => '{queue}',
        'timeout' => 2,
        'retry_seconds' => 5,
        'handle_timeout' => 10,
        'processes' => 1,
        'concurrent' => [
            'limit' => 10,
        ],
    ],
    
    // 3.2 新增：支持基于配置的自动注册
    'consumers' => [
        [
            'consumer' => \App\Consumer\OrderConsumer::class,
            'queue' => 'order_queue',
            'nums' => 1,
        ],
        [
            'consumer' => \App\Consumer\NotificationConsumer::class,
            'queue' => 'notification_queue',
            'nums' => 2,
        ],
    ],
];
```

## Carbon 时区使用规范

### 错误用法

```php
use Carbon\Carbon;

$timestamp = time();

// ❌ 3.2 之后不再自动读取默认时区
$date = Carbon::createFromTimestamp($timestamp);
// 可能得到 UTC 时间，而非预期时区
```

### 正确用法

```php
use Carbon\Carbon;

$timestamp = time();

// ✅ 方式一：显式传入时区
$date = Carbon::createFromTimestamp($timestamp, date_default_timezone_get());

// ✅ 方式二：使用固定时区
$date = Carbon::createFromTimestamp($timestamp, 'Asia/Shanghai');

// ✅ 方式三：先创建再设置时区
$date = Carbon::createFromTimestamp($timestamp)
    ->timezone('Asia/Shanghai');

// ✅ 方式四：使用 now() 并设置时区
$date = Carbon::now('Asia/Shanghai');
```

### 完整示例

```php
namespace App\Service;

use Carbon\Carbon;

class TimeService
{
    /**
     * 将时间戳转换为指定时区的日期
     */
    public function timestampToDate(int $timestamp, string $timezone = 'Asia/Shanghai'): Carbon
    {
        return Carbon::createFromTimestamp($timestamp, $timezone);
    }

    /**
     * 获取当前时间（指定时区）
     */
    public function getCurrentTime(string $timezone = 'Asia/Shanghai'): Carbon
    {
        return Carbon::now($timezone);
    }

    /**
     * 格式化时间戳
     */
    public function formatTimestamp(int $timestamp, string $format = 'Y-m-d H:i:s'): string
    {
        return Carbon::createFromTimestamp($timestamp, 'Asia/Shanghai')
            ->format($format);
    }
}
```

## PHP 8.4 CSV 函数兼容

### 问题说明

PHP 8.4 要求 `fgetcsv()` 和 `fputcsv()` 必须提供 `escape` 参数的默认值。

### 修复方案

```php
// ❌ 旧代码（PHP 8.4 会报错）
fputcsv($fp, $fields);
$data = fgetcsv($fp);

// ✅ 新代码（兼容 PHP 8.4）
fputcsv($fp, $fields, escape: '');
$data = fgetcsv($fp, escape: '');

// ✅ 或者使用命名参数
fputcsv($fp, $fields, escape: '');
$data = fgetcsv($fp, escape: '');
```

### 完整示例

```php
namespace App\Service;

class CsvService
{
    /**
     * 导出 CSV
     */
    public function export(array $data, string $filename): void
    {
        $fp = fopen($filename, 'w');
        
        // 写入 BOM（Excel 兼容）
        fprintf($fp, chr(0xEF).chr(0xBB).chr(0xBF));
        
        foreach ($data as $row) {
            // PHP 8.4 兼容
            fputcsv($fp, $row, escape: '');
        }
        
        fclose($fp);
    }

    /**
     * 导入 CSV
     */
    public function import(string $filename): array
    {
        $data = [];
        $fp = fopen($filename, 'r');
        
        while (($row = fgetcsv($fp, escape: '')) !== false) {
            $data[] = $row;
        }
        
        fclose($fp);
        return $data;
    }
}
```

## Swow 引擎依赖注册

### 配置说明

如果使用 Swow 协程引擎，需要在 `config/autoload/dependencies.php` 中手动注册 `ResponseEmitter`。

### 配置示例

```php
// config/autoload/dependencies.php
use Hyperf\Contract\ResponseEmitterInterface;
use Hyperf\Engine\ResponseEmitter;

return [
    // Swow 引擎必需
    ResponseEmitterInterface::class => ResponseEmitter::class,
    
    // 其他自定义依赖绑定
    App\Repository\UserRepositoryInterface::class => App\Repository\UserRepository::class,
];
```

### 检测是否使用 Swow

```bash
# 检查 composer.json
cat composer.json | grep swow

# 或检查已安装的包
composer show | grep swow
```

## Prometheus 监控集成

### 安装依赖

```bash
composer require promphp/prometheus_client_php
composer require hyperf/metric
```

### 配置示例

```php
// config/autoload/metric.php
return [
    'default' => 'prometheus',
    'use_standalone_process' => true,
    'enable_default_metric' => true,
    'enable_command_metric' => false,
    'metric' => [
        'prometheus' => [
            'driver' => Hyperf\Metric\Adapter\Prometheus\MetricFactory::class,
            'mode' => Hyperf\Metric\Constant::MODE_STANDALONE,
            'namespace' => 'hyperf',
            'storage_adapter' => 'memory',  // 或 'redis'
        ],
    ],
];
```

### Redis 存储适配器

```php
// config/autoload/metric.php
return [
    'metric' => [
        'prometheus' => [
            'driver' => Hyperf\Metric\Adapter\Prometheus\MetricFactory::class,
            'mode' => Hyperf\Metric\Constant::MODE_STANDALONE,
            'namespace' => 'hyperf',
            'storage_adapter' => 'redis',
            'redis' => [
                'host' => '127.0.0.1',
                'port' => 6379,
                'password' => null,
                'timeout' => 0.1,
                'read_timeout' => 10,
            ],
        ],
    ],
];
```

## 依赖版本对照表

### Hyperf 3.2 主要依赖版本

| 组件 | 版本要求 | 说明 |
|------|---------|------|
| PHP | >= 8.2 | 最低版本要求 |
| Swoole | >= 5.0 | 协程引擎 |
| Symfony Components | ^6.0 \|\| ^7.0 | HTTP、事件等组件 |
| PHPUnit | ^11.0 | 测试框架 |
| Elasticsearch | ^8.0 \|\| ^9.0 | 搜索引擎客户端 |
| Google Protobuf | ^3.6.1 \|\| ^4.2 | gRPC 支持 |
| Guzzle | ^7.0 | HTTP 客户端 |
| Monolog | ^3.0 | 日志库（不再支持 2.x） |
| Carbon | ^3.0 | 时间处理库 |

### 更新命令

```bash
# 更新 Hyperf 核心
composer require hyperf/framework:~3.2.0

# 更新 Symfony 组件
composer require symfony/http-foundation:^7.0
composer require symfony/event-dispatcher:^7.0

# 更新测试框架
composer require --dev phpunit/phpunit:^11.0

# 更新 Elasticsearch
composer require elasticsearch/elasticsearch:^8.0

# 更新 Monolog
composer require monolog/monolog:^3.0
```

## 升级脚本示例

### 自动化检查脚本

```php
#!/usr/bin/env php
<?php
// scripts/check_upgrade.php

echo "=== Hyperf 升级检查 ===\n\n";

// 检查 PHP 版本
$phpVersion = PHP_VERSION;
echo "PHP 版本: {$phpVersion}\n";
if (version_compare($phpVersion, '8.2.0', '<')) {
    echo "❌ PHP 版本过低，需要 >= 8.2\n";
} else {
    echo "✅ PHP 版本符合要求\n";
}

echo "\n";

// 检查扩展
$requiredExtensions = ['swoole', 'redis', 'pdo_mysql', 'mbstring', 'openssl'];
foreach ($requiredExtensions as $ext) {
    if (extension_loaded($ext)) {
        echo "✅ {$ext} 扩展已加载\n";
    } else {
        echo "❌ {$ext} 扩展未加载\n";
    }
}

echo "\n";

// 检查配置文件
$configFiles = [
    'config/autoload/logger.php',
    'config/autoload/cache.php',
    'config/autoload/dependencies.php',
];

foreach ($configFiles as $file) {
    if (file_exists(BASE_PATH . '/' . $file)) {
        echo "✅ {$file} 存在\n";
        
        // 检查配置结构
        $content = file_get_contents(BASE_PATH . '/' . $file);
        
        if (str_contains($file, 'logger.php')) {
            if (str_contains($content, "'channels'")) {
                echo "   ✅ Logger 配置结构正确\n";
            } else {
                echo "   ❌ Logger 配置需要更新为 3.2 结构\n";
            }
        }
        
        if (str_contains($file, 'cache.php')) {
            if (str_contains($content, "'stores'")) {
                echo "   ✅ Cache 配置结构正确\n";
            } else {
                echo "   ❌ Cache 配置需要更新为 3.2 结构\n";
            }
        }
    } else {
        echo "⚠️  {$file} 不存在\n";
    }
}

echo "\n=== 检查完成 ===\n";
```

## 回滚方案

如果升级后出现问题，可以快速回滚：

### 1. 回滚 Composer 依赖

```bash
# 如果有 composer.lock
git checkout composer.lock
composer install

# 或者指定版本
composer require hyperf/framework:~3.1.0
```

### 2. 回滚配置文件

```bash
# 恢复备份的配置
cp config/autoload/logger.php.bak config/autoload/logger.php
cp config/autoload/cache.php.bak config/autoload/cache.php
```

### 3. 清理缓存

```bash
# 清理运行时缓存
rm -rf runtime/container/*
rm -rf runtime/cache/*

# 重启服务
php bin/hyperf.php start
```

## 相关资源

- [Hyperf 官方升级文档](https://hyperf.wiki/#/zh-cn/upgrade)
- [Hyperf 3.1 升级指南](https://github.com/hyperf/hyperf/blob/master/docs/zh-cn/upgrade/3.1.md)
- [Hyperf 3.2 Changelog](https://github.com/hyperf/hyperf/blob/3.2/CHANGELOG-3.2.md)
- [Monolog 3.x 升级指南](https://github.com/Seldaek/monolog/blob/main/UPGRADE.md)
- [Carbon 3.x 升级指南](https://github.com/briannesbitt/Carbon/blob/master/README.md)
