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

## 公众号开发

### 1. 服务器配置验证

```php
namespace App\Controller;

use App\Factory\WeChatFactory;
use Hyperf\HttpServer\Annotation\AutoController;
use Hyperf\HttpServer\Contract\RequestInterface;
use Hyperf\HttpServer\Contract\ResponseInterface;

#[AutoController]
class OfficialAccountController
{
    public function __construct(
        private WeChatFactory $weChatFactory
    ) {}

    /**
     * 微信服务器验证(GET请求)
     */
    public function verify(RequestInterface $request, ResponseInterface $response)
    {
        $app = $this->weChatFactory->officialAccount();
        $server = $app->getServer();

        return $response->raw($server->serve()->getBody()->getContents());
    }

    /**
     * 接收微信消息(POST请求)
     */
    public function serve(RequestInterface $request, ResponseInterface $response)
    {
        $app = $this->weChatFactory->officialAccount();
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

### 2. OAuth 网页授权

```php
namespace App\Service;

use App\Factory\WeChatFactory;

class WeChatAuthService
{
    public function __construct(
        private WeChatFactory $weChatFactory
    ) {}

    /**
     * 生成授权链接
     */
    public function getAuthUrl(string $redirectUrl): string
    {
        $app = $this->weChatFactory->officialAccount();
        return $app->oauth->scopes(['snsapi_userinfo'])->redirect($redirectUrl);
    }

    /**
     * 获取用户信息
     */
    public function getUserInfo(): array
    {
        $app = $this->weChatFactory->officialAccount();
        $user = $app->oauth->user();

        return [
            'openid' => $user->getId(),
            'nickname' => $user->getNickname(),
            'avatar' => $user->getAvatar(),
            'original' => $user->getOriginal(),
        ];
    }
}
```

### 3. 发送模板消息

```php
namespace App\Service;

use App\Factory\WeChatFactory;

class TemplateMessageService
{
    public function __construct(
        private WeChatFactory $weChatFactory
    ) {}

    public function sendOrderNotice(string $openid, array $orderData): void
    {
        $app = $this->weChatFactory->officialAccount();

        $app->template_message->send([
            'touser' => $openid,
            'template_id' => 'TEMPLATE_ID_HERE',
            'url' => "https://example.com/order/{$orderData['id']}",
            'data' => [
                'first' => '您的订单已发货',
                'keyword1' => $orderData['order_no'],
                'keyword2' => $orderData['product_name'],
                'keyword3' => date('Y-m-d H:i:s'),
                'remark' => '如有疑问请联系客服',
            ],
        ]);
    }
}
```

### 4. 菜单管理

```php
namespace App\Service;

use App\Factory\WeChatFactory;

class MenuService
{
    public function __construct(
        private WeChatFactory $weChatFactory
    ) {}

    public function createMenu(): void
    {
        $app = $this->weChatFactory->officialAccount();

        $app->menu->create([
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
        ]);
    }

    public function deleteMenu(): void
    {
        $app = $this->weChatFactory->officialAccount();
        $app->menu->delete();
    }
}
```

## 小程序开发

### 1. 登录获取 OpenID

```php
namespace App\Service;

use App\Factory\WeChatFactory;

class MiniProgramAuthService
{
    public function __construct(
        private WeChatFactory $weChatFactory
    ) {}

    public function login(string $code): array
    {
        $app = $this->weChatFactory->miniProgram();
        $result = $app->auth->session($code);

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

### 2. 完整的微信登录流程

```php
namespace App\Service;

use App\Factory\WeChatFactory;

class WeChatLoginService
{
    public function __construct(
        private WeChatFactory $weChatFactory
    ) {}

    public function miniProgramLogin(string $code, string $encryptedData, string $iv): array
    {
        // 1. 获取 session
        $auth = $this->weChatFactory->miniProgram()->auth;
        $session = $auth->session($code);

        if (isset($session['errcode'])) {
            throw new \RuntimeException("Code 无效: {$session['errmsg']}");
        }

        // 2. 解密用户信息
        $userInfo = $auth->decryptSession($session['session_key'], $encryptedData, $iv);

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

### 3. 发送订阅消息

```php
namespace App\Service;

use App\Factory\WeChatFactory;

class MiniProgramMessageService
{
    public function __construct(
        private WeChatFactory $weChatFactory
    ) {}

    public function sendSubscribeMessage(string $openid, string $templateId, array $data): void
    {
        $app = $this->weChatFactory->miniProgram();

        $app->subscribe_message->send([
            'touser' => $openid,
            'template_id' => $templateId,
            'page' => '/pages/order/detail',
            'data' => [
                'thing1' => ['value' => '订单通知'],
                'time2' => ['value' => date('Y-m-d H:i:s')],
                'thing3' => ['value' => '您的订单已发货'],
            ],
        ]);
    }
}
```

### 4. 生成小程序码

```php
namespace App\Service;

use App\Factory\WeChatFactory;

class MiniProgramCodeService
{
    public function __construct(
        private WeChatFactory $weChatFactory
    ) {}

    public function getQrCode(string $scene, string $page = 'pages/index/index'): string
    {
        $app = $this->weChatFactory->miniProgram();

        $response = $app->app_code->getUnlimit($scene, [
            'page' => $page,
            'width' => 430,
            'check_path' => false,
        ]);

        return base64_encode($response->getBody()->getContents());
    }
}
```

## 微信支付

### 1. JSAPI 支付(公众号/小程序)

```php
namespace App\Service;

use App\Factory\WeChatFactory;

class WeChatPayService
{
    public function __construct(
        private WeChatFactory $weChatFactory
    ) {}

    public function createJsApiOrder(string $openid, string $outTradeNo, int $totalAmount, string $description): array
    {
        $app = $this->weChatFactory->pay();

        $order = $app->order->create([
            'appid' => $app->getConfig()->get('app_id'),
            'mchid' => $app->getConfig()->get('mch_id'),
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
        ]);

        // 获取前端调起支付的参数
        $jsapiConfig = $app->bridge->order($order['prepay_id']);

        return [
            'prepay_id' => $order['prepay_id'],
            'jsapi_config' => $jsapiConfig,
        ];
    }
}
```

### 2. 支付回调处理

```php
namespace App\Controller;

use App\Factory\WeChatFactory;
use Hyperf\HttpServer\Annotation\AutoController;
use Hyperf\HttpServer\Contract\RequestInterface;
use Hyperf\HttpServer\Contract\ResponseInterface;
use Psr\Log\LoggerInterface;

#[AutoController]
class PayNotifyController
{
    public function __construct(
        private WeChatFactory $weChatFactory,
        private LoggerInterface $logger
    ) {}

    public function notify(RequestInterface $request, ResponseInterface $response)
    {
        $app = $this->weChatFactory->pay();

        try {
            $message = $app->getServer()->handlePaidNotify(function ($message, $fail) {
                if ($message['return_code'] === 'SUCCESS' && $message['result_code'] === 'SUCCESS') {
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

            return $response->json($message);
        } catch (\Exception $e) {
            $this->logger->error("支付回调异常: {$e->getMessage()}");
            return $response->json(['return_code' => 'FAIL', 'return_msg' => $e->getMessage()]);
        }
    }

    private function updateOrderStatus(string $outTradeNo, string $transactionId): void
    {
        // 更新订单逻辑
    }
}
```

### 3. 申请退款

```php
namespace App\Service;

use App\Factory\WeChatFactory;

class WeChatRefundService
{
    public function __construct(
        private WeChatFactory $weChatFactory
    ) {}

    public function refund(string $transactionId, string $outRefundNo, int $totalAmount, int $refundAmount): array
    {
        $app = $this->weChatFactory->pay();

        return $app->refund->create([
            'transaction_id' => $transactionId,
            'out_refund_no' => $outRefundNo,
            'amount' => [
                'refund' => $refundAmount,
                'total' => $totalAmount,
                'currency' => 'CNY',
            ],
            'reason' => '用户申请退款',
        ]);
    }
}
```

## 通过 Listener 注册为容器服务

适用于需要将 EasyWeChat 实例直接注册到容器的场景。**容器注册的实例也是单例的**。

```php
namespace App\Listener;

use App\Factory\WeChatFactory;
use Hyperf\Event\Annotation\Listener;
use Hyperf\Framework\Event\BootApplication;
use Psr\Container\ContainerInterface;
use Psr\EventDispatcher\ListenerProviderInterface;

#[Listener]
class WeChatServiceRegisterListener implements ListenerProviderInterface
{
    public function __construct(
        private ContainerInterface $container,
        private WeChatFactory $weChatFactory
    ) {}

    public function listen(): array
    {
        return [BootApplication::class];
    }

    public function process(object $event): void
    {
        // 将公众号实例注册到容器(单例)
        // 注意: 这会在应用启动时就创建实例
        $this->container->set(
            'wechat.official_account',
            $this->weChatFactory->officialAccount()
        );
        
        // 使用方式
        // $app = $this->container->get('wechat.official_account');
    }
}
```

### Factory 模式 vs 容器注册对比

| 特性 | Factory 模式 | 容器注册 |
|------|-------------|---------|
| **单例保证** | ✅ 是 | ✅ 是 |
| **类型安全** | ✅ IDE 自动补全 | ❌ 返回 mixed |
| **延迟初始化** | ✅ 首次使用时创建 | ❌ 启动时即创建 |
| **依赖注入** | ✅ 类型明确的依赖 | ⚠️ 需要字符串 key |
| **代码可读性** | ✅ 更清晰 | ⚠️ 较隐式 |

**推荐**: 优先使用 Factory 模式，除非有特殊需求（如需要在其他地方通过容器获取）。
