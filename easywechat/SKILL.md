---
name: easywechat
description: w7corp/easywechat 微信开发 SDK 使用指南,涵盖公众号、小程序、企业微信、微信支付等平台接口开发。包含 Hyperf 框架集成、消息处理、OAuth 授权、支付流程等。使用当开发微信相关功能、处理微信回调、实现微信登录或支付时。
---

# EasyWeChat 微信开发指南

## 核心概念

EasyWeChat 是 PHP 微信开发 SDK，支持公众号、小程序、企业微信、微信支付等。

**当前版本**: 6.x（使用 Symfony Http Client）

### ⚠️ Hyperf 协程环境注意事项

由于 EasyWeChat 6.x 使用 Symfony Http Client（基于 curl），在 Hyperf 多协程环境下会产生并发冲突。**必须安装协程适配组件**：

```bash
composer require limingxinleo/easywechat-classmap
```

该组件通过 Class Map 机制将 HTTP 客户端替换为协程安全实现，无需修改代码即可使用。

### 支持的平台

| 平台     | 命名空间                     | 用途                           |
| -------- | ---------------------------- | ------------------------------ |
| 公众号   | `EasyWeChat\OfficialAccount` | 消息推送、菜单、素材、用户管理 |
| 小程序   | `EasyWeChat\MiniProgram`     | 登录、模板消息、订阅消息       |
| 企业微信 | `EasyWeChat\Work`            | 通讯录、消息推送、审批         |
| 微信支付 | `EasyWeChat\Pay`             | 下单、退款、回调               |
| 开放平台 | `EasyWeChat\OpenPlatform`    | 第三方平台代开发               |

## 安装配置

```bash
composer require w7corp/easywechat
```

### ⚠️ Hyperf 协程环境必需组件

**EasyWeChat 6.x 使用 Symfony Http Client，在 Hyperf 多协程环境下会产生并发冲突。**

必须安装以下组件来适配 Hyperf 协程：

```bash
composer require limingxinleo/easywechat-classmap
```

该组件提供了：

- **协程安全的 HTTP 客户端**：将 Symfony Http Client 适配为 Hyperf 协程兼容
- **自动 Class Map**：无需额外配置，安装即用
- **解决并发问题**：避免多个协程共享 curl 句柄导致的错误

配置文件 `config/autoload/wechat.php`:

```php
return [
    'official_account' => [
        'app_id' => env('WECHAT_OFFICIAL_ACCOUNT_APPID'),
        'secret' => env('WECHAT_OFFICIAL_ACCOUNT_SECRET'),
        'token' => env('WECHAT_OFFICIAL_ACCOUNT_TOKEN'),
        'aes_key' => env('WECHAT_OFFICIAL_ACCOUNT_AES_KEY'),
    ],
    'mini_program' => [
        'app_id' => env('WECHAT_MINI_PROGRAM_APPID'),
        'secret' => env('WECHAT_MINI_PROGRAM_SECRET'),
    ],
    'pay' => [
        'default_merchant' => [
            'mch_id' => env('WECHAT_PAY_MCH_ID'),
            'key' => env('WECHAT_PAY_KEY'),
            'cert_path' => env('WECHAT_PAY_CERT_PATH'),
            'key_path' => env('WECHAT_PAY_KEY_PATH'),
        ],
    ],
];
```

## Hyperf 集成(推荐方式)

### 创建 WeChat Factory(单例模式)

**重要**: EasyWeChat 实例存在大量循环依赖，必须使用单例模式避免内存泄露。

```php
namespace App\Factory;

use EasyWeChat\MiniProgram\Application as MiniProgramApp;
use EasyWeChat\OfficialAccount\Application as OfficialAccountApp;
use EasyWeChat\Pay\Application as PayApp;
use Hyperf\Contract\ConfigInterface;
use Psr\Container\ContainerInterface;

class WeChatFactory
{
    private array $instances = [];

    public function __construct(
        private ContainerInterface $container,
        private ConfigInterface $config
    ) {}

    /**
     * 获取公众号实例(单例)
     */
    public function officialAccount(): OfficialAccountApp
    {
        if (!isset($this->instances['official_account'])) {
            $this->instances['official_account'] = new OfficialAccountApp(
                $this->config->get('wechat.official_account')
            );
        }

        return $this->instances['official_account'];
    }

    /**
     * 获取小程序实例(单例)
     */
    public function miniProgram(): MiniProgramApp
    {
        if (!isset($this->instances['mini_program'])) {
            $this->instances['mini_program'] = new MiniProgramApp(
                $this->config->get('wechat.mini_program')
            );
        }

        return $this->instances['mini_program'];
    }

    /**
     * 获取支付实例(单例)
     */
    public function pay(): PayApp
    {
        if (!isset($this->instances['pay'])) {
            $this->instances['pay'] = new PayApp(
                $this->config->get('wechat.pay')
            );
        }

        return $this->instances['pay'];
    }
}
```

### 在 Controller/Service 中使用

```php
namespace App\Controller;

use App\Factory\WeChatFactory;
use Hyperf\HttpServer\Annotation\AutoController;

#[AutoController]
class WeChatController
{
    public function __construct(
        private WeChatFactory $weChatFactory
    ) {}

    public function getJsConfig(string $url)
    {
        // 每次调用都返回同一个实例,不会造成内存泄露
        $app = $this->weChatFactory->officialAccount();
        return $app->jssdk->buildConfig(['chooseImage'], false, false, false);
    }
}
```

## 快速开始

### 公众号开发要点

1. **服务器验证**: 处理 GET 请求验证 Token
2. **消息接收**: 处理 POST 请求接收微信消息
3. **OAuth 授权**: 实现网页授权获取用户信息
4. **模板消息**: 发送业务通知

详细示例见 [reference.md](reference.md) 的"公众号开发"章节。

### 小程序开发要点

1. **登录流程**: code → session → openid
2. **订阅消息**: 发送模板化消息
3. **小程序码**: 生成带参二维码

详细示例见 [reference.md](reference.md) 的"小程序开发"章节。

### 微信支付要点

1. **JSAPI 支付**: 公众号/小程序内支付
2. **回调处理**: 验签并更新订单状态
3. **申请退款**: 调用退款接口

详细示例见 [reference.md](reference.md) 的"微信支付"章节。

## 最佳实践

### DO ✓

- **必须安装协程适配组件**：`composer require limingxinleo/easywechat-classmap`，解决 EasyWeChat 6.x 在 Hyperf 多协程环境下的并发冲突问题
- **使用单例模式创建实例**,避免内存泄露。EasyWeChat 存在大量循环依赖,每次 new 都会导致内存无法完全释放
- **使用 Factory 模式**管理单例,便于统一维护和测试
- **敏感配置使用环境变量**,不要硬编码
- **支付回调必须验签**,确保请求来自微信
- **记录关键日志**,特别是支付、登录流程
- **使用 HTTPS**,微信接口要求安全传输

### DON'T ✗

- **不要每次调用都 new 新实例**,会导致严重的内存泄露
- **不要在未安装 classmap 组件的情况下使用**，会导致多协程并发错误
- **不要硬编码 AppID/Secret**
- **不要忽略错误码**,检查 API 返回的 errcode
- **不要在回调中执行耗时操作**,先返回成功再异步处理
- **不要缓存 access_token 到文件**,Hyperf 中应使用 Redis
- **不要在前端暴露 Secret 或 Key**

## 常见问题

### 1. 为什么必须使用单例模式?

EasyWeChat 内部存在大量循环依赖(如 Application、Container、Service Provider 之间的相互引用)。如果每次调用接口都 `new` 一个新实例：

- **内存泄露**: PHP 的垃圾回收机制无法完全释放循环引用的对象
- **性能下降**: 频繁创建和销毁大型对象消耗资源
- **连接池浪费**: 每次创建都会建立新的 HTTP 客户端连接

**解决方案**: 使用单例模式，整个应用生命周期中只创建一个实例。

### 2. Hyperf 协程环境下的并发问题

**问题描述**：
EasyWeChat 6.x 使用 Symfony Http Client，底层基于 curl。在 Hyperf 的多协程环境下，如果多个协程同时使用同一个 EasyWeChat 实例发送请求，会导致：
- curl 句柄冲突
- 响应数据错乱
- 随机性报错（如 "Connection reset"、"SSL handshake failed" 等）

**解决方案**：
安装协程适配组件：
```bash
composer require limingxinleo/easywechat-classmap
```

该组件会自动将 Symfony Http Client 替换为 Hyperf 协程安全的实现，无需修改任何代码。

**原理说明**：
- 通过 Class Map 机制，将 `Symfony\Component\HttpClient\CurlHttpClient` 映射到协程安全的实现
- 每个协程使用独立的 curl 句柄，避免并发冲突
- 保持单例模式的优势，同时解决协程安全问题

### 3. Access Token 缓存

EasyWeChat 默认使用文件缓存,在 Hyperf 中应改为 Redis:

```php
use EasyWeChat\Core\AccessToken;
use Hyperf\Redis\Redis;

$app->rebind('access_token', function () use ($redis) {
    return new AccessToken($appId, $appSecret, $redis);
});
```

### 4. 签名验证失败

检查:
- AppID 和 Secret 是否正确
- Token 是否与微信公众平台配置一致
- 服务器时间是否同步

### 5. 支付回调收不到

- 确认 notify_url 可公网访问
- 确认未设置 IP 白名单限制
- 检查服务器防火墙规则

## 完整示例参考

对于详细的代码示例,包括:

- 公众号消息处理完整实现
- OAuth 授权流程
- 小程序登录完整流程
- 支付下单和回调处理
- 退款流程

详见 [reference.md](reference.md)。

## 相关资源

- EasyWeChat 官方文档: https://easywechat.com/
- 微信公众平台文档: https://developers.weixin.qq.com/doc/
- 微信支付文档: https://pay.weixin.qq.com/wiki/doc/apiv3/
