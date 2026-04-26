---
name: easywechat
description: w7corp/easywechat 微信开发 SDK 使用指南,涵盖公众号、小程序、企业微信、微信支付等平台接口开发。包含 Hyperf 框架集成、消息处理、OAuth 授权、支付流程等。使用当开发微信相关功能、处理微信回调、实现微信登录或支付时。
---

# EasyWeChat 微信开发指南

## 核心概念

EasyWeChat 是 PHP 微信开发 SDK,支持公众号、小程序、企业微信、微信支付等。

### 支持的平台

| 平台 | 命名空间 | 用途 |
|------|---------|------|
| 公众号 | `EasyWeChat\OfficialAccount` | 消息推送、菜单、素材、用户管理 |
| 小程序 | `EasyWeChat\MiniProgram` | 登录、模板消息、订阅消息 |
| 企业微信 | `EasyWeChat\Work` | 通讯录、消息推送、审批 |
| 微信支付 | `EasyWeChat\Pay` | 下单、退款、回调 |
| 开放平台 | `EasyWeChat\OpenPlatform` | 第三方平台代开发 |

## 安装配置

```bash
composer require w7corp/easywechat
```

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

### 创建 WeChat Factory

```php
namespace App\Factory;

use EasyWeChat\MiniProgram\Application as MiniProgramApp;
use EasyWeChat\OfficialAccount\Application as OfficialAccountApp;
use EasyWeChat\Pay\Application as PayApp;
use Hyperf\Contract\ConfigInterface;
use Psr\Container\ContainerInterface;

class WeChatFactory
{
    public function __construct(
        private ContainerInterface $container,
        private ConfigInterface $config
    ) {}

    public function officialAccount(): OfficialAccountApp
    {
        return new OfficialAccountApp($this->config->get('wechat.official_account'));
    }

    public function miniProgram(): MiniProgramApp
    {
        return new MiniProgramApp($this->config->get('wechat.mini_program'));
    }

    public function pay(): PayApp
    {
        return new PayApp($this->config->get('wechat.pay'));
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

- **使用 Factory 模式**创建实例,便于管理和测试
- **敏感配置使用环境变量**,不要硬编码
- **支付回调必须验签**,确保请求来自微信
- **记录关键日志**,特别是支付、登录流程
- **使用 HTTPS**,微信接口要求安全传输

### DON'T ✗

- **不要硬编码 AppID/Secret**
- **不要忽略错误码**,检查 API 返回的 errcode
- **不要在回调中执行耗时操作**,先返回成功再异步处理
- **不要缓存 access_token 到文件**,Hyperf 中应使用 Redis
- **不要在前端暴露 Secret 或 Key**

## 常见问题

### 1. Access Token 缓存

EasyWeChat 默认使用文件缓存,在 Hyperf 中应改为 Redis:

```php
use EasyWeChat\Core\AccessToken;
use Hyperf\Redis\Redis;

$app->rebind('access_token', function () use ($redis) {
    return new AccessToken($appId, $appSecret, $redis);
});
```

### 2. 签名验证失败

检查:
- AppID 和 Secret 是否正确
- Token 是否与微信公众平台配置一致
- 服务器时间是否同步

### 3. 支付回调收不到

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
