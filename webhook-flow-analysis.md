# Firefly III Webhook 流程分析文档

## 目录

1. [Webhook 创建流程](#1-webhook-创建流程)
2. [交易创建后 Webhook 消息生成流程](#2-交易创建后-webhook-消息生成流程)
3. [Webhook 消息投递流程](#3-webhook-消息投递流程)
4. [配置项兼容性风险分析](#4-配置项兼容性风险分析)

---

## 1. Webhook 创建流程

### 1.1 CreateRequest 字段验证

**文件**: [CreateRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/Webhook/CreateRequest.php#L70-L91)

#### 禁止旧字段

在 `rules()` 方法中，明确禁止使用旧的单值字段，强制使用数组字段：

| 禁止字段 | 必需数组字段 | 验证规则 |
|---------|-------------|---------|
| `trigger` | `triggers` | `prohibited` → `required`, `array`, `min:1`, `max:10` |
| `response` | `responses` | `prohibited` → `required`, `array`, `min:1`, `max:1` |
| `delivery` | `deliveries` | `prohibited` → `required`, `array`, `min:1`, `max:1` |

#### 验证规则代码

```php
return [
    'trigger'      => 'prohibited',      // 禁止旧字段
    'triggers'     => ['required', 'array', 'min:1', 'max:10'],
    'triggers.*'   => sprintf('required|in:%s', $triggers),
    
    'response'     => 'prohibited',      // 禁止旧字段
    'responses'    => ['required', 'array', 'min:1', 'max:1'],
    'responses.*'  => sprintf('required|in:%s', $responses),
    
    'delivery'     => 'prohibited',      // 禁止旧字段
    'deliveries'   => ['required', 'array', 'min:1', 'max:1'],
    'deliveries.*' => sprintf('required|in:%s', $deliveries),
];
```

#### 附加验证（ValidatesWebhooks trait）

**文件**: [ValidatesWebhooks.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Request/ValidatesWebhooks.php#L34-L94)

- 如果 `triggers` 包含 `ANY`，则不能包含其他触发器
- 验证触发器和响应的组合是否被 `forbidden_responses` 配置禁止

### 1.2 StoreController 配置控制

**文件**: [StoreController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Webhook/StoreController.php#L63-L92)

#### allow_webhooks 配置检查

在 `store()` 方法中，首先检查 webhook 功能是否启用：

```php
public function store(CreateRequest $request): JsonResponse
{
    $data = $request->getData();
    
    // 检查 allow_webhooks 配置
    if (false === FireflyConfig::get('allow_webhooks', config('firefly.allow_webhooks'))->data) {
        Log::channel('audit')->info('User tries to store new webhook, but webhooks are DISABLED.', $data);
        throw new NotFoundHttpException('Webhooks are not enabled.');
    }
    
    $webhook = $this->repository->store($data);
    // ...
}
```

### 1.3 WebhookRepository::store 存储逻辑

**文件**: [WebhookRepository.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Webhook/WebhookRepository.php#L98-L153)

#### Secret 生成

使用 `Str::random(24)` 生成 24 位随机字符串作为 webhook secret：

```php
$secret = Str::random(24);
```

#### Webhook 主记录创建

```php
$fullData = [
    'user_id'       => $this->user->id,
    'user_group_id' => $this->user->user_group_id,
    'active'        => $data['active'] ?? false,
    'title'         => $data['title'] ?? null,
    'trigger'       => 1,        // 标记为已升级格式
    'response'      => 1,        // 标记为已升级格式
    'delivery'      => 1,        // 标记为已升级格式
    'secret'        => $secret,
    'url'           => $data['url'],
];

$webhook = Webhook::create($fullData);
```

#### 关系表写入

写入三张关系表：

1. **`webhook_webhook_trigger`** - webhook 与触发器的多对多关系
2. **`webhook_webhook_response`** - webhook 与响应的多对多关系
3. **`webhook_webhook_delivery`** - webhook 与投递方式的多对多关系

```php
// 写入触发器关系
foreach ($data['triggers'] as $trigger) {
    $object = WebhookTrigger::query()->where('title', $trigger)->first();
    $triggers->push($object);
}
$webhook->webhookTriggers()->saveMany($triggers);

// 写入响应关系
foreach ($data['responses'] as $response) {
    $object = WebhookResponse::query()->where('title', $response)->first();
    $responses->push($object);
}
$webhook->webhookResponses()->saveMany($responses);

// 写入投递关系
foreach ($data['deliveries'] as $delivery) {
    $object = WebhookDelivery::query()->where('title', $delivery)->first();
    $deliveries->push($object);
}
$webhook->webhookDeliveries()->saveMany($deliveries);
```

---

## 2. 交易创建后 Webhook 消息生成流程

### 2.1 ProcessesNewTransactionGroup::handle

**文件**: [ProcessesNewTransactionGroup.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesNewTransactionGroup.php#L39-L75)

当交易创建事件触发时，`handle()` 方法被调用：

```php
public function handle(CreatedSingleTransactionGroup|UserRequestedBatchProcessing $event): void
{
    // ... 其他处理
    
    if ($event->flags->fireWebhooks) {
        $this->createWebhookMessages($event->objects->transactionGroups, WebhookTrigger::STORE_TRANSACTION);
    }
    
    // ...
}
```

### 2.2 SupportsGroupProcessingTrait::createWebhookMessages

**文件**: [SupportsGroupProcessingTrait.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/TransactionGroup/SupportsGroupProcessingTrait.php#L131-L155)

```php
private function createWebhookMessages(Collection $groups, WebhookTrigger $trigger): void
{
    if (0 === $groups->count()) {
        return;
    }

    $first  = $groups->first();
    $user   = $first->user;

    /** @var MessageGeneratorInterface $engine */
    $engine = app(MessageGeneratorInterface::class);
    $engine->setUser($user);
    $engine->setTrigger($trigger);       // STORE_TRANSACTION
    $engine->setObjects($groups);        // 交易组集合
    $engine->generateMessages();         // 生成消息
}
```

### 2.3 StandardMessageGenerator::getWebhooks

**文件**: [StandardMessageGenerator.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Generator/Webhook/StandardMessageGenerator.php#L314-L324)

获取匹配的 webhook：

```php
private function getWebhooks(): Collection
{
    return $this->user
        ->webhooks()
        ->leftJoin('webhook_webhook_trigger', 'webhook_webhook_trigger.webhook_id', 'webhooks.id')
        ->leftJoin('webhook_triggers', 'webhook_webhook_trigger.webhook_trigger_id', 'webhook_triggers.id')
        ->where('active', true)
        ->whereIn('webhook_triggers.title', [$this->trigger->name, WebhookTrigger::ANY->name])
        ->get(['webhooks.*']);
}
```

**匹配逻辑**：
- Webhook 必须是 `active = true`
- 触发器必须匹配当前触发事件（如 `STORE_TRANSACTION`）或设置为 `ANY`

### 2.4 StandardMessageGenerator::generateMessage

**文件**: [StandardMessageGenerator.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Generator/Webhook/StandardMessageGenerator.php#L130-L266)

#### 消息基本结构

```php
$basicMessage = [
    'uuid'          => $uuid->toString(),      // UUID v4
    'user_id'       => 0,                      // 后续填充
    'user_group_id' => 0,                      // 后续填充
    'trigger'       => $this->trigger->name,   // STORE_TRANSACTION
    'response'      => $response->title,       // 响应类型
    'url'           => $webhook->url,          // 目标 URL
    'version'       => sprintf('v%d', $this->getVersion()),
    'content'       => [],                     // 实际内容
];
```

#### RELEVANT 响应选择

**文件**: [StandardMessageGenerator.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Generator/Webhook/StandardMessageGenerator.php#L268-L300)

当 webhook 响应设置为 `RELEVANT` 时，根据对象类型动态选择实际响应：

```php
private function getRelevantResponse(WebhookResponseModel $response, string $class): string
{
    if (WebhookResponse::NONE->name === $response->title) {
        return WebhookResponse::NONE->name;
    }

    if (WebhookResponse::RELEVANT->name === $response->title) {
        switch ($class) {
            case TransactionGroup::class:
                return WebhookResponse::TRANSACTIONS->name;

            case Budget::class:
            case BudgetLimit::class:
                return WebhookResponse::BUDGET->name;

            default:
                throw new FireflyException(...);
        }
    }
    return $response->title;
}
```

| 对象类型 | RELEVANT 映射到 |
|---------|----------------|
| TransactionGroup | TRANSACTIONS |
| Budget / BudgetLimit | BUDGET |

#### 内容生成

根据响应类型生成不同内容：

- **TRANSACTIONS**: 使用 `TransactionGroupTransformer` 转换交易组
- **ACCOUNTS**: 收集交易涉及的所有账户并转换
- **BUDGET**: 使用 `BudgetTransformer` 或 `BudgetLimitTransformer`
- **NONE**: 空数组

### 2.5 WebhookMessageFactory::create

**文件**: [WebhookMessageFactory.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/WebhookMessageFactory.php#L33-L45)

创建 WebhookMessage 记录：

```php
public function create(Webhook $webhook, array $data): WebhookMessage
{
    $webhookMessage          = new WebhookMessage();
    $webhookMessage->webhook()->associate($webhook);
    $webhookMessage->sent    = false;    // 初始状态：未发送
    $webhookMessage->errored = false;    // 初始状态：无错误
    $webhookMessage->uuid    = $data['uuid'];
    $webhookMessage->message = $data;    // 存储完整消息
    $webhookMessage->save();

    return $webhookMessage;
}
```

---

## 3. Webhook 消息投递流程

### 3.1 WebhookMessagesRequestSending 事件

**文件**: [WebhookMessagesRequestSending.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Events/Model/Webhook/WebhookMessagesRequestSending.php#L30-L32)

触发投递请求的事件类，仅作为事件标记。

### 3.2 SendsWebhookMessages 监听器

**文件**: [SendsWebhookMessages.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/Webhook/SendsWebhookMessages.php#L36-L70)

#### 配置检查

```php
if (false === config('firefly.feature_flags.webhooks') 
    || false === FireflyConfig::get('allow_webhooks', config('firefly.allow_webhooks'))->data) {
    Log::debug('Webhook event handler is disabled.');
    return;
}
```

#### 消息筛选

```php
$messages = WebhookMessage::query()
    ->where('webhook_messages.sent', false)
    ->get(['webhook_messages.*'])
    ->filter(static fn (WebhookMessage $message): bool => $message->webhookAttempts()->count() <= 2)
    ->splice(0, 5);  // 每批最多 5 条
```

**筛选规则**：
- `sent = false` - 未发送
- 尝试次数 `<= 2` - 最多重试 2 次（共 3 次尝试）
- 每批最多 **5 条** 消息

#### 消息分发

```php
foreach ($messages as $message) {
    if (false === $message->sent) {
        $message->sent = true;
        $message->save();
        SendWebhookMessage::dispatch($message)->afterResponse();
    }
}
```

### 3.3 SendWebhookMessage Job

**文件**: [SendWebhookMessage.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/SendWebhookMessage.php#L56-L63)

```php
public function handle(): void
{
    $sender = app(WebhookSenderInterface::class);
    $sender->setMessage($this->message);
    $sender->send();
}
```

### 3.4 StandardWebhookSender::send

**文件**: [StandardWebhookSender.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Webhook/StandardWebhookSender.php#L60-L162)

#### 签名生成

```php
$signatureGenerator = app(SignatureGeneratorInterface::class);
$signature = $signatureGenerator->generate($this->message);
```

#### 请求选项

```php
$options = [
    'body'            => $json,
    'allow_redirects' => false,
    'headers'         => [
        'Content-Type'    => 'application/json',
        'Accept'          => 'application/json',
        'Signature'       => $signature,    // 签名头
        'connect_timeout' => 3.14,
        'User-Agent'      => sprintf('FireflyIII/%s', config('firefly.version')),
        'timeout'         => 10,
    ],
];
```

#### 状态管理

| 场景 | sent | errored |
|------|------|---------|
| 发送成功 | `true` | `false` |
| JSON 编码失败 | `false` | `true` |
| 签名生成失败 | `false` | `true` |
| HTTP 请求失败 | `false` | `true` |

#### 异常记录

所有异常都会记录到 `webhook_attempts` 表：

```php
$attempt = new WebhookAttempt();
$attempt->webhookMessage()->associate($this->message);
$attempt->status_code   = $statusCode;  // 0 表示非 HTTP 错误
$attempt->logs          = $errorMessage . "\n" . $stackTrace;
$attempt->save();
```

### 3.5 Sha3SignatureGenerator::generate

**文件**: [Sha3SignatureGenerator.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Helpers/Webhook/Sha3SignatureGenerator.php#L45-L79)

#### 签名算法

```php
// 1. 获取时间戳
$timestamp = Carbon::now()->getTimestamp();

// 2. 构造签名载荷：timestamp.JSON
$payload = sprintf('%s.%s', $timestamp, $json);

// 3. 使用 HMAC-SHA3-256 签名
$signature = hash_hmac('sha3-256', $payload, (string) $message->webhook->secret);

// 4. 格式化签名头
return sprintf('t=%s,v%d=%s', $timestamp, $this->getVersion(), $signature);
```

#### 签名头格式

```
t=1620000000,v1=abc123def456...
```

---

## 4. 配置项兼容性风险分析

**文件**: [webhooks.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/webhooks.php#L31-L106)

### 4.1 forbidden_responses 配置

#### 配置内容

| Trigger | Forbidden Responses |
|---------|-------------------|
| `ANY` | `BUDGET`, `TRANSACTIONS`, `ACCOUNTS` |
| `STORE_TRANSACTION` | `BUDGET` |
| `UPDATE_TRANSACTION` | `BUDGET` |
| `DESTROY_TRANSACTION` | `BUDGET` |
| `STORE_BUDGET` | `TRANSACTIONS`, `ACCOUNTS` |
| `UPDATE_BUDGET` | `TRANSACTIONS`, `ACCOUNTS` |
| `DESTROY_BUDGET` | `TRANSACTIONS`, `ACCOUNTS` |
| `STORE_UPDATE_BUDGET_LIMIT` | `TRANSACTIONS`, `ACCOUNTS` |

#### 兼容性风险与测试覆盖

| 风险场景 | 测试点 |
|---------|-------|
| **旧版本 webhook 迁移** | 验证升级脚本是否正确处理旧的单字段 trigger/response 组合，避免出现被禁止的组合 |
| **API 向后兼容** | 测试使用旧字段（trigger/response/delivery）是否正确返回 prohibited 错误 |
| **ANY 触发器组合** | 测试 ANY 触发器与任何响应类型的组合验证 |
| **新触发器/响应添加** | 添加新的 WebhookTrigger 或 WebhookResponse 枚举值时，必须同步更新此配置 |
| **跨类型触发** | 测试 STORE_TRANSACTION 触发时不会发送 BUDGET 响应内容 |

### 4.2 force_relevant_response 配置

#### 配置内容

| 触发事件 | 关联触发事件 |
|---------|------------|
| `STORE_TRANSACTION` | `STORE_BUDGET`, `UPDATE_BUDGET`, `DESTROY_BUDGET`, `STORE_UPDATE_BUDGET_LIMIT` |
| `UPDATE_TRANSACTION` | `STORE_BUDGET`, `UPDATE_BUDGET`, `DESTROY_BUDGET`, `STORE_UPDATE_BUDGET_LIMIT` |
| `DESTROY_TRANSACTION` | `STORE_BUDGET`, `UPDATE_BUDGET`, `DESTROY_BUDGET`, `STORE_UPDATE_BUDGET_LIMIT` |
| `STORE_BUDGET` | `STORE_TRANSACTION`, `UPDATE_TRANSACTION`, `DESTROY_TRANSACTION` |
| `UPDATE_BUDGET` | `STORE_TRANSACTION`, `UPDATE_TRANSACTION`, `DESTROY_TRANSACTION` |
| `DESTROY_BUDGET` | `STORE_TRANSACTION`, `UPDATE_TRANSACTION`, `DESTROY_TRANSACTION` |
| `STORE_UPDATE_BUDGET_LIMIT` | `STORE_TRANSACTION`, `UPDATE_TRANSACTION`, `DESTROY_TRANSACTION` |

#### 兼容性风险与测试覆盖

| 风险场景 | 测试点 |
|---------|-------|
| **RELEVANT 响应一致性** | 当配置为 RELEVANT 时，不同触发事件应返回正确的内容类型 |
| **级联触发场景** | 创建交易时同时修改预算，验证 webhook 消息是否正确关联 |
| **新事件类型添加** | 添加新触发事件时必须考虑是否需要添加关联映射 |
| **消息去重** | 同一操作可能触发多个关联事件，需验证消息不会重复发送 |
| **数据一致性** | 确保关联触发的消息内容与主触发内容保持一致 |

### 4.3 综合测试建议

1. **升级测试**：从旧版本（使用单值 trigger/response 字段）升级到新版本（数组字段）
2. **边界测试**：
   - 每批正好 5 条消息
   - 尝试次数正好 2 次
   - webhook 启用/禁用切换
3. **错误场景测试**：
   - URL 无效
   - 签名生成失败
   - 目标服务器超时
   - 目标服务器返回错误状态码
4. **并发测试**：多个交易同时创建时的消息生成顺序

---

## 附录：关键枚举定义

### WebhookTrigger 枚举

**文件**: [WebhookTrigger.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Enums/WebhookTrigger.php#L30-L40)

```php
enum WebhookTrigger: int
{
    case ANY                       = 50;
    case STORE_TRANSACTION         = 100;
    case UPDATE_TRANSACTION        = 110;
    case DESTROY_TRANSACTION       = 120;
    case STORE_BUDGET              = 200;
    case UPDATE_BUDGET             = 210;
    case DESTROY_BUDGET            = 220;
    case STORE_UPDATE_BUDGET_LIMIT = 230;
}
```

### WebhookResponse 枚举

**文件**: [WebhookResponse.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Enums/WebhookResponse.php#L30-L37)

```php
enum WebhookResponse: int
{
    case TRANSACTIONS = 200;
    case ACCOUNTS     = 210;
    case BUDGET       = 230;
    case RELEVANT     = 240;
    case NONE         = 220;
}
```
