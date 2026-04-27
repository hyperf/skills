---
name: hyperf-upgrade
description: Hyperf 版本感知开发指南。当检测到项目使用 Hyperf 3.x 时，自动应用对应版本的编码规范：3.1+ 必须使用 `\Hyperf\Support\env()` 或 `use function Hyperf\Support\env` 而非全局 `env()`；3.2+ 需使用新的 Logger/Cache 配置结构。适用于代码生成、配置编写、问题诊断等所有 Hyperf 开发场景。
---

# Hyperf 版本感知开发指南

## ⚠️ 核心原则

**本 Skill 不仅用于升级，更用于日常开发中的版本适配！**

当检测到项目使用特定 Hyperf 版本时，应**自动应用**对应的编码规范和最佳实践。

### 版本检测与自动适配规则

| 检测到的版本 | 自动应用的规范 |
|------------|--------------|
| **Hyperf 3.0** | 可使用全局 `env()`，扁平配置结构 |
| **Hyperf 3.1+** | ❌ 禁止全局 `env()` → ✅ 必须使用 `\Hyperf\Support\env()` 或导入函数 |
| **Hyperf 3.2+** | ✅ Logger/Cache 必须使用嵌套配置结构（channels/stores） |

## 核心概念

本指南涵盖 Hyperf 3.x 系列版本的**版本感知开发规范**，确保代码始终符合当前版本的 best practices。

### 版本要求对比

| 版本 | PHP 最低版本 | Swoole 最低版本 | 关键开发规范 |
|------|------------|---------------|------------|
| 3.0  | 8.0        | 4.6           | 可使用全局 `env()` |
| 3.1  | 8.1        | 5.0           | **必须**使用 `\Hyperf\Support\env()` |
| 3.2  | 8.2        | 5.0           | **必须**使用嵌套配置结构 |

## 版本感知开发规范

### 规则 1：环境变量函数使用（3.1+ 强制）

**触发条件**：检测到项目使用 Hyperf >= 3.1

**❌ 错误做法**（会导致运行时错误）：
```php
// config/autoload/wechat.php
return [
    'app_id' => env('WECHAT_APPID'),  // ❌ 3.1+ 已移除全局 env()
];
```

**✅ 正确做法**（二选一）：

**方式一：完整命名空间（推荐用于配置文件）**
```php
// config/autoload/wechat.php
return [
    'app_id' => \Hyperf\Support\env('WECHAT_APPID'),
    'secret' => \Hyperf\Support\env('WECHAT_SECRET'),
];
```

**方式二：导入函数（推荐用于 PHP 类文件）**
```php
<?php
namespace App\Service;

use function Hyperf\Support\env;

class WeChatService
{
    public function getConfig(): array
    {
        return [
            'app_id' => env('WECHAT_APPID'),
            'secret' => env('WECHAT_SECRET'),
        ];
    }
}
```

**AI 自动修正示例**：
```
检测到：env('APP_NAME')
Hyperf 版本：3.2
自动修正为：\Hyperf\Support\env('APP_NAME')
原因：Hyperf 3.1+ 已移除全局 env() 函数
```

### 规则 2：Logger 配置结构（3.2+ 强制）

**触发条件**：检测到项目使用 Hyperf >= 3.2

**❌ 错误做法**（3.1 及之前的扁平结构）：
```php
// config/autoload/logger.php
return [
    'default' => [
        'driver' => 'daily',
        'path' => BASE_PATH . '/runtime/logs/hyperf.log',
        'level' => 'debug',
    ],
];
```

**✅ 正确做法**（3.2+ 嵌套结构）：
```php
// config/autoload/logger.php
return [
    'default' => 'default',  // 必须：指向默认 channel
    'channels' => [          // 必须：channels 包裹
        'default' => [
            'driver' => 'daily',
            'path' => BASE_PATH . '/runtime/logs/hyperf.log',
            'level' => 'debug',
        ],
    ],
];
```

**AI 自动修正示例**：
```
检测到：config/autoload/logger.php 使用扁平结构
Hyperf 版本：3.2
自动修正为：添加 'default' 指向和 'channels' 包裹
原因：Hyperf 3.2+ Logger 配置结构变更
```

### 规则 3：Cache 配置结构（3.2+ 强制）

**触发条件**：检测到项目使用 Hyperf >= 3.2

**❌ 错误做法**（3.1 及之前的扁平结构）：
```php
// config/autoload/cache.php
return [
    'default' => [
        'driver' => RedisDriver::class,
        'packer' => PhpSerializerPacker::class,
        'prefix' => 'c:',
    ],
];
```

**✅ 正确做法**（3.2+ 嵌套结构）：
```php
// config/autoload/cache.php
return [
    'default' => env('CACHE_DRIVER', 'default'),  // 必须：指向默认 store
    'stores' => [                                  // 必须：stores 包裹
        'default' => [
            'driver' => RedisDriver::class,
            'packer' => PhpSerializerPacker::class,
            'prefix' => 'c:',
        ],
    ],
];
```

### 规则 4：Request/Response Headers 处理（3.1+ 注意）

**触发条件**：检测到项目使用 Hyperf >= 3.1

**⚠️ 行为变更**：`withHeaders()` 从合并改为替换

**❌ 可能导致 Bug 的做法**：
```php
// 期望保留 foo 和 bar 两个 header
$response->withHeaders(['foo' => 1])
         ->withHeaders(['bar' => 2]);
// 3.1+ 结果：只有 bar，foo 被覆盖！
```

**✅ 正确做法**：
```php
// 方式一：逐个设置
$response->withHeader('foo', 1)
         ->withHeader('bar', 2);

// 方式二：一次性设置所有
$response->withHeaders([
    'foo' => 1,
    'bar' => 2,
]);
```

## 快速参考卡片

### Hyperf 3.1+ 开发检查清单

在生成或审查代码时，自动检查以下项：

- [ ] **env() 函数**：是否使用了 `\Hyperf\Support\env()` 或导入了函数？
- [ ] **配置文件**：子目录配置是否使用 `config('sub.key')` 语法？
- [ ] **AMQP 消费者**：返回值类型是否为 `Result` 枚举而非 `string`？
- [ ] **Headers 处理**：是否避免了多次调用 `withHeaders()`？

### Hyperf 3.2+ 开发检查清单

在 3.1 基础上增加：

- [ ] **Logger 配置**：是否使用 `channels` 嵌套结构？
- [ ] **Cache 配置**：是否使用 `stores` 嵌套结构？
- [ ] **Async Queue**：是否使用 `getPoolName()` 而非 `getQueueName()`？
- [ ] **Carbon 时区**：`createFromTimestamp()` 是否显式传入时区？
- [ ] **PHP 版本**：是否 >= 8.2？

## 实际应用场景

### 场景 1：AI 生成配置文件

**用户请求**：帮我创建一个微信配置文件

**AI 检测到**：项目使用 Hyperf 3.2

**AI 自动生成**：
```php
<?php
// config/autoload/wechat.php

use function Hyperf\Support\env;  // ✅ 自动添加导入

return [
    'official_account' => [
        'app_id' => env('WECHAT_OFFICIAL_ACCOUNT_APPID'),  // ✅ 使用导入的 env
        'secret' => env('WECHAT_OFFICIAL_ACCOUNT_SECRET'),
        'token' => env('WECHAT_OFFICIAL_ACCOUNT_TOKEN'),
    ],
    'mini_program' => [
        'app_id' => env('WECHAT_MINI_PROGRAM_APPID'),
        'secret' => env('WECHAT_MINI_PROGRAM_SECRET'),
    ],
];
```

**而不是**：
```
// ❌ 错误：直接使用全局 env()
return [
    'official_account' => [
        'app_id' => env('WECHAT_OFFICIAL_ACCOUNT_APPID'),
    ],
];
```

### 场景 2：AI 审查现有代码

**检测到代码**：
```php
$appId = env('WECHAT_APPID');
```

**AI 分析**：
```
⚠️ 发现问题：使用了全局 env() 函数
Hyperf 版本：3.2
建议修正：
  1. 在文件顶部添加：use function Hyperf\Support\env;
  2. 或使用完整命名空间：\Hyperf\Support\env('WECHAT_APPID')
原因：Hyperf 3.1+ 已移除全局 env() 函数
```

### 场景 3：AI 创建 Logger 配置

**用户请求**：帮我配置日志系统

**AI 检测到**：项目使用 Hyperf 3.2

**AI 自动生成**：
```php
<?php
// config/autoload/logger.php

return [
    'default' => 'default',  // ✅ 自动添加 default 指向
    'channels' => [          // ✅ 自动使用 channels 包裹
        'default' => [
            'driver' => 'daily',
            'path' => BASE_PATH . '/runtime/logs/hyperf.log',
            'level' => 'debug',
            'days' => 7,
        ],
    ],
];
```

**而不是**：
```
// ❌ 错误：使用旧的扁平结构
return [
    'default' => [
        'driver' => 'daily',
        'path' => BASE_PATH . '/runtime/logs/hyperf.log',
    ],
];
```

## 最佳实践

### DO ✓

- **版本检测优先**：在生成代码前先检测 Hyperf 版本
- **自动应用规范**：根据版本自动选择正确的编码方式
- **明确标注版本**：在代码注释中标注适用的最低版本
- **提供备选方案**：同时展示新旧版本的写法对比

### DON'T ✗

- **不要假设版本**：始终先确认项目的 Hyperf 版本
- **不要混用规范**：避免在同一项目中混用不同版本的写法
- **不要忽略警告**：对版本相关的警告必须处理
- **不要硬编码版本**：使用动态检测而非硬编码版本号

## 常见问题

### 1. 如何检测项目的 Hyperf 版本？

**方法一：查看 composer.json**
```bash
cat composer.json | grep "hyperf/framework"
```

**方法二：查看 composer.lock**
```bash
composer show hyperf/framework
```

### 2. 为什么不能继续使用全局 env()？

**原因**：
- Hyperf 3.1 将 Utils 包中的全局助手函数迁移至 `hyperf/helper`
- 避免与其他 Composer 包的函数冲突
- 提高代码的可维护性和命名空间清晰度

**影响**：
- 未安装 `hyperf/helper` 的项目会报错：`Call to undefined function env()`
- 即使安装了 helper，也推荐使用命名空间版本以提高代码清晰度

### 3. Logger/Cache 配置为什么要改结构？

**原因**：
- 支持多通道/多存储配置
- 更符合现代框架的配置设计理念
- 便于动态切换不同的日志通道或缓存存储

**优势**：
```php
// 可以轻松切换不同的 channel
$logger = $loggerFactory->get('error');  // 错误日志
$logger = $loggerFactory->get('access'); // 访问日志

// 可以轻松切换不同的 store
$cache = $cacheManager->getCache('redis');  // Redis 缓存
$cache = $cacheManager->getCache('file');   // 文件缓存
```

### 4. 如何在不同版本间保持兼容？

**方案一：条件判断**
```php
use function Hyperf\Support\env;

// 如果担心兼容性问题，可以这样写
if (defined('\Hyperf\Support\env')) {
    $value = \Hyperf\Support\env('KEY');
} else {
    $value = env('KEY');  //  fallback
}
```

**方案二：统一使用新规范**
- 推荐直接采用新版本的规范
- 通过升级项目来保持一致性
- 避免维护多套代码

## 版本特性速查表

### Hyperf 3.0 → 3.1 变化

| 特性 | 3.0 | 3.1+ | 影响程度 |
|------|-----|------|---------|
| env() 函数 | ✅ 全局可用 | ❌ 需导入或使用命名空间 | 🔴 高 |
| 配置文件加载 | 文件名作为键 | 支持 . 语法 | 🟡 中 |
| AMQP 返回值 | string | Result 枚举 | 🟡 中 |
| withHeaders() | 合并 | 替换 | 🟡 中 |

### Hyperf 3.1 → 3.2 变化

| 特性 | 3.1 | 3.2+ | 影响程度 |
|------|-----|------|---------|
| Logger 配置 | 扁平结构 | channels 嵌套 | 🔴 高 |
| Cache 配置 | 扁平结构 | stores 嵌套 | 🔴 高 |
| Async Queue | getQueueName() | getPoolName() | 🟡 中 |
| Carbon 时区 | 自动读取 | 需显式传入 | 🟡 中 |
| PHP 版本 | >= 8.1 | >= 8.2 | 🔴 高 |

## AI 使用指南

### 何时应用此 Skill

1. **检测到 Hyperf 项目**：自动加载此 Skill
2. **生成配置文件**：根据版本选择正确的结构
3. **生成 PHP 代码**：使用符合版本的函数调用方式
4. **代码审查**：检查是否符合当前版本规范
5. **问题诊断**：识别版本相关的常见错误

### AI 工作流程

```
1. 检测项目使用的 Hyperf 版本
   ↓
2. 加载对应版本的开发规范
   ↓
3. 生成/审查代码时自动应用规范
   ↓
4. 发现不符合规范的代码时给出修正建议
   ↓
5. 提供版本特定的最佳实践提示
```

### 示例对话

**用户**：帮我创建一个缓存服务

**AI**：
```
检测到您的项目使用 Hyperf 3.2，我将按照 3.2 的规范生成代码：

1. Cache 配置使用 stores 嵌套结构
2. 使用 \Hyperf\Support\env() 获取环境变量
3. 支持多存储切换

[生成符合 3.2 规范的代码...]
```

## 相关资源

- [Hyperf 3.1 升级文档](https://github.com/hyperf/hyperf/blob/master/docs/zh-cn/upgrade/3.1.md)
- [Hyperf 3.2 Changelog](https://github.com/hyperf/hyperf/blob/3.2/CHANGELOG-3.2.md)
- [Hyperf 官方文档](https://hyperf.wiki/)
- [Hyperf GitHub](https://github.com/hyperf/hyperf)