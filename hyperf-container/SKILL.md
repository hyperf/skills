---
name: hyperf-container
description: Hyperf 依赖注入容器使用指南,包括服务注册、依赖获取、自动注入等基础概念。使用当处理 Hyperf 项目中的依赖注入、Container 操作、DI 配置或服务管理时。
---

# Hyperf Container 容器使用

## 核心概念

Hyperf 基于 PSR-11 标准实现依赖注入容器,是框架的核心组件。

### 基本原则

1. **面向接口编程**:优先通过接口定义依赖,而非具体实现
2. **构造函数注入**:首选的依赖注入方式
3. **自动装配**:利用 PHP 反射机制自动解析依赖
4. **懒加载**:服务在首次使用时才实例化

## 基础用法

### 1. 获取 Container 实例

```php
use Hyperf\Utils\ApplicationContext;

// 方式一:通过 ApplicationContext(推荐)
$container = ApplicationContext::getContainer();

// 方式二:在 Controller/Service 中直接注入
public function __construct(
    private \Psr\Container\ContainerInterface $container
) {}
```

### 2. 从容器获取服务

```php
// 直接获取实例
$userService = $container->get(App\Service\UserService::class);

// 判断服务是否存在
if ($container->has(App\Service\UserService::class)) {
    $userService = $container->get(App\Service\UserService::class);
}
```

### 3. 构造函数自动注入(推荐)

```php
namespace App\Controller;

use App\Service\UserService;
use Hyperf\HttpServer\Annotation\AutoController;

#[AutoController]
class UserController
{
    public function __construct(
        private UserService $userService
    ) {}

    public function index()
    {
        // 直接使用 $this->userService
        return $this->userService->list();
    }
}
```

**优势**:
- 代码更清晰,依赖关系明确
- 便于单元测试
- 支持 IDE 自动补全和类型检查

## 服务注册

### 1. 自动扫描注册(默认)

Hyperf 会自动扫描 `App` 命名空间下的类并注册到容器,无需手动配置。

```php
// app/Service/UserService.php
namespace App\Service;

class UserService
{
    public function list(): array
    {
        return [];
    }
}
```

### 2. 通过 ConfigProvider 注册

用于第三方组件或需要自定义配置的服务:

```php
// config/autoload/dependencies.php
return [
    // 绑定接口到实现
    App\Repository\UserRepositoryInterface::class => App\Repository\UserRepository::class,

    // 单例模式(默认就是单例)
    App\Service\CacheService::class => \Hyperf\Di\Annotation\Singleton::class,
];
```

### 3. 使用 Listener 注册服务

对于需要手动注册的服务,应创建 Listener 监听 `BootApplication` 事件:

**步骤 1: 创建 Listener**

```php
namespace App\Listener;

use Hyperf\Contract\ConfigInterface;
use Hyperf\Event\Annotation\Listener;
use Hyperf\Framework\Event\BootApplication;
use Psr\Container\ContainerInterface;
use Psr\EventDispatcher\ListenerProviderInterface;

#[Listener]
class ServiceRegisterListener implements ListenerProviderInterface
{
    public function __construct(
        private ContainerInterface $container
    ) {}

    public function listen(): array
    {
        return [
            BootApplication::class,
        ];
    }

    public function process(object $event): void
    {
        // 注册自定义服务
        $this->container->set(
            'custom.service',
            new \App\Service\CustomService(
                $this->container->get(ConfigInterface::class)
            )
        );
    }
}
```

**关键点**:
- 必须监听 `Hyperf\Framework\Event\BootApplication` 事件
- 使用 `#[Listener]` 注解让 Hyperf 自动发现
- 在 `process()` 方法中执行服务注册逻辑

### 4. 使用 Factory 工厂模式

当需要动态创建对象或控制实例化过程时,使用 Factory:

**步骤 1: 定义接口**

```php
namespace App\Service;

interface OrderServiceInterface
{
    public function create(array $data): int;
    public function findById(int $id): ?array;
}
```

**步骤 2: 实现接口**

```php
namespace App\Service;

use App\Repository\OrderRepository;
use Hyperf\DbConnection\Db;

class OrderService implements OrderServiceInterface
{
    public function __construct(
        private OrderRepository $orderRepository,
        private int $timeout,
        private Db $db
    ) {}

    public function create(array $data): int
    {
        return $this->orderRepository->insert($data);
    }

    public function findById(int $id): ?array
    {
        return $this->orderRepository->find($id);
    }
}
```

**步骤 3: 创建 Factory 类**

```php
namespace App\Factory;

use App\Service\OrderService;
use App\Service\OrderServiceInterface;
use Hyperf\Contract\ConfigInterface;
use Hyperf\DbConnection\Db;
use Psr\Container\ContainerInterface;

class OrderServiceFactory
{
    public function __invoke(ContainerInterface $container): OrderServiceInterface
    {
        // 可以在此处进行复杂的初始化逻辑
        $config = $container->get(ConfigInterface::class);
        $db = $container->get(Db::class);

        return new OrderService(
            $container->get(\App\Repository\OrderRepository::class),
            $config->get('app.order_timeout', 30),
            $db
        );
    }
}
```

**步骤 4: 在 dependencies.php 中绑定接口到 Factory**

```php
// config/autoload/dependencies.php
return [
    // 将接口绑定到 Factory,避免循环依赖
    App\Service\OrderServiceInterface::class => App\Factory\OrderServiceFactory::class,
];
```

**使用示例**:

```php
namespace App\Controller;

use App\Service\OrderServiceInterface;
use Hyperf\HttpServer\Annotation\AutoController;

#[AutoController]
class OrderController
{
    public function __construct(
        private OrderServiceInterface $orderService
    ) {}

    public function create()
    {
        $id = $this->orderService->create(['product_id' => 1]);
        return ['id' => $id];
    }
}
```

**适用场景**:
- 需要根据配置动态创建不同实现
- 构造过程复杂,需要额外初始化逻辑
- 需要控制对象的生命周期
- 第三方库的类无法直接通过构造函数注入

**关键注意**:
- **必须绑定到接口**,而非具体实现类,否则会导致循环依赖
- Factory 返回类型应与绑定的接口类型一致

## 依赖注入方式对比

| 方式 | 适用场景 | 示例 |
|------|---------|------|
| 构造函数注入 | 大部分场景(推荐) | `__construct(private UserService $service)` |
| Factory 工厂 | 复杂初始化、动态创建 | 创建 Factory 类并在 dependencies.php 绑定 |
| Listener 注册 | 应用启动时注册服务 | 监听 BootApplication 事件 |
| 方法注入 | 中间件、事件监听器 | `public function handle(Context $ctx, Closure $next)` |
| 属性注入 | 不推荐,仅用于特殊情况 | 使用 `@Inject` 注解 |
| 容器获取 | 无法使用注入的场景 | `$container->get(Service::class)` |

## 常见场景

### 场景 1: Service 层调用 Repository

```php
namespace App\Service;

use App\Repository\UserRepository;

class UserService
{
    public function __construct(
        private UserRepository $userRepository
    ) {}

    public function findById(int $id): ?array
    {
        return $this->userRepository->find($id);
    }
}
```

### 场景 2: 多个依赖注入

```php
namespace App\Service;

use App\Repository\UserRepository;
use App\Repository\OrderRepository;
use Hyperf\DbConnection\Db;

class OrderService
{
    public function __construct(
        private UserRepository $userRepository,
        private OrderRepository $orderRepository,
        private Db $db
    ) {}
}
```

### 场景 3: 循环依赖解决

当 A 依赖 B,B 又依赖 A 时:

```php
// 方案一:使用容器延迟获取
class ServiceA
{
    private ?ServiceB $serviceB = null;

    public function __construct(
        private \Psr\Container\ContainerInterface $container
    ) {}

    private function getServiceB(): ServiceB
    {
        if (!$this->serviceB) {
            $this->serviceB = $this->container->get(ServiceB::class);
        }
        return $this->serviceB;
    }
}

// 方案二:重构代码,提取共同依赖到第三个服务
```

## 最佳实践

### DO ✓

- **优先使用构造函数注入**,代码更清晰
- **面向接口编程**,便于替换实现和测试
- **保持服务无状态**,避免在 Service 中存储可变状态
- **合理使用单例**,大部分服务默认就是单例

### DON'T ✗

- **避免在构造函数中执行耗时操作**
- **不要在静态方法中使用容器**
- **避免循环依赖**,及时重构代码结构
- **不要过度注入**,单个类依赖不超过 7 个为宜

## 调试技巧

### 查看容器中已注册的服务

```php
// 在任意可注入的地方
public function debug(\Psr\Container\ContainerInterface $container)
{
    // 检查服务是否注册
    var_dump($container->has(UserService::class));

    // 获取服务定义(需要安装 hyperf/di 调试工具)
}
```

### 常见问题排查

1. **类找不到**:检查命名空间和文件路径是否匹配 PSR-4
2. **依赖无法解析**:检查构造函数参数是否有类型声明
3. **循环依赖**:使用容器延迟获取或重构代码

## 相关资源

- Hyperf 官方文档 - 依赖注入: https://hyperf.wiki/#/zh-cn/di
- PSR-11 容器规范: https://www.php-fig.org/psr/psr-11/
