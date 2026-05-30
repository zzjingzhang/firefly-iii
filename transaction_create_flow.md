# Firefly III 交易创建流程追踪文档

本文档详细追踪一次包含两个 split 的交易创建请求的完整流程，从 `POST v1/transactions` 开始，深入分析各层代码的处理逻辑。

---

## 1. 路由入口

**路由定义**：[api.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/routes/api.php#L605)
```php
Route::post('', ['uses' => 'StoreController@store', 'as' => 'store']);
```
命名空间：`FireflyIII\Api\V1\Controllers\Models\Transaction`

---

## 2. StoreRequest 数据验证与清洗

### 2.1 rules() 方法 - 验证约束

**文件**：[StoreRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/Transaction/StoreRequest.php#L81-L172)

`rules()` 方法定义了完整的验证规则，对 `transactions` 字段的约束包括：

#### 组级别约束
| 字段 | 规则 |
|------|------|
| `group_title` | `min:1`, `max:1000`, `nullable` |
| `error_if_duplicate_hash` | `IsBoolean` |
| `fire_webhooks` | `IsBoolean` |
| `apply_rules` | `IsBoolean` |

#### transactions.* 数组约束（每个 split）
| 字段组 | 约束规则 |
|--------|----------|
| **基本信息** | `type` (required, in:withdrawal/deposit/transfer/opening-balance/reconciliation), `date` (required, IsDateOrTime), `order` (numeric, min:0) |
| **货币信息** | `currency_id` (exists), `currency_code` (exists), `foreign_currency_id`, `foreign_currency_code` |
| **金额** | `amount` (required, IsValidPositiveAmount), `foreign_amount` (IsValidZeroOrMoreAmount) |
| **描述** | `description` (min:1, max:1000, nullable) |
| **源账户** | `source_id`, `source_name`, `source_iban` (iban), `source_number`, `source_bic` (bic) |
| **目标账户** | `destination_id`, `destination_name`, `destination_iban`, `destination_number`, `destination_bic` |
| **关联对象** | `budget_id`, `budget_name`, `category_id`, `category_name`, `bill_id`, `piggy_bank_id` |
| **其他** | `reconciled` (IsBoolean), `notes` (max:32768), `tags`, `tags.*` (max:255) |
| **元信息** | `internal_reference`, `external_id`, `recurrence_id`, `bunq_payment_id`, `external_url` |
| **SEPA字段** | `sepa_cc`, `sepa_ct_op`, `sepa_ct_id`, `sepa_db`, `sepa_country`, `sepa_ep`, `sepa_ci`, `sepa_batch_id` |
| **日期字段** | `interest_date`, `book_date`, `process_date`, `due_date`, `payment_date`, `invoice_date` |
| **位置** | `latitude`, `longitude`, `zoom_level` |

#### withValidator 额外验证
在 [withValidator](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/Transaction/StoreRequest.php#L177-L209) 方法中，还有额外的后置验证：
- `validateTransactionArray()` - 验证数组结构有效性
- `validateOneTransaction()` - 至少提交一个交易
- `validateDescriptions()` - 所有交易必须有描述
- `validateTransactionTypes()` - 所有交易类型必须一致
- `validateForeignCurrencyInformation()` - 验证外币信息
- `validateAccountInformation()` - 验证账户信息
- `validateEqualAccounts()` - 根据交易类型验证源/目标账户是否相等
- `validateGroupDescription()` - 如果 >1 个交易，组必须有描述

### 2.2 getAll() 方法 - 数据清洗

**文件**：[StoreRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/Transaction/StoreRequest.php#L62-L76)

```php
public function getAll(): array
{
    return [
        'group_title'             => $this->convertString('group_title'),
        'error_if_duplicate_hash' => $this->boolean('error_if_duplicate_hash'),
        'batch_submission'        => $this->boolean('batch_submission'),
        'apply_rules'             => $this->boolean('apply_rules', true),
        'fire_webhooks'           => $this->boolean('fire_webhooks', true),
        'transactions'            => $this->getTransactionData(),
    ];
}
```

#### getTransactionData() 深层清洗

**文件**：[StoreRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/Transaction/StoreRequest.php#L214-L309)

对每个交易 split 进行的清洗操作：

| 处理方法 | 作用 | 字段示例 |
|----------|------|----------|
| `clearString()` | 清理字符串，去除空白 | `type`, `description`, `source_name` |
| `dateFromValue()` | 转换为 Carbon 日期对象 | `date`, `interest_date`, `book_date` |
| `integerFromValue()` | 转换为整数 | `order`, `currency_id`, `source_id` |
| `floatFromValue()` | 转换为浮点数 | `latitude`, `longitude` |
| `clearIban()` | 清理 IBAN | `source_iban`, `destination_iban` |
| `convertBoolean()` | 转换为布尔值 | `reconciled` |
| `clearStringKeepNewlines()` | 清理字符串但保留换行 | `notes` |
| `arrayFromValue()` | 转换为数组 | `tags` |

额外补充字段：
- `original_source` = `sprintf('ff3-v%s', config('firefly.version'))`

---

## 3. StoreController::store 处理逻辑

**文件**：[StoreController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Transaction/StoreController.php#L85-L145)

### 3.1 补充字段

```php
$data               = $request->getAll();
$data['user']       = auth()->user();           // 补充当前用户
$data['user_group'] = $this->userGroup;         // 补充用户组
```

### 3.2 异常处理流程

```php
try {
    $transactionGroup = $this->groupRepository->store($data);
} catch (DuplicateTransactionException $e) {
    // 处理重复交易异常
    $validator = Validator::make(['transactions' => [['description' => $e->getMessage()]]], [
        'transactions.0.description' => new IsDuplicateTransaction(),
    ]);
    throw new ValidationException($validator);
} catch (FireflyException $e) {
    // 处理通用异常
    $message   = sprintf('Internal exception: %s', $e->getMessage());
    $validator = Validator::make(['transactions' => [['description' => $message]]], ['transactions.0.description' => new IsDuplicateTransaction()]);
    throw new ValidationException($validator);
}
```

**关键点**：
- 捕获 `DuplicateTransactionException` 时，使用 `IsDuplicateTransaction` 规则（[IsDuplicateTransaction.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Rules/IsDuplicateTransaction.php)）将异常信息直接作为验证错误返回
- `IsDuplicateTransaction::validate()` 直接调用 `$fail($value)`，将异常消息作为验证错误

### 3.3 410 Gone 错误路径

**文件**：[StoreController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Transaction/StoreController.php#L129-L132)

```php
$selectedGroup = $collector->getGroups()->first();
if (null === $selectedGroup) {
    throw HttpException::fromStatusCode(410, '200032: Cannot find transaction. Possibly, a rule deleted this transaction after its creation.');
}
```

**为什么返回 410**：
- 交易创建成功后，触发事件异步执行规则
- 规则可能包含 `DeleteTransaction` 动作（[DeleteTransaction.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Actions/DeleteTransaction.php)）
- 如果规则删除了刚创建的交易，后续查询时 `$selectedGroup` 为 null
- 410 Gone 表示资源曾经存在但现在已被删除（永久删除）

---

## 4. TransactionGroupRepository::store

**文件**：[TransactionGroupRepository.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L349-L380)

```php
public function store(array $data): TransactionGroup
{
    $factory = app(TransactionGroupFactory::class);
    $factory->setUser($data['user']);
    $factory->setUserGroup($data['user_group']);

    try {
        $transactionGroup = $factory->create($data);
    } catch (DuplicateTransactionException $e) {
        throw new DuplicateTransactionException($e->getMessage(), 0, $e);
    } catch (FireflyException $e) {
        throw new FireflyException($e->getMessage(), 0, $e);
    }

    // 触发事件
    $flags = new TransactionGroupEventFlags();
    $flags->applyRules      = $data['apply_rules'] ?? true;
    $flags->fireWebhooks    = $data['fire_webhooks'] ?? true;
    $flags->batchSubmission = $data['batch_submission'] ?? false;
    
    event(new CreatedSingleTransactionGroup($flags, $objects));
    event(new WebhookMessagesRequestSending());

    return $transactionGroup;
}
```

**职责**：
- 委托 `TransactionGroupFactory` 创建交易组
- 异常转译（重新抛出相同异常）
- 触发 `CreatedSingleTransactionGroup` 事件，传递 `apply_rules`、`fire_webhooks`、`batch_submission` 标志

---

## 5. TransactionGroupFactory::create

**文件**：[TransactionGroupFactory.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/TransactionGroupFactory.php#L57-L90)

```php
public function create(array $data): TransactionGroup
{
    $this->journalFactory->setUser($data['user']);
    $this->journalFactory->setUserGroup($data['user_group']);
    $this->journalFactory->setErrorOnHash($data['error_if_duplicate_hash'] ?? false);

    try {
        $collection = $this->journalFactory->create($data);
    } catch (DuplicateTransactionException $e) {
        throw new DuplicateTransactionException($e->getMessage(), 0, $e);
    }

    // 处理 group_title
    $title = $data['group_title'] ?? null;
    $title = '' === $title ? null : $title;
    if (null !== $title) {
        $title = substr((string) $title, 0, 1000);
    }

    // 创建 TransactionGroup
    $group = new TransactionGroup();
    $group->user()->associate($this->user);
    $group->userGroup()->associate($this->userGroup);
    $group->title = $title;
    $group->save();

    // 关联所有 TransactionJournal
    $group->transactionJournals()->saveMany($collection);

    return $group;
}
```

**关键点**：
- `group_title` 生成：优先使用用户输入，截断到 1000 字符
- 关联 `TransactionJournal` 集合到 `TransactionGroup`
- 传递 `error_if_duplicate_hash` 标志给 `TransactionJournalFactory`

---

## 6. TransactionJournalFactory::createJournal

**文件**：[TransactionJournalFactory.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/TransactionJournalFactory.php#L230-L410)

### 6.1 import_hash_v2 生成与重复检查

```php
private function createJournal(NullArrayObject $row): ?TransactionJournal
{
    // 1. 生成 import_hash_v2
    $row['import_hash_v2'] = $this->hashArray($row);
    
    // 2. 检查重复（如果 error_if_duplicate_hash = true）
    $this->errorIfDuplicate($row['import_hash_v2']);
    
    // ... 后续创建逻辑
}
```

#### hashArray() 方法

**文件**：[TransactionJournalFactory.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/TransactionJournalFactory.php#L525-L539)

```php
private function hashArray(NullArrayObject $row): string
{
    unset($row['import_hash_v2'], $row['original_source']);
    
    try {
        $json = json_encode($row, JSON_THROW_ON_ERROR);
    } catch (JsonException $e) {
        $json = microtime();
    }
    $hash = hash('sha256', $json);
    
    return $hash;
}
```

**哈希计算内容**：
- 排除 `import_hash_v2` 和 `original_source` 字段
- 对整个 $row 数组做 JSON 编码
- 计算 SHA256 哈希值

#### errorIfDuplicate() 方法

**文件**：[TransactionJournalFactory.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/TransactionJournalFactory.php#L419-L445)

```php
private function errorIfDuplicate(string $hash): void
{
    if (false === $this->errorOnHash) {
        return;
    }

    // 查询 journal_meta 表中是否已存在该 hash
    $result = TransactionJournalMeta::query()
        ->withTrashed()
        ->leftJoin('transaction_journals', 'transaction_journals.id', '=', 'journal_meta.transaction_journal_id')
        ->whereNotNull('transaction_journals.id')
        ->where('transaction_journals.user_id', $this->user->id)
        ->where('data', json_encode($hash, JSON_THROW_ON_ERROR))
        ->first(['journal_meta.*']);
        
    if (null !== $result) {
        $journal = $result->transactionJournal()->withTrashed()->first();
        $group   = $journal?->transactionGroup()->withTrashed()->first();
        $groupId = (int) $group?->id;

        throw new DuplicateTransactionException(sprintf('Duplicate of transaction #%d.', $groupId));
    }
}
```

### 6.2 TransactionJournal 创建

```php
$journal = TransactionJournal::create([
    'user_id'                 => $this->user->id,
    'user_group_id'           => $this->userGroup->id,
    'transaction_type_id'     => $type->id,
    'bill_id'                 => $billId,
    'transaction_currency_id' => $currency->id,
    'description'             => substr($description, 0, 1000),
    'date'                    => $carbon,
    'date_tz'                 => $carbon->format('e'),
    'order'                   => $order,
    'tag_count'               => 0,
    'completed'               => is_bool($row['batch_submission']) && !$row['batch_submission'],
]);
```

### 6.3 两条 Transaction 记录创建

每个 TransactionJournal 包含两条 Transaction 记录（一正一负，复式记账）：

**第一条（负向，源账户）**：
```php
$transactionFactory->setAccount($sourceAccount);
$negative = $transactionFactory->createNegative((string) $row['amount'], (string) $row['foreign_amount']);
```

**第二条（正向，目标账户）**：
```php
$transactionFactory->setAccount($destinationAccount);
// 对于外币转账，会交换货币和金额
if (/* 外币转账条件 */) {
    $transactionFactory->setCurrency($foreignCurrency);
    $transactionFactory->setForeignCurrency($currency);
    $amount        = (string) $row['foreign_amount'];
    $foreignAmount = (string) $row['amount'];
}
$transactionFactory->createPositive($amount, $foreignAmount);
```

**TransactionFactory::create()**：[TransactionFactory.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/TransactionFactory.php#L121-L172)

```php
private function create(string $amount, ?string $foreignAmount): Transaction
{
    $data = [
        'reconciled'              => $this->reconciled,
        'account_id'              => $this->account->id,
        'transaction_journal_id'  => $this->journal->id,
        'description'             => null,
        'transaction_currency_id' => $this->currency->id,
        'amount'                  => $amount,
        'foreign_amount'          => null,
        'foreign_currency_id'     => null,
        'identifier'              => 0,
    ];
    $result = Transaction::create($data);
    
    // 设置外币信息
    if ($this->foreignCurrency instanceof TransactionCurrency && null !== $foreignAmount) {
        $result->foreign_currency_id = $this->foreignCurrency->id;
        $result->foreign_amount      = $foreignAmount;
    }
    $result->save();
    
    return $result;
}
```

### 6.4 副作用处理

创建 TransactionJournal 后，依次处理各种关联数据：

| 方法 | 功能 | 关键逻辑 |
|------|------|----------|
| `storeBudget()` | 关联预算 | 仅对 withdrawal 类型有效，通过 `budget_id` 或 `budget_name` 查找，使用 `sync()` 关联 |
| `storeCategory()` | 关联分类 | 通过 `category_id` 或 `category_name` 查找，使用 `sync()` 关联 |
| `storeNotes()` | 存储备注 | 创建或更新 `Note` 模型，空字符串则删除 |
| `storePiggyEvent()` | 关联储蓄罐 | 仅对 transfer 类型有效，创建 `PiggyBankEvent` |
| `storeTags()` | 关联标签 | 通过 `TagFactory::findOrCreate()` 查找或创建，使用 `sync()` 关联 |
| `storeMetaFields()` | 存储元字段 | 存储 `sepa_*`、`external_id` 等字段到 `journal_meta` 表 |
| `storeLocation()` | 存储位置 | 如果经纬度和缩放级别都不为 null，创建 `Location` 模型 |

**各方法实现位置**：[JournalServiceTrait.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Internal/Support/JournalServiceTrait.php)

---

## 7. 事件处理：规则与 Webhook

### 7.1 CreatedSingleTransactionGroup 事件

**事件类**：[CreatedSingleTransactionGroup.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Events/Model/TransactionGroup/CreatedSingleTransactionGroup.php)

**监听器**：[ProcessesNewTransactionGroup.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesNewTransactionGroup.php)

```php
public function handle(CreatedSingleTransactionGroup $event): void
{
    if ($event->flags->applyRules) {
        $this->processRules($journals, 'store-journal');
    }
    if ($event->flags->fireWebhooks) {
        $this->createWebhookMessages($event->objects->transactionGroups, WebhookTrigger::STORE_TRANSACTION);
    }
    // ... 其他处理
}
```

### 7.2 规则引擎执行

**SupportsGroupProcessingTrait::processRules()**：[SupportsGroupProcessingTrait.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/TransactionGroup/SupportsGroupProcessingTrait.php#L29-L66)

```php
protected function processRules(Collection $set, string $type): void
{
    $ruleGroupRepository = app(RuleGroupRepositoryInterface::class);
    $groups = $ruleGroupRepository->getRuleGroupsWithRules($type);
    
    $newRuleEngine = app(RuleEngineInterface::class);
    $newRuleEngine->setUser($user);
    $newRuleEngine->setRuleGroups($groups);
    
    foreach ($array as $journalId) {
        $newRuleEngine->removeOperator('journal_id');
        $newRuleEngine->addOperator(['type' => 'journal_id', 'value' => $journalId]);
        $newRuleEngine->fire();
    }
}
```

### 7.3 DeleteTransaction 规则动作

**文件**：[DeleteTransaction.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Actions/DeleteTransaction.php)

```php
public function actOnArray(array $journal): bool
{
    $count = TransactionJournal::query()->where('transaction_group_id', $journal['transaction_group_id'])->count();

    // 如果组内只有一个 journal，删除整个组
    if (1 === $count) {
        $group = TransactionGroup::find($journal['transaction_group_id']);
        $service = app(TransactionGroupDestroyService::class);
        $service->destroy($group);
        return true;
    }
    
    // 否则只删除单个 journal
    $object = TransactionJournal::find($journal['transaction_journal_id']);
    $service = app(JournalDestroyService::class);
    $service->destroy($object);
    
    return true;
}
```

**这就是为什么可能返回 410 的原因**：
- 交易创建后，事件队列处理规则
- 规则触发 `DeleteTransaction` 动作，删除整个交易组
- StoreController 继续执行，尝试查询刚创建的交易组
- 查询结果为 null，抛出 410 Gone 异常

---

## 8. error_if_duplicate_hash = true 时的错误路径

### 完整调用链

```
POST v1/transactions
    ↓
StoreRequest::rules() + withValidator() 验证
    ↓
StoreRequest::getAll() 清洗数据
    ↓
StoreController::store()
    ├─ 补充 user, user_group
    └─ groupRepository->store($data)
        └─ TransactionGroupFactory::create()
            ├─ setErrorOnHash(true)
            └─ TransactionJournalFactory::create()
                └─ 遍历每个 split:
                    └─ createJournal()
                        ├─ hashArray() 生成 import_hash_v2
                        ├─ errorIfDuplicate() 检查
                        │   └─ 发现重复 → throw DuplicateTransactionException
                        └─ ... (创建逻辑)
```

### 异常冒泡路径

```
TransactionJournalFactory::createJournal()
    ↓ throw DuplicateTransactionException
TransactionJournalFactory::create()
    ↓ catch + forceDeleteOnError() + rethrow
TransactionGroupFactory::create()
    ↓ catch + rethrow
TransactionGroupRepository::store()
    ↓ catch + rethrow
StoreController::store()
    ↓ catch
    ├─ 创建 Validator，使用 IsDuplicateTransaction 规则
    └─ throw ValidationException
```

### 错误回滚机制

在 `TransactionJournalFactory::create()` 中，发生异常时会调用 `forceDeleteOnError()`：

```php
catch (DuplicateTransactionException $e) {
    $this->forceDeleteOnError($collection);  // 删除已创建的 journals
    throw new DuplicateTransactionException($e->getMessage(), 0, $e);
}
```

`forceDeleteOnError()` 使用 `JournalDestroyService` 删除所有已创建的 TransactionJournal，确保数据一致性。

---

## 9. 两个 split 的交易创建流程总结

对于包含两个 split 的交易：

1. **请求数据结构**：
   ```json
   {
     "group_title": "超市购物",
     "error_if_duplicate_hash": true,
     "apply_rules": true,
     "fire_webhooks": true,
     "transactions": [
       { "type": "withdrawal", "description": " groceries", "amount": 50, ... },
       { "type": "withdrawal", "description": "日用品", "amount": 30, ... }
     ]
   }
   ```

2. **生成的数据库记录**：
   - 1 条 `TransactionGroup` 记录（group_title = "超市购物"）
   - 2 条 `TransactionJournal` 记录（每个 split 一条）
   - 4 条 `Transaction` 记录（每个 journal 2 条，一正一负）
   - N 条关联记录（budget, category, tags, meta, location, piggy 等）

3. **每个 split 的 import_hash_v2**：
   - 各自独立计算，基于各自的交易数据
   - 即使同一请求中的两个 split 内容完全相同，也会各自检查重复

4. **错误场景**：
   - 如果第二个 split 的 hash 重复，已创建的第一个 split 会被回滚删除
   - 整个请求失败，返回 ValidationException

---

## 10. 关键文件索引

| 组件 | 文件路径 |
|------|----------|
| 路由 | [api.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/routes/api.php#L597-L613) |
| StoreRequest | [StoreRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/Transaction/StoreRequest.php) |
| StoreController | [StoreController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Transaction/StoreController.php) |
| TransactionGroupRepository | [TransactionGroupRepository.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php) |
| TransactionGroupFactory | [TransactionGroupFactory.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/TransactionGroupFactory.php) |
| TransactionJournalFactory | [TransactionJournalFactory.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/TransactionJournalFactory.php) |
| TransactionFactory | [TransactionFactory.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/TransactionFactory.php) |
| JournalServiceTrait | [JournalServiceTrait.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Internal/Support/JournalServiceTrait.php) |
| 事件监听器 | [ProcessesNewTransactionGroup.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesNewTransactionGroup.php) |
| 规则引擎 | [SearchRuleEngine.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php) |
| 删除交易动作 | [DeleteTransaction.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Actions/DeleteTransaction.php) |
| 重复交易异常 | [DuplicateTransactionException.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Exceptions/DuplicateTransactionException.php) |
| 重复验证规则 | [IsDuplicateTransaction.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Rules/IsDuplicateTransaction.php) |

---

## 11. 关键流程图

```
POST v1/transactions
    │
    ▼
StoreRequest::rules() ── 验证失败 → ValidationException
    │
    ▼
StoreRequest::getAll() ── 清洗数据，补充 original_source
    │
    ▼
StoreController::store()
    ├─ 补充 user, user_group
    ├─ apply_rules = true (默认)
    ├─ fire_webhooks = true (默认)
    │
    ▼
TransactionGroupRepository::store()
    ├─ 委托 TransactionGroupFactory
    │
    ▼
TransactionGroupFactory::create()
    ├─ setErrorOnHash($data['error_if_duplicate_hash'])
    │
    ▼
TransactionJournalFactory::create()
    │
    ├─ 遍历每个 transaction split
    │   │
    │   ├─ createJournal(split1)
    │   │   ├─ hashArray() → import_hash_v2
    │   │   ├─ errorIfDuplicate() → 重复则 throw
    │   │   ├─ 创建 TransactionJournal
    │   │   ├─ 创建 2 条 Transaction (一正一负)
    │   │   ├─ storeBudget / storeCategory
    │   │   ├─ storeTags / storeNotes
    │   │   ├─ storeMetaFields
    │   │   └─ storeLocation / storePiggyEvent
    │   │
    │   └─ createJournal(split2)
    │       └─ (同上流程)
    │
    ▼
创建 TransactionGroup，关联 journals
    │
    ▼
触发 CreatedSingleTransactionGroup 事件
    │
    ├─ apply_rules=true → 规则引擎执行
    │   └─ 可能触发 DeleteTransaction 动作 → 删除交易组
    │
    └─ fire_webhooks=true → 创建 webhook 消息
    │
    ▼
StoreController 查询 TransactionGroup
    │
    ├─ 存在 → 返回 200 + 交易数据
    └─ 不存在（被规则删除）→ 410 Gone
```
