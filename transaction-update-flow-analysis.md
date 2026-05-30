# Firefly III PUT v1/transactions/{transactionGroup} 更新流程分析

## 概述

本文档详细分析了 Firefly III 中交易组（Transaction Group）更新的完整流程，针对复杂场景：请求 payload 包含一个已有 transaction_journal_id、一个没有 transaction_journal_id 的新 split、并遗漏一个原有 split，同时修改 date 和 amount。

---

## 1. UpdateRequest 数据验证与提取

### 1.1 UpdateRequest::getAll() 方法

**文件路径**: [UpdateRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/Transaction/UpdateRequest.php#L68-L136)

该方法负责从请求中提取并规范化所有交易数据：

#### 字段分类处理
```php
// 整数字段 - 包含 transaction_journal_id
$this->integerFields = [
    'order', 'currency_id', 'foreign_currency_id',
    'transaction_journal_id',  // 关键：用于识别已有journal
    'source_id', 'destination_id', 'budget_id', 'category_id',
    'bill_id', 'recurrence_id',
];

// 日期字段
$this->dateFields = ['date', 'interest_date', 'book_date', ...];

// 金额字段
$this->floatFields = ['amount', 'foreign_amount'];

// 字符串字段 - 包含 type
$this->stringFields = ['type', 'currency_code', 'description', ...];
```

#### 交易数据提取流程
1. 遍历请求中的 `transactions` 数组
2. 对每个 transaction，通过 `getTransactionData()` 进行字段转换
3. 对于每个字段类型（integer、string、date、boolean、array、float）进行规范化处理
4. 返回完整的数据数组

**关键点**:
- 如果请求中的 split 包含 `transaction_journal_id`，会被保留用于后续更新识别
- 如果 split 不包含 `transaction_journal_id`，该字段不会出现在数据中（后续会被当作新 split）
- `date` 和 `amount` 字段会被正确转换和格式化

### 1.2 UpdateRequest::withValidator() 方法

**文件路径**: [UpdateRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/Transaction/UpdateRequest.php#L224-L258)

该方法在验证器上添加后置验证规则：

```php
$validator->after(function (Validator $validator) use ($transactionGroup): void {
    $this->validateJournalIds($validator, $transactionGroup);
    $this->validateGroupDescription($validator);
    $this->validateTransactionTypesForUpdate($validator);
    $this->preventUpdateReconciled($validator, $transactionGroup);
    $this->validateEqualAccountsForUpdate($validator, $transactionGroup);
    $this->validateAccountInformationUpdate($validator, $transactionGroup);
});
```

**关键验证**:
- `validateJournalIds`: 验证提交的 journal_id 是否属于该 transaction group
- `validateTransactionTypesForUpdate`: 确保所有交易类型一致
- `validateEqualAccountsForUpdate`: 确保源/目标账户与交易类型匹配

---

## 2. UpdateController::update 方法

**文件路径**: [UpdateController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Transaction/UpdateController.php#L76-L130)

### 核心执行流程

```php
public function update(UpdateRequest $request, TransactionGroup $transactionGroup): JsonResponse
{
    // 1. 获取所有请求数据
    $data = $request->getAll();
    
    // 2. 计算更新前的 hash
    $oldHash = $this->groupRepository->getCompareHash($transactionGroup);
    
    // 3. 收集更新前的对象（用于后续事件处理）
    $objects = TransactionGroupEventObjects::collectFromTransactionGroup($transactionGroup);
    
    // 4. 执行实际更新
    $transactionGroup = $this->groupRepository->update($transactionGroup, $data);
    
    // 5. 追加更新后的对象
    $objects->appendFromTransactionGroup($transactionGroup);
    
    // 6. 计算更新后的 hash
    $newHash = $this->groupRepository->getCompareHash($transactionGroup);
    
    // 7. 判断是否需要重新计算
    $applyRules = $data['apply_rules'] ?? true;
    $fireWebhooks = $data['fire_webhooks'] ?? true;
    $runRecalculations = $oldHash !== $newHash;  // 关键判断
    
    // 8. 设置事件标志
    $flags = new TransactionGroupEventFlags();
    $flags->applyRules = $applyRules;
    $flags->fireWebhooks = $fireWebhooks;
    $flags->recalculateCredit = $runRecalculations;  // hash 变化时才为 true
    $flags->batchSubmission = $data['batch_submission'] ?? false;
    
    // 9. 触发更新事件
    event(new UpdatedSingleTransactionGroup($flags, $objects));
    event(new WebhookMessagesRequestSending());
    
    // 10. 返回更新后的数据
    // ...
}
```

### recalculateCredit 何时为 false

**核心逻辑**（第 90 行）:
```php
$runRecalculations = $oldHash !== $newHash;
$flags->recalculateCredit = $runRecalculations;
```

**recalculateCredit = false 的情况**:
当 `oldHash === newHash` 时，即交易组的关键数据没有发生实质性变化。

**getCompareHash 的计算方式**（[TransactionGroupRepository.php#L139-L158](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L139-L158)）:

```php
public function getCompareHash(TransactionGroup $group): string
{
    $sum = '0';
    $names = '';
    
    foreach ($group->transactionJournals as $journal) {
        // 1. 包含 journal 的日期
        $names = sprintf('%s%s', $names, $journal->date->format('Y-m-d-H:i:s'));
        
        foreach ($journal->transactions as $transaction) {
            // 2. 包含所有正数金额（支出/转账的目标方）
            if (-1 === bccomp('0', (string) $transaction->amount)) {
                $sum = bcadd($sum, (string) $transaction->amount);
                // 3. 包含账户名称
                $names = sprintf('%s%s', $names, $transaction->account->name);
            }
        }
    }
    
    return hash('sha256', sprintf('%s-%s', $names, $sum));
}
```

**Hash 计算包含的要素**:
1. **日期**: 每个 journal 的 `date` 字段
2. **金额总和**: 所有正数交易的金额总和
3. **账户名称**: 所有正数交易对应的账户名称

**因此，recalculateCredit = false 的场景**:
- 仅修改了描述、标签、分类等不影响金额/日期/账户的字段
- 仅修改了 notes、meta 字段等
- 仅修改了 group_title

---

## 3. TransactionGroupRepository::update 方法

**文件路径**: [TransactionGroupRepository.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php#L386-L392)

这是一个简单的委托方法：

```php
public function update(TransactionGroup $transactionGroup, array $data): TransactionGroup
{
    /** @var GroupUpdateService $service */
    $service = app(GroupUpdateService::class);
    
    return $service->update($transactionGroup, $data);
}
```

---

## 4. GroupUpdateService 更新逻辑

**文件路径**: [GroupUpdateService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Internal/Update/GroupUpdateService.php)

### 4.1 GroupUpdateService::update 主方法

```php
public function update(TransactionGroup $transactionGroup, array $data): TransactionGroup
{
    // 1. 更新 group_title（如果提供）
    if (array_key_exists('group_title', $data)) {
        $transactionGroup->title = $data['group_title'];
        $transactionGroup->save();
    }
    
    $transactions = $data['transactions'] ?? [];
    if (0 === count($transactions)) {
        return $transactionGroup;
    }
    
    // 特殊情况：单 journal 更新
    if (1 === count($transactions) && 1 === $transactionGroup->transactionJournals()->count()) {
        $first = $transactionGroup->transactionJournals()->first();
        $this->updateTransactionJournal($transactionGroup, $first, reset($transactions));
        $transactionGroup->touch();
        $transactionGroup->refresh();
        return $transactionGroup;
    }
    
    // 2. 多 split 场景：获取所有现有 journal ID
    $existing = $transactionGroup->transactionJournals->pluck('id')->toArray();
    
    // 3. 执行更新/创建（返回更新的 journal ID 列表）
    $updated = $this->updateTransactions($transactionGroup, $transactions);
    
    // 4. 计算差异 - 找出需要删除的 journal
    $result = array_diff($existing, $updated);  // 关键：找出遗漏的 split
    
    // 5. 删除遗漏的 journal
    foreach ($result as $deletedId) {
        $journal = $transactionGroup->transactionJournals()->find((int) $deletedId);
        /** @var JournalDestroyService $service */
        $service = app(JournalDestroyService::class);
        $service->destroy($journal);  // 调用删除服务
    }
    
    $transactionGroup->touch();
    $transactionGroup->refresh();
    
    return $transactionGroup;
}
```

**遗漏 split 的删除逻辑**:
- `$existing` 数组包含数据库中该 group 的所有 journal ID
- `$updated` 数组包含请求中提到的所有 journal（包括已有和新建的）
- `array_diff($existing, $updated)` 找出只在 `$existing` 中存在的 ID
- 这些 ID 对应的 journal 会被 `JournalDestroyService` 删除

### 4.2 GroupUpdateService::updateTransactions 方法

**文件路径**: [GroupUpdateService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Internal/Update/GroupUpdateService.php#L172-L221)

这是处理新 split 创建和已有 journal 更新的核心方法：

```php
private function updateTransactions(TransactionGroup $transactionGroup, array $transactions): array
{
    $updated = [];
    
    foreach ($transactions as $index => $transaction) {
        $journalId = (int) ($transaction['transaction_journal_id'] ?? 0);
        
        // 尝试查找已有 journal
        $journal = $transactionGroup->transactionJournals()->find($journalId);
        
        if (null === $journal) {
            // =============================================
            // 场景 A: 没有找到 journal - 创建新 split
            // =============================================
            
            // 继承 type 的关键逻辑
            if (!array_key_exists('type', $transaction)) {
                // 从该 group 的其他 journal 随机获取一个类型
                $randomJournal = $transactionGroup->transactionJournals()
                    ->inRandomOrder()
                    ->with(['transactionType'])
                    ->first();
                    
                if (null !== $randomJournal) {
                    $transaction['type'] = $randomJournal->transactionType->type;
                }
            }
            
            // 创建新的 transaction journal
            $newJournal = $this->createTransactionJournal($transactionGroup, $transaction);
            if ($newJournal instanceof TransactionJournal) {
                $updated[] = $newJournal->id;
            }
        }
        
        if (null !== $journal) {
            // =============================================
            // 场景 B: 找到已有 journal - 执行更新
            // =============================================
            
            $this->updateTransactionJournal($transactionGroup, $journal, $transaction);
            $updated[] = $journal->id;
        }
    }
    
    return $updated;
}
```

**新 split 继承 type 的机制**:
1. 如果请求中的新 split 没有指定 `type` 字段
2. 系统会从该 transaction group 的现有 journal 中随机选取一个
3. 将该 journal 的 transaction type 赋值给新 split
4. 这确保了同一个 group 内的所有 split 类型一致

### 4.3 GroupUpdateService::createTransactionJournal 方法

```php
private function createTransactionJournal(TransactionGroup $transactionGroup, array $data): ?TransactionJournal
{
    $submission = ['transactions' => [$data]];
    
    /** @var TransactionJournalFactory $factory */
    $factory = app(TransactionJournalFactory::class);
    $factory->setUser($transactionGroup->user);
    
    $collection = $factory->create($submission);
    
    // 将新创建的 journal 关联到 group
    $collection->each(static function (TransactionJournal $journal) use ($transactionGroup): void {
        $transactionGroup->transactionJournals()->save($journal);
    });
    
    return $collection->first();
}
```

---

## 5. JournalUpdateService::update 方法

**文件路径**: [JournalUpdateService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Internal/Update/JournalUpdateService.php#L161-L202)

### 5.1 已有 journal 的更新流程

```php
public function update(): void
{
    $this->data['reconciled'] ??= null;
    
    // 1. 验证并更新账户信息（如果有效）
    if ($this->hasValidAccounts()) {
        $this->updateAccounts();
        $this->updateType();  // 可能更新交易类型
        $this->transactionJournal->refresh();
    }
    
    // 2. 更新账单关联
    $this->updateBill();
    
    // 3. 更新核心字段（描述、日期、顺序）
    $this->updateField('description');
    $this->updateField('date');       // 日期更新
    $this->updateField('order');
    $this->transactionJournal->save();
    $this->transactionJournal->refresh();
    
    // 4. 更新关联数据
    $this->updateCategory();
    $this->updateBudget();
    $this->updateTags();
    $this->updateReconciled();
    $this->updateNotes();
    $this->updateMeta();
    
    // 5. 更新货币和金额（关键）
    $this->updateCurrency();
    $this->updateAmount();             // 金额更新
    $this->updateForeignAmount();
    
    Preferences::mark();
    $this->transactionJournal->refresh();
}
```

### 5.2 日期更新逻辑 (updateField)

**文件路径**: [JournalUpdateService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Internal/Update/JournalUpdateService.php#L622-L672)

```php
private function updateField(string $fieldName): void
{
    if (array_key_exists($fieldName, $this->data) && '' !== (string) $this->data[$fieldName]) {
        $value = $this->data[$fieldName];
        
        if ('date' === $fieldName) {
            $value = new Carbon($value);
            $value->setTimezone(config('app.timezone'));
            
            // 记录_internal_previous_date 用于后续余额计算
            $res = $value->gt($this->transactionJournal->date);
            $factory = app(TransactionJournalMetaFactory::class);
            $set = ['journal' => $this->transactionJournal, 'name' => '_internal_previous_date', 'data' => null];
            
            if ($res) {
                // 新日期 > 旧日期：保存旧日期
                $set['data'] = clone $this->transactionJournal->date;
            }
            
            $factory->updateOrCreate($set);
        }
        
        // 记录审计日志
        event(new TransactionGroupRequestsAuditLogEntry(...));
        
        $this->transactionJournal->{$fieldName} = $value;
    }
}
```

### 5.3 金额更新逻辑 (updateAmount)

**文件路径**: [JournalUpdateService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Internal/Update/JournalUpdateService.php#L483-L552)

```php
private function updateAmount(): void
{
    if (!$this->hasFields(['amount'])) {
        return;
    }
    
    $value = $this->data['amount'] ?? '';
    $amount = $this->getAmount($value);
    
    $origSourceTransaction = $this->getSourceTransaction();
    $destTransaction = $this->getDestinationTransaction();
    
    // 更新源交易（负数金额）
    $origSourceTransaction->amount = Steam::negative($amount);
    $origSourceTransaction->balance_dirty = true;  // 标记需要重新计算余额
    
    // 更新目标交易（正数金额）
    $destTransaction->amount = Steam::positive($amount);
    $destTransaction->balance_dirty = true;
    
    $destTransaction->save();
    $origSourceTransaction->save();
    
    // 记录审计日志
    if (0 !== bccomp($origSourceTransaction->amount, $originalSourceAmount)) {
        event(new TransactionGroupRequestsAuditLogEntry(...));
    }
}
```

**金额更新关键点**:
- 同时更新源交易（负）和目标交易（正）
- 设置 `balance_dirty = true` 标记需要重新计算余额
- 金额变化时记录审计日志

---

## 6. JournalDestroyService 删除逻辑

**文件路径**: [JournalDestroyService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Internal/Destroy/JournalDestroyService.php#L35-L49)

```php
public function destroy(TransactionJournal $journal): void
{
    // 如果删除后 group 为空，也删除 group
    $group = $journal->transactionGroup;
    if (null !== $group) {
        $count = $group->transactionJournals->count();
        if (0 === $count) {
            $group->delete();
        }
    }
    
    // 删除 journal（软删除）
    $journal->delete();
}
```

**删除触发时机**（在 GroupUpdateService::update 中）:
```php
$existing = $transactionGroup->transactionJournals->pluck('id')->toArray();
$updated = $this->updateTransactions($transactionGroup, $transactions);
$result = array_diff($existing, $updated);  // 遗漏的 journal ID

foreach ($result as $deletedId) {
    $journal = $transactionGroup->transactionJournals()->find((int) $deletedId);
    $service = app(JournalDestroyService::class);
    $service->destroy($journal);  // 执行删除
}
```

---

## 7. UpdatedSingleTransactionGroup 事件处理

### 7.1 事件定义

**文件路径**: [UpdatedSingleTransactionGroup.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Events/Model/TransactionGroup/UpdatedSingleTransactionGroup.php)

```php
class UpdatedSingleTransactionGroup extends Event
{
    public function __construct(
        public TransactionGroupEventFlags $flags,
        public TransactionGroupEventObjects $objects
    ) {}
}
```

### 7.2 ProcessesUpdatedTransactionGroup 监听器

**文件路径**: [ProcessesUpdatedTransactionGroup.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesUpdatedTransactionGroup.php#L40-L75)

```php
public function handle(UpdatedSingleTransactionGroup $event): void
{
    // 1. 首先统一账户
    $effect = $this->unifyAccounts($event);
    
    // 2. 应用规则（如果标志为 true）
    if ($event->flags->applyRules) {
        $this->processRules($event->objects->transactionJournals, 'update-journal');
    }
    
    // 3. 重新计算信用额度（如果标志为 true）
    if ($event->flags->recalculateCredit) {
        $this->recalculateCredit($event->objects->accounts);
    }
    
    // 4. 触发 webhooks（如果标志为 true）
    if ($event->flags->fireWebhooks) {
        $this->createWebhookMessages($event->objects->transactionGroups, WebhookTrigger::UPDATE_TRANSACTION);
    }
    
    // 5. 清除周期统计
    $this->removePeriodStatistics($event->objects);
    
    // 6. 重新计算运行余额
    if (0 === $effect && true === $event->flags->unifyOnly) {
        // 账户没有变化且只需要统一，不重新计算
    }
    if (0 !== $effect || false === $event->flags->unifyOnly) {
        $this->recalculateRunningBalance($event->objects);
    }
}
```

### 7.3 unifyAccounts 逻辑

**文件路径**: [ProcessesUpdatedTransactionGroup.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesUpdatedTransactionGroup.php#L94-L155)

```php
private function unifyAccountsForGroup(TransactionGroup $group): int
{
    if (1 === $group->transactionJournals->count()) {
        return 0;  // 单 journal 不需要统一
    }
    
    // 获取第一个 journal 作为基准
    $first = $group->transactionJournals()
        ->orderBy('transaction_journals.date', 'DESC')
        ->orderBy('transaction_journals.order', 'ASC')
        ->orderBy('transaction_journals.id', 'DESC')
        ->first();
    
    $sourceAccount = $first->transactions()->where('amount', '<', '0')->first()->account;
    $destAccount = $first->transactions()->where('amount', '>', '0')->first()->account;
    
    $type = $first->transactionType->type;
    $effect = 0;
    
    // 对于 TRANSFER 或 WITHDRAWAL，统一所有源账户
    if (TransactionTypeEnum::TRANSFER->value === $type || TransactionTypeEnum::WITHDRAWAL->value === $type) {
        $effect += Transaction::query()
            ->whereIn('transaction_journal_id', $all)
            ->where('account_id', '!=', $sourceAccount->id)
            ->where('amount', '<', 0)
            ->update(['account_id' => $sourceAccount->id]);
    }
    
    // 对于 TRANSFER 或 DEPOSIT，统一所有目标账户
    if (TransactionTypeEnum::TRANSFER->value === $type || TransactionTypeEnum::DEPOSIT->value === $type) {
        $effect += Transaction::query()
            ->whereIn('transaction_journal_id', $all)
            ->where('account_id', '!=', $destAccount->id)
            ->where('amount', '>', 0)
            ->update(['account_id' => $destAccount->id]);
    }
    
    return $effect;  // 返回被修改的交易数量
}
```

### 7.4 各处理步骤的相互影响

#### 执行顺序与依赖关系

```
事件触发
    ↓
1. unifyAccounts 统一账户
    │  └─ 返回 $effect（被修改的交易数）
    │
    ├─ apply_rules（可选）
    │   └─ 对更新的 journal 应用规则引擎
    │
    ├─ recalculateCredit（可选）
    │   └─ 重新计算账户信用额度
    │
    ├─ fire_webhooks（可选）
    │   └─ 生成并发送 webhook 消息
    │
    ├─ removePeriodStatistics（总是执行）
    │   └─ 清除相关的周期统计数据
    │
    └─ recalculateRunningBalance（条件执行）
         └─ 条件: $effect !== 0 或 unifyOnly === false
```

#### 关键交互逻辑

1. **unifyAccounts 对后续步骤的影响**:
   - `$effect` 值表示有多少交易的账户被统一
   - `$effect` 直接影响是否需要重新计算运行余额

2. **apply_rules 的独立性**:
   - 规则应用独立于其他步骤
   - 规则应用可能修改交易属性，但不直接触发重新计算

3. **recalculateCredit 的条件性**:
   - 仅在 `oldHash !== newHash` 时执行
   - 即只有日期、金额、账户发生实质变化时才重新计算信用

4. **fire_webhooks 的独立性**:
   - webhook 触发独立于其他计算
   - 即使没有实质变化（hash 相同），只要 `fire_webhooks = true` 就会触发

5. **recalculateRunningBalance 的触发条件**:
   ```php
   if (0 !== $effect || false === $event->flags->unifyOnly) {
       $this->recalculateRunningBalance($event->objects);
   }
   ```
   - **情况 A**: `$effect > 0`（账户被统一）→ 重新计算
   - **情况 B**: `unifyOnly = false` → 重新计算
   - **情况 C**: `$effect = 0 且 unifyOnly = true` → 不重新计算

6. **removePeriodStatistics 总是执行**:
   - 无论其他标志如何，都会清除周期统计
   - 确保后续统计查询能正确反映更新

---

## 8. 完整场景追踪

### 场景假设
- **原有状态**: TransactionGroup 包含 3 个 split (journals: 101, 102, 103)
- **请求 payload**:
  - Split 1: `transaction_journal_id = 101`, `date = '2024-01-15'`, `amount = 150` (修改日期和金额)
  - Split 2: 无 `transaction_journal_id` (新 split，不指定 type)
  - Split 3: **遗漏**（不包含 journal 102）

### 执行流程

#### 阶段 1: UpdateRequest 处理
1. `getAll()` 提取数据
   - Split 1: 保留 `transaction_journal_id = 101`，包含 `date` 和 `amount`
   - Split 2: 无 `transaction_journal_id` 字段
2. `withValidator()` 验证通过

#### 阶段 2: UpdateController
1. 计算 `oldHash`（包含 journals 101, 102, 103 的日期、金额、账户）
2. 调用 `groupRepository->update()`

#### 阶段 3: GroupUpdateService
1. `updateTransactions()` 遍历请求中的 2 个 split:
   - **Split 1 (id=101)**: 找到已有 journal，调用 `updateTransactionJournal`
     - JournalUpdateService 更新 date 和 amount
   - **Split 2 (无 id)**: 未找到 journal，创建新 split
     - 从 group 中随机获取一个 journal 的 type（如 'withdrawal'）
     - 调用 `createTransactionJournal` 创建新 journal (id=104)
   - 返回 `$updated = [101, 104]`

2. 计算差异 `array_diff([101, 102, 103], [101, 104]) = [102, 103]`
3. 删除 journals 102 和 103（调用 JournalDestroyService）

#### 阶段 4: UpdateController 后续
1. 计算 `newHash`（包含 journals 101, 104 的日期、金额、账户）
2. `oldHash !== newHash` → `recalculateCredit = true`
3. 触发 `UpdatedSingleTransactionGroup` 事件

#### 阶段 5: ProcessesUpdatedTransactionGroup
1. `unifyAccounts()`: 如果 group 有多个 split，统一所有账户
2. 应用规则（如果 `apply_rules = true`）
3. 重新计算信用（因为 hash 变化了）
4. 触发 webhooks（如果 `fire_webhooks = true`）
5. 清除周期统计
6. 重新计算运行余额（因为账户可能变化了）

---

## 9. 关键代码参考汇总

| 组件 | 文件路径 | 关键方法 |
|------|---------|---------|
| UpdateRequest | [UpdateRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/Transaction/UpdateRequest.php) | `getAll()`, `withValidator()` |
| UpdateController | [UpdateController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Transaction/UpdateController.php) | `update()` |
| TransactionGroupRepository | [TransactionGroupRepository.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/TransactionGroup/TransactionGroupRepository.php) | `update()`, `getCompareHash()` |
| GroupUpdateService | [GroupUpdateService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Internal/Update/GroupUpdateService.php) | `update()`, `updateTransactions()`, `createTransactionJournal()` |
| JournalUpdateService | [JournalUpdateService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Internal/Update/JournalUpdateService.php) | `update()`, `updateField()`, `updateAmount()` |
| JournalDestroyService | [JournalDestroyService.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Internal/Destroy/JournalDestroyService.php) | `destroy()` |
| UpdatedSingleTransactionGroup | [UpdatedSingleTransactionGroup.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Events/Model/TransactionGroup/UpdatedSingleTransactionGroup.php) | 事件定义 |
| ProcessesUpdatedTransactionGroup | [ProcessesUpdatedTransactionGroup.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/TransactionGroup/ProcessesUpdatedTransactionGroup.php) | `handle()`, `unifyAccounts()` |
| SupportsGroupProcessingTrait | [SupportsGroupProcessingTrait.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/TransactionGroup/SupportsGroupProcessingTrait.php) | `processRules()`, `recalculateCredit()`, `recalculateRunningBalance()` |

---

## 10. 总结

### 核心机制
1. **已有 journal 更新**: 通过 `transaction_journal_id` 匹配，`JournalUpdateService::update()` 逐字段更新
2. **新 split 创建**: 无 `transaction_journal_id` 时，从 group 继承 type，通过 `TransactionJournalFactory` 创建
3. **遗漏 split 删除**: 通过 `array_diff(existing, updated)` 找出遗漏项，`JournalDestroyService::destroy()` 软删除

### recalculateCredit = false 的情况
- 当 `oldHash === newHash`，即交易的**日期**、**金额**、**账户**都没有实质变化
- Hash 计算包含：所有 journal 的日期、所有正数金额的总和、所有账户名称

### 事件处理交互
1. **unifyAccounts** 首先执行，确保多 split 账户一致
2. **apply_rules**、**fire_webhooks** 独立控制
3. **recalculateCredit** 基于 hash 变化判断
4. **recalculateRunningBalance** 取决于账户是否被修改（`$effect`）或是否强制更新
5. **removePeriodStatistics** 总是执行，确保数据一致性
