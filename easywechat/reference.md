# EasyWeChat 详细参考

## ⚠️ 重要提示

**EasyWeChat 6.x + Hyperf 协程环境必须安装适配组件**：

```bash
composer require limingxinleo/easywechat-classmap
```

原因：
- EasyWeChat 6.x 使用 Symfony Http Client（基于 curl）
- Hyperf 是多协程框架，多个协程可能同时使用同一个 curl 实例
- 会导致并发冲突、响应错乱、随机报错等问题
- classmap 组件提供协程安全的 HTTP 客户端实现

**API 调用说明**：
- EasyWeChat 6.x **不再提供封装好的便捷方法**（如 `jssdk`、`oauth->user()` 等）
- 需要开发者直接调用底层 HTTP 客户端：`$app->getClient()`
- 具体接口参数请参考 [微信官方文档](https://developers.weixin.qq.com/doc/)

## 公众号开发

### 1. 创建 WeChat Service

```php
namespace App\Service;

use EasyWeChat\OfficialAccount\Application;
use Hyperf\Contract\ConfigInterface;
use Psr\Container\ContainerInterface;

class OfficialAccountService
{
    private Application $application;

    public function __construct(ContainerInterface $container)
    {
        $config = $container->get(ConfigInterface::class)->get('wechat.official_account', []);
        $this->application = new Application($config);
    }

    public function getApplication(): Application
    {
        return $this->application;
    }
}
```

### 2. 服务器配置验证

``php
namespace App\Controller;

use App\Service\OfficialAccountService;
use Hyperf\HttpServer\Annotation\AutoController;
use Hyperf\HttpServer\Contract\RequestInterface;
use Hyperf\HttpServer\Contract\ResponseInterface;

#[AutoController]
class OfficialAccountController
{
    public function __construct(
        private OfficialAccountService $officialAccountService
    ) {}

    /**
     * 微信服务器验证(GET请求)
     */
    public function verify(RequestInterface $request, ResponseInterface $response)
    {
        $app = $this->officialAccountService->getApplication();
        $server = $app->getServer();

        return $response->raw($server->serve()->getBody()->getContents());
    }

    /**
     * 接收微信消息(POST请求)
     */
    public function serve(RequestInterface $request, ResponseInterface $response)
    {
        $app = $this->officialAccountService->getApplication();
        $server = $app->getServer();

        // 处理文本消息
        $server->withText(function ($message) {
            return "收到文本: {$message['Content']}";
        });

        // 处理图片消息
        $server->withImage(function ($message) {
            return "收到图片: {$message['PicUrl']}";
        });

        // 处理事件
        $server->withEvent('subscribe', function ($message) {
            return "欢迎关注!";
        });

        return $response->raw($server->serve()->getBody()->getContents());
    }
}
```

### 3. OAuth 网页授权

**注意**: EasyWeChat 6.x 需要手动实现 OAuth 流程

```php
namespace App\Service;

use App\Service\OfficialAccountService;
use Hyperf\Codec\Json;
use Hyperf\HttpServer\Contract\RequestInterface;

class WeChatAuthService
{
    public function __construct(
        private OfficialAccountService $officialAccountService
    ) {}

    /**
     * 生成授权链接
     */
    public function getAuthUrl(string $redirectUrl): string
    {
        $app = $this->officialAccountService->getApplication();
        $appId = $app->getAccount()->getAppId();
        
        // 构造 OAuth 授权 URL
        $params = [
            'appid' => $appId,
            'redirect_uri' => urlencode($redirectUrl),
            'response_type' => 'code',
            'scope' => 'snsapi_userinfo',
            'state' => 'STATE#wechat_redirect',
        ];
        
        return 'https://open.weixin.qq.com/connect/oauth2/authorize?' . http_build_query($params);
    }

    /**
     * 通过 code 获取用户信息
     */
    public function getUserInfo(string $code): array
    {
        $app = $this->officialAccountService->getApplication();
        $client = $app->getClient();
        $account = $app->getAccount();
        
        // 1. 获取 access_token
        $response = $client->get('/sns/oauth2/access_token', [
            'query' => [
                'appid' => $account->getAppId(),
                'secret' => $account->getSecret(),
                'code' => $code,
                'grant_type' => 'authorization_code',
            ],
        ]);
        
        $tokenData = Json::decode($response->getContent());
        
        if (isset($tokenData['errcode'])) {
            throw new \RuntimeException("获取 access_token 失败: {$tokenData['errmsg']}");
        }
        
        // 2. 获取用户信息
        $response = $client->get('/sns/userinfo', [
            'query' => [
                'access_token' => $tokenData['access_token'],
                'openid' => $tokenData['openid'],
                'lang' => 'zh_CN',
            ],
        ]);
        
        return Json::decode($response->getContent());
    }
}
```

### 4. 发送模板消息

``php
namespace App\Service;

use App\Service\OfficialAccountService;
use Hyperf\Codec\Json;

class TemplateMessageService
{
    public function __construct(
        private OfficialAccountService $officialAccountService
    ) {}

    public function sendOrderNotice(string $openid, array $orderData): void
    {
        $app = $this->officialAccountService->getApplication();
        $client = $app->getClient();
        
        $response = $client->post('/cgi-bin/message/template/send', [
            'json' => [
                'touser' => $openid,
                'template_id' => 'TEMPLATE_ID_HERE',
                'url' => "https://example.com/order/{$orderData['id']}",
                'data' => [
                    'first' => ['value' => '您的订单已发货'],
                    'keyword1' => ['value' => $orderData['order_no']],
                    'keyword2' => ['value' => $orderData['product_name']],
                    'keyword3' => ['value' => date('Y-m-d H:i:s')],
                    'remark' => ['value' => '如有疑问请联系客服'],
                ],
            ],
        ]);
        
        $result = Json::decode($response->getContent());
        
        if ($result['errcode'] !== 0) {
            throw new \RuntimeException("发送失败: {$result['errmsg']}");
        }
    }
}
```

### 5. 菜单管理

```php
namespace App\Service;

use App\Service\OfficialAccountService;
use Hyperf\Codec\Json;

class MenuService
{
    public function __construct(
        private OfficialAccountService $officialAccountService
    ) {}

    public function createMenu(): void
    {
        $app = $this->officialAccountService->getApplication();
        $client = $app->getClient();
        
        $response = $client->post('/cgi-bin/menu/create', [
            'json' => [
                'button' => [
                    [
                        'type' => 'view',
                        'name' => '官网',
                        'url' => 'https://example.com',
                    ],
                    [
                        'type' => 'click',
                        'name' => '联系我们',
                        'key' => 'CONTACT_US',
                    ],
                    [
                        'name' => '更多',
                        'sub_button' => [
                            [
                                'type' => 'view',
                                'name' => '帮助中心',
                                'url' => 'https://example.com/help',
                            ],
                        ],
                    ],
                ],
            ],
        ]);
        
        $result = Json::decode($response->getContent());
        
        if ($result['errcode'] !== 0) {
            throw new \RuntimeException("创建菜单失败: {$result['errmsg']}");
        }
    }

    public function deleteMenu(): void
    {
        $app = $this->officialAccountService->getApplication();
        $client = $app->getClient();
        
        $response = $client->get('/cgi-bin/menu/delete');
        $result = Json::decode($response->getContent());
        
        if ($result['errcode'] !== 0) {
            throw new \RuntimeException("删除菜单失败: {$result['errmsg']}");
        }
    }
}
```

## 小程序开发

### 1. 创建 WeChat Service

```php
namespace App\Service;

use EasyWeChat\MiniProgram\Application;
use Hyperf\Contract\ConfigInterface;
use Psr\Container\ContainerInterface;

class MiniProgramService
{
    private Application $application;

    public function __construct(ContainerInterface $container)
    {
        $config = $container->get(ConfigInterface::class)->get('wechat.mini_program', []);
        $this->application = new Application($config);
    }

    public function getApplication(): Application
    {
        return $this->application;
    }
}
```

### 2. 登录获取 OpenID

```php
namespace App\Service;

use App\Service\MiniProgramService;
use Hyperf\Codec\Json;

class MiniProgramAuthService
{
    public function __construct(
        private MiniProgramService $miniProgramService
    ) {}

    public function login(string $code): array
    {
        $app = $this->miniProgramService->getApplication();
        $client = $app->getClient();
        
        $response = $client->get('/sns/jscode2session', [
            'query' => [
                'appid' => $app->getAccount()->getAppId(),
                'secret' => $app->getAccount()->getSecret(),
                'js_code' => $code,
                'grant_type' => 'authorization_code',
            ],
        ]);

        $result = Json::decode($response->getContent());
        
        if (isset($result['errcode'])) {
            throw new \RuntimeException("登录失败: {$result['errmsg']}");
        }

        return [
            'openid' => $result['openid'],
            'session_key' => $result['session_key'],
            'unionid' => $result['unionid'] ?? null,
        ];
    }
}
```

### 3. 完整的微信登录流程

``php
namespace App\Service;

use App\Service\MiniProgramService;
use Hyperf\Codec\Json;

class WeChatLoginService
{
    public function __construct(
        private MiniProgramService $miniProgramService
    ) {}

    public function miniProgramLogin(string $code, string $encryptedData, string $iv): array
    {
        $app = $this->miniProgramService->getApplication();
        
        // 1. 获取 session
        $client = $app->getClient();
        $response = $client->get('/sns/jscode2session', [
            'query' => [
                'appid' => $app->getAccount()->getAppId(),
                'secret' => $app->getAccount()->getSecret(),
                'js_code' => $code,
                'grant_type' => 'authorization_code',
            ],
        ]);
        
        $session = Json::decode($response->getContent());

        if (isset($session['errcode'])) {
            throw new \RuntimeException("Code 无效: {$session['errmsg']}");
        }

        // 2. 解密用户信息
        $userInfo = $this->decryptSession(
            $session['session_key'], 
            $encryptedData, 
            $iv
        );

        // 3. 查找或创建用户
        $user = $this->findOrCreateUser([
            'openid' => $session['openid'],
            'unionid' => $session['unionid'] ?? null,
            'nickname' => $userInfo['nickName'] ?? '',
            'avatar' => $userInfo['avatarUrl'] ?? '',
        ]);

        // 4. 生成 Token
        $token = $this->generateToken($user['id']);

        return [
            'token' => $token,
            'user' => $user,
        ];
    }

    private function decryptSession(string $sessionKey, string $encryptedData, string $iv): array
    {
        // 使用 EasyWeChat 提供的解密方法
        $app = $this->miniProgramService->getApplication();
        return $app->getUtils()->decryptSession($sessionKey, $encryptedData, $iv);
    }

    private function findOrCreateUser(array $data): array
    {
        // 数据库操作
        return ['id' => 1, 'openid' => $data['openid']];
    }

    private function generateToken(int $userId): string
    {
        // 生成 JWT Token
        return 'jwt_token_here';
    }
}
```

### 4. 发送订阅消息

```php
namespace App\Service;

use App\Service\MiniProgramService;

class MiniProgramMessageService
{
    public function __construct(
        private MiniProgramService $miniProgramService
    ) {}

    public function sendSubscribeMessage(string $openid, string $templateId, array $data): void
    {
        $app = $this->miniProgramService->getApplication();
        $client = $app->getClient();
        
        $response = $client->post('/cgi-bin/message/subscribe/send', [
            'json' => [
                'touser' => $openid,
                'template_id' => $templateId,
                'page' => '/pages/order/detail',
                'data' => [
                    'thing1' => ['value' => '订单通知'],
                    'time2' => ['value' => date('Y-m-d H:i:s')],
                    'thing3' => ['value' => '您的订单已发货'],
                ],
            ],
        ]);
        
        $result = Json::decode($response->getContent());
        
        if ($result['errcode'] !== 0) {
            throw new \RuntimeException("发送失败: {$result['errmsg']}");
        }
    }
}
```

### 5. 生成小程序码

```php
namespace App\Service;

use App\Service\MiniProgramService;

class MiniProgramCodeService
{
    public function __construct(
        private MiniProgramService $miniProgramService
    ) {}

    public function getQrCode(string $scene, string $page = 'pages/index/index'): string
    {
        $app = $this->miniProgramService->getApplication();
        $client = $app->getClient();
        
        $response = $client->post('/wxa/getwxacodeunlimit', [
            'json' => [
                'scene' => $scene,
                'page' => $page,
                'width' => 430,
                'check_path' => false,
            ],
        ]);

        return base64_encode($response->getContent());
    }
}
```

## 微信支付

### 1. 创建 WeChat Pay Service

```php
namespace App\Service;

use EasyWeChat\Pay\Application;
use Hyperf\Contract\ConfigInterface;
use Psr\Container\ContainerInterface;

class WeChatPayService
{
    private Application $application;

    public function __construct(ContainerInterface $container)
    {
        $config = $container->get(ConfigInterface::class)->get('wechat.pay', []);
        $this->application = new Application($config);
    }

    public function getApplication(): Application
    {
        return $this->application;
    }
}
```

### 2. JSAPI 支付(公众号/小程序)

```php
namespace App\Service;

use App\Service\WeChatPayService;
use Hyperf\Codec\Json;

class PayOrderService
{
    public function __construct(
        private WeChatPayService $payService
    ) {}

    public function createJsApiOrder(string $openid, string $outTradeNo, int $totalAmount, string $description): array
    {
        $app = $this->payService->getApplication();
        $client = $app->getClient();
        
        // 创建订单
        $response = $client->post('/v3/pay/transactions/jsapi', [
            'json' => [
                'appid' => $app->getAccount()->getAppId(),
                'mchid' => $app->getMerchant()->getMerchantId(),
                'description' => $description,
                'out_trade_no' => $outTradeNo,
                'notify_url' => 'https://example.com/pay/notify',
                'amount' => [
                    'total' => $totalAmount, // 单位:分
                    'currency' => 'CNY',
                ],
                'payer' => [
                    'openid' => $openid,
                ],
            ],
        ]);
        
        $order = Json::decode($response->getContent());
        
        // 获取前端调起支付的参数（需要签名）
        $prepayId = $order['prepay_id'];
        $jsapiConfig = $this->buildJsApiConfig($prepayId);
        
        return [
            'prepay_id' => $prepayId,
            'jsapi_config' => $jsapiConfig,
        ];
    }
    
    private function buildJsApiConfig(string $prepayId): array
    {
        $app = $this->payService->getApplication();
        
        // 使用 EasyWeChat 提供的签名工具
        $utils = $app->getUtils();
        
        return $utils->buildBridgeConfig($prepayId);
    }
}
```

### 3. 支付回调处理

``php
namespace App\Controller;

use App\Service\WeChatPayService;
use Hyperf\HttpServer\Annotation\AutoController;
use Hyperf\HttpServer\Contract\RequestInterface;
use Hyperf\HttpServer\Contract\ResponseInterface;
use Psr\Log\LoggerInterface;

#[AutoController]
class PayNotifyController
{
    public function __construct(
        private WeChatPayService $payService,
        private LoggerInterface $logger
    ) {}

    public function notify(RequestInterface $request, ResponseInterface $response)
    {
        $app = $this->payService->getApplication();
        $server = $app->getServer();

        try {
            // 处理支付成功回调
            $result = $server->handlePaidNotify(function ($message, $fail) {
                if ($message['trade_state'] === 'SUCCESS') {
                    $outTradeNo = $message['out_trade_no'];
                    $transactionId = $message['transaction_id'];

                    // 更新订单状态
                    $this->updateOrderStatus($outTradeNo, $transactionId);

                    $this->logger->info("支付成功: out_trade_no={$outTradeNo}");
                    return true;
                }

                $fail('支付失败');
                return false;
            });

            return $response->json([
                'code' => 'SUCCESS',
                'message' => '成功',
            ]);
        } catch (\Exception $e) {
            $this->logger->error("支付回调异常: {$e->getMessage()}");
            return $response->json([
                'code' => 'FAIL',
                'message' => $e->getMessage(),
            ], 500);
        }
    }

    private function updateOrderStatus(string $outTradeNo, string $transactionId): void
    {
        // 更新订单逻辑
    }
}
```

### 4. 申请退款

```php
namespace App\Service;

use App\Service\WeChatPayService;
use Hyperf\Codec\Json;

class WeChatRefundService
{
    public function __construct(
        private WeChatPayService $payService
    ) {}

    public function refund(string $transactionId, string $outRefundNo, int $totalAmount, int $refundAmount): array
    {
        $app = $this->payService->getApplication();
        $client = $app->getClient();
        
        $response = $client->post('/v3/refund/domestic/refunds', [
            'json' => [
                'transaction_id' => $transactionId,
                'out_refund_no' => $outRefundNo,
                'amount' => [
                    'refund' => $refundAmount,
                    'total' => $totalAmount,
                    'currency' => 'CNY',
                ],
                'reason' => '用户申请退款',
            ],
        ]);
        
        return Json::decode($response->getContent());
    }
}
```

## 通过 Listener 注册为容器服务

适用于需要将 EasyWeChat 实例直接注册到容器的场景。**容器注册的实例也是单例的**。

```php
namespace App\Listener;

use App\Service\WeChatService;
use Hyperf\Event\Annotation\Listener;
use Hyperf\Framework\Event\BootApplication;
use Psr\Container\ContainerInterface;
use Psr\EventDispatcher\ListenerProviderInterface;

#[Listener]
class WeChatServiceRegisterListener implements ListenerProviderInterface
{
    public function __construct(
        private ContainerInterface $container,
        private WeChatService $weChatService
    ) {}

    public function listen(): array
    {
        return [BootApplication::class];
    }

    public function process(object $event): void
    {
        // 将 Service 注册到容器(单例)
        // 注意: 这会在应用启动时就创建实例
        $this->container->set(
            'wechat.service',
            $this->weChatService
        );
        
        // 使用方式
        // $service = $this->container->get('wechat.service');
        // $app = $service->getMiniProgram();
    }
}
```

### Factory/Service 模式 vs 容器注册对比

| 特性 | Service 模式 | 容器注册 |
|------|-------------|---------|
| **单例保证** | ✅ 是 | ✅ 是 |
| **类型安全** | ✅ IDE 自动补全 | ❌ 返回 mixed |
| **延迟初始化** | ✅ 可选 | ❌ 启动时即创建 |
| **依赖注入** | ✅ 类型明确的依赖 | ⚠️ 需要字符串 key |
| **代码可读性** | ✅ 更清晰 | ⚠️ 较隐式 |

**推荐**: 优先使用 Service 模式，直接在 Controller/Service 中注入，除非有特殊需求（如需要在其他地方通过容器获取）。

## 重要提醒

### EasyWeChat 6.x API 调用规范

1. **不再提供封装好的便捷方法**
   - ❌ 旧版本：`$app->jssdk->buildConfig()`
   - ❌ 旧版本：`$app->oauth->user()`
   - ✅ 新版本：直接调用 HTTP 客户端

2. **标准调用流程**
   ```php
   // 1. 获取应用实例
   $app = $service->getMiniProgram();
   
   // 2. 获取 HTTP 客户端
   $client = $app->getClient();
   
   // 3. 调用微信 API
   $response = $client->get('/sns/jscode2session', [
       'query' => [
           'appid' => $app->getAccount()->getAppId(),
           'secret' => $app->getAccount()->getSecret(),
           // ... 其他参数
       ],
   ]);
   
   // 4. 解析响应
   $result = Json::decode($response->getContent());
   ```

3. **常用工具方法**
   - `$app->getAccount()` - 获取账号配置（AppID、Secret 等）
   - `$app->getClient()` - 获取 HTTP 客户端
   - `$app->getServer()` - 获取消息服务器（公众号）
   - `$app->getUtils()` - 获取工具类（签名、解密等）
   - `$app->getMerchant()` - 获取商户配置（微信支付）

4. **参考文档**
   - [微信官方文档](https://developers.weixin.qq.com/doc/)
   - [EasyWeChat 6.x 文档](https://easywechat.com/)
   - 具体接口参数请以微信官方文档为准
