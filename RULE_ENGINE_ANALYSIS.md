# Firefly III 规则引擎深度分析

## 1. API 行为对比：testRule vs triggerRule

### 1.1 testRule (GET v1/rules/{rule}/test)

**位置**: [TriggerController.php:testRule](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Rule/TriggerController.php#L70-L115)

**核心行为**:
```php
$transactions = $ruleEngine->find();  // 只查找，不修改
// 返回交易列表，分页处理
```

**关键特性**:
- 调用 `$ruleEngine->find()` 方法
- 仅查找匹配的交易，**不产生任何副作用**
- 返回完整的交易列表（支持分页）
- 支持通过参数过滤：`start`、`end`、`accounts`
- 调用 `TransactionGroupEnrichment` 增强数据

### 1.2 triggerRule (POST v1/rules/{rule}/trigger)

**位置**: [TriggerController.php:triggerRule](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Rule/TriggerController.php#L123-L150)

**核心行为**:
```php
$ruleEngine->fire();  // 执行规则，产生副作用
// 返回 204 No Content
```

**关键特性**:
- 调用 `$ruleEngine->fire()` 方法
- **执行规则动作，产生数据库修改等副作用**
- 不返回修改后的交易数据，只返回 HTTP 204
- 同样支持 `start`、`end`、`accounts` 参数过滤

### 1.3 对比总结

| 特性 | testRule (find) | triggerRule (fire) |
|------|-----------------|-------------------|
| HTTP 方法 | GET | POST |
| 核心方法 | `$ruleEngine->find()` | `$ruleEngine->fire()` |
| 副作用 | 无 | 有（修改数据库）|
| 返回值 | 交易列表（JSON） | 204 No Content |
| 目的 | 预览匹配的交易 | 实际执行规则 |

---

## 2. SearchRuleEngine 核心方法追踪

**位置**: [SearchRuleEngine.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php)

### 2.1 findStrictRule - 严格模式查找

**位置**: [findStrictRule](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L304-L367)

**执行流程**:
1. 获取规则的所有触发器（按 order 排序）
2. 构建搜索数组，所有触发器条件**同时满足**
3. 支持 `needs_context` 配置判断是否需要值
4. 添加额外的操作符（如日期范围、账户过滤）
5. 调用搜索引擎执行查询
6. 返回匹配的交易集合

**关键代码**:
```php
// 所有触发器条件合并到同一个搜索查询
$searchArray[$ruleTrigger->trigger_type][] = sprintf('"%s"', $ruleTrigger->trigger_value);
// 单次搜索即可得到所有条件匹配的结果
```

### 2.2 findNonStrictRule - 非严格模式查找

**位置**: [findNonStrictRule](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L205-L299)

**执行流程**:
1. 遍历每个触发器，**单独执行搜索**
2. 每个触发器的结果合并到总集合
3. 支持 `stop_processing` 提前终止
4. 最后对结果去重

**关键特性**:
- **OR 逻辑**：任一触发器匹配即命中
- 支持 `stop_processing`：当某个触发器匹配到结果且 `stop_processing=true` 时，停止后续触发器处理

**关键代码**:
```php
foreach ($triggers as $ruleTrigger) {
    // 每个触发器单独搜索
    $result = $searchEngine->searchTransactions();
    $total = $total->merge($collection);
    
    // 支持 stop_processing 提前终止
    if (true === $ruleTrigger->stop_processing && $result->count() > 0) {
        break;
    }
}
// 最后去重
$unique = $total->unique(...);
```

### 2.3 fireRule - 执行规则

**位置**: [fireRule](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L427-L443)

**执行流程**:
```
fireRule
  ├─ 检查规则是否激活
  ├─ 根据 strict 值分支
  │   ├─ true  → fireStrictRule()
  │   └─ false → fireNonStrictRule()
  └─ 返回规则是否被触发
```

### 2.4 fireStrictRule / fireNonStrictRule

**位置**: [fireStrictRule](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L450-L478)

**关键步骤**:
1. 调用对应的 find 方法获取匹配交易
2. 调用 `processResults()` 处理结果
3. 触发 `UpdatedSingleTransactionGroup` 事件
4. 返回是否有匹配的交易

### 2.5 processTransactionJournal - 处理交易日志

**位置**: [processTransactionJournal](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L576-L591)

**执行流程**:
```
processTransactionJournal
  ├─ 获取规则的所有动作（按 order 排序）
  └─ 遍历每个动作
      ├─ 检查动作是否激活
      ├─ processRuleAction() 执行动作
      └─ 如返回 true（break），停止后续动作
```

### 2.6 processRuleAction - 执行单个动作

**位置**: [processRuleAction](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L526-L558)

**关键逻辑**:
```php
$actionClass = ActionFactory::getAction($ruleAction);
$result = $actionClass->actOnArray($transaction);

// 只有当动作成功执行 AND stop_processing=true 时才终止
if (true === $ruleAction->stop_processing && true === $result) {
    return true;  // break
}
// 注意：动作执行失败但 stop_processing=true 不会终止！
```

**动作终止条件总结**:

| 动作执行结果 | stop_processing | 是否终止后续动作 |
|-------------|-----------------|-----------------|
| true（成功） | true | ✅ 终止 |
| true（成功） | false | ❌ 继续 |
| false（失败）| true | ❌ 继续（关键！）|
| false（失败）| false | ❌ 继续 |

---

## 3. config/search.php 配置解析

### 3.1 needs_context 配置影响

**位置**: [search.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/search.php#L26-L263)

**作用**: 决定搜索操作符是否需要上下文值

#### needs_context = true

需要提供具体的值进行搜索，例如：
- `description_is: "Groceries"` - 描述等于某个值
- `amount_is: "100"` - 金额等于某个值
- `category_is: "Food"` - 分类等于某个值

**代码应用** ([findNonStrictRule](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php#L242-L250)):
```php
$needsContext = config(sprintf('search.operators.%s.needs_context', $ruleTrigger->trigger_type)) ?? true;
if (true === $needsContext) {
    $searchArray[$ruleTrigger->trigger_type] = sprintf('"%s"', $ruleTrigger->trigger_value);
}
```

#### needs_context = false

不需要值，作为布尔标志使用，例如：
- `reconciled:true` - 已对账
- `has_attachments:true` - 有附件
- `has_any_budget:true` - 有预算

**代码应用**:
```php
if (false === $needsContext) {
    $searchArray[$ruleTrigger->trigger_type] = 'true';
}
```

#### 典型配置示例

```php
'reconciled'           => ['alias' => false, 'needs_context' => false],
'description_is'       => ['alias' => false, 'needs_context' => true],
'has_attachments'      => ['alias' => false, 'needs_context' => false],
'amount_is'            => ['alias' => false, 'needs_context' => true],
```

### 3.2 alias 配置影响

**作用**: 为搜索操作符定义别名，支持多种写法

#### alias = false

该操作符是主操作符，不是别名

#### alias = true

该操作符是别名，实际映射到 `alias_for` 指定的操作符

**配置示例**:
```php
'description_is'   => ['alias' => false, 'needs_context' => true],
'description'      => ['alias' => true, 'alias_for' => 'description_is', 'needs_context' => true],

'tag_is'           => ['alias' => false, 'needs_context' => true],
'tag'              => ['alias' => true, 'alias_for' => 'tag_is', 'needs_context' => true],

'source_account_contains' => ['alias' => false, 'needs_context' => true],
'source'                   => ['alias' => true, 'alias_for' => 'source_account_contains', 'needs_context' => true],
'from'                     => ['alias' => true, 'alias_for' => 'source_account_contains', 'needs_context' => true],
```

#### 别名机制的好处

1. **用户友好**: 支持更自然的搜索语法
   - `tag:Groceries` 等同于 `tag_is:Groceries`
   - `from:Alice` 等同于 `source_account_contains:Alice`

2. **向后兼容**: 可以重命名操作符而不破坏现有查询

3. **多语言支持**: 理论上可以支持不同语言的搜索词

---

## 4. rule-actions 与 ActionFactory 映射机制

### 4.1 配置定义

**位置**: [firefly.php:rule-actions](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/firefly.php#L417-L450)

```php
'rule-actions' => [
    'set_category'            => SetCategory::class,
    'clear_category'          => ClearCategory::class,
    'set_budget'              => SetBudget::class,
    'clear_budget'            => ClearBudget::class,
    'add_tag'                 => AddTag::class,
    'remove_tag'              => RemoveTag::class,
    'set_description'         => SetDescription::class,
    'set_source_account'      => SetSourceAccount::class,
    'set_destination_account' => SetDestinationAccount::class,
    'set_notes'               => SetNotes::class,
    'link_to_bill'            => LinkToBill::class,
    'convert_withdrawal'      => ConvertToWithdrawal::class,
    'delete_transaction'      => DeleteTransaction::class,
    // ... 更多动作
],
```

### 4.2 Domain 桥接层

**位置**: [Domain.php:getRuleActions](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Domain.php#L36-L39)

```php
public static function getRuleActions(): array
{
    return config('firefly.rule-actions');
}
```

### 4.3 ActionFactory 工厂类

**位置**: [ActionFactory.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Factory/ActionFactory.php)

#### 类图关系

```
RuleAction (数据库模型)
    ↓ action_type (字符串, 如 'set_budget')
ActionFactory::getAction()
    ↓ 查找配置
    ↓ 实例化类
ActionInterface (接口)
    ↓ 实现类
SetBudget (具体动作)
```

#### 核心方法

**getAction()** - 获取动作实例:
```php
public static function getAction(RuleAction $action): ActionInterface
{
    $class = self::getActionClass($action->action_type);
    return new $class($action);
}
```

**getActionClass()** - 获取动作类名:
```php
public static function getActionClass(string $actionType): string
{
    $actionTypes = self::getActionTypes();
    
    if (!array_key_exists($actionType, $actionTypes)) {
        throw new FireflyException('No such action exists...');
    }
    
    $class = $actionTypes[$actionType];
    if (!class_exists($class)) {
        throw new FireflyException('Could not instantiate class...');
    }
    
    return $class;
}
```

### 4.4 完整映射流程

```
数据库 RuleAction
  ├─ action_type: 'set_budget'
  └─ action_value: 'Monthly Groceries'
        ↓
ActionFactory::getAction($ruleAction)
  ├─ getActionClass('set_budget')
  │   └─ Domain::getRuleActions()
  │       └─ config('firefly.rule-actions')
  │           └─ ['set_budget' => SetBudget::class]
  └─ new SetBudget($ruleAction)
        ↓
SetBudget 实例 (实现 ActionInterface)
  └─ actOnArray($transaction)
```

### 4.5 ActionInterface 接口

所有动作都必须实现 `actOnArray` 方法:
```php
interface ActionInterface
{
    public function actOnArray(array $journal): bool;
}
```

---

## 5. SetBudget 失败分支深入分析

**位置**: [SetBudget.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Actions/SetBudget.php)

### 5.1 整体执行流程

```
actOnArray($journal)
  ├─ 查找目标预算
  │   └─ 找不到 → 失败分支1
  ├─ 检查交易类型是否为 Withdrawal
  │   └─ 不是 → 失败分支2
  ├─ 检查是否已关联同一预算
  │   └─ 已关联 → 失败分支3
  ├─ 删除旧预算关联
  ├─ 插入新预算关联
  ├─ 触发审计日志事件
  └─ 返回 true (成功)
```

### 5.2 三个失败分支详解

#### 失败分支1：找不到预算

**触发条件**: 用户输入的预算名称不存在

**代码位置**: [SetBudget.php:54-63](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Actions/SetBudget.php#L54-L63)

```php
$budget = $user->budgets()->where('name', $search)->first();
if (null === $budget) {
    event(new RuleActionFailedOnArray(
        $this->action, 
        $journal, 
        trans('rules.cannot_find_budget', ['name' => $search])
    ));
    return false;
}
```

**事件触发**:
- 事件: `RuleActionFailedOnArray`
- 错误消息: `rules.cannot_find_budget`（"无法找到预算: {name}"）

#### 失败分支2：非 Withdrawal 交易

**触发条件**: 交易类型不是取款（Withdrawal）

**代码位置**: [SetBudget.php:65-78](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Actions/SetBudget.php#L65-L78)

```php
if (TransactionTypeEnum::WITHDRAWAL->value !== $journal['transaction_type_type']) {
    event(new RuleActionFailedOnArray(
        $this->action, 
        $journal, 
        trans('rules.cannot_set_budget', [
            'type' => $journal['transaction_type_type'],
            'name' => $search,
        ])
    ));
    return false;
}
```

**事件触发**:
- 事件: `RuleActionFailedOnArray`
- 错误消息: `rules.cannot_set_budget`（"无法为 {type} 类型交易设置预算: {name}"）

**设计原因**: 预算系统只适用于支出（取款），收入（存款）和转账不适用

#### 失败分支3：已关联同一预算

**触发条件**: 交易已经关联了目标预算

**代码位置**: [SetBudget.php:81-89](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Actions/SetBudget.php#L81-L89)

```php
$oldBudget = $object->budgets()->first();
if ((int) $oldBudget?->id === $budget->id) {
    event(new RuleActionFailedOnArray(
        $this->action, 
        $journal, 
        trans('rules.already_linked_to_budget', ['name' => $budget->name])
    ));
    return false;
}
```

**事件触发**:
- 事件: `RuleActionFailedOnArray`
- 错误消息: `rules.already_linked_to_budget`（"已关联预算: {name}"）

**设计原因**: 避免无意义的重复操作，保持操作幂等性

### 5.3 RuleActionFailedOnArray 事件详解

**位置**: [RuleActionFailedOnArray.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Events/Model/Rule/RuleActionFailedOnArray.php)

**事件结构**:
```php
class RuleActionFailedOnArray
{
    public function __construct(
        public RuleAction $ruleAction,  // 规则动作模型
        public array $journal,          // 交易数据
        public string $error            // 本地化错误消息
    ) {}
}
```

**事件用途**:
1. **日志记录**: 可通过监听器记录失败原因到日志
2. **调试追踪**: 开发人员可追踪规则执行失败的具体原因
3. **用户反馈**: 可收集并展示给用户哪些动作执行失败
4. **统计分析**: 可统计规则动作的成功率

### 5.4 审计事件（成功分支）

**位置**: [TransactionGroupRequestsAuditLogEntry.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Events/Model/TransactionGroup/TransactionGroupRequestsAuditLogEntry.php)

**触发时机**（成功分支）:
```php
// 删除旧关联，插入新关联后
event(new TransactionGroupRequestsAuditLogEntry(
    $this->action->rule,        // 变更者：规则
    $object,                    // 被审计对象：交易日志
    'set_budget',               // 字段名
    $oldBudgetName,             // 变更前值
    $budget->name               // 变更后值
));
```

**审计事件结构**:
```php
class TransactionGroupRequestsAuditLogEntry extends Event
{
    public function __construct(
        public Model $changer,    // 触发变更的对象（如 Rule）
        public Model $auditable,  // 被变更的对象（如 TransactionJournal）
        public string $field,     // 变更的字段
        public mixed $before,     // 变更前的值
        public mixed $after       // 变更后的值
    ) {}
}
```

### 5.5 成功 vs 失败事件对比

| 场景 | 触发事件 | 事件类型 | 目的 |
|------|---------|---------|------|
| 找不到预算 | `RuleActionFailedOnArray` | 失败事件 | 记录失败原因 |
| 交易类型错误 | `RuleActionFailedOnArray` | 失败事件 | 记录失败原因 |
| 已关联同一预算 | `RuleActionFailedOnArray` | 失败事件 | 记录失败原因 |
| 设置成功 | `TransactionGroupRequestsAuditLogEntry` | 审计事件 | 记录变更历史 |

---

## 6. 完整数据流图

### 6.1 testRule 数据流

```
HTTP GET /api/v1/rules/{rule}/test
    ↓
TriggerController::testRule()
    ├─ 设置规则到 RuleEngine
    ├─ 添加可选过滤操作符（日期/账户）
    ├─ $ruleEngine->find()
    │   └─ 遍历规则
    │       ├─ strict=true  → findStrictRule()
    │       │   └─ 构建搜索查询 → SearchEngine → 返回交易
    │       └─ strict=false → findNonStrictRule()
    │           └─ 遍历触发器 → 各自搜索 → 合并去重
    └─  enrich() + 分页 + 返回 JSON
```

### 6.2 triggerRule 数据流

```
HTTP POST /api/v1/rules/{rule}/trigger
    ↓
TriggerController::triggerRule()
    ├─ 设置规则到 RuleEngine
    ├─ 添加可选过滤操作符
    └─ $ruleEngine->fire()
        └─ 遍历规则
            ├─ fireRule()
            │   ├─ findStrictRule() / findNonStrictRule()
            │   └─ processResults()
            │       └─ processTransactionGroup()
            │           └─ processTransactionJournal()
            │               └─ 遍历动作
            │                   ├─ ActionFactory::getAction()
            │                   │   └─ SetBudget::actOnArray()
            │                   │       ├─ 失败 → RuleActionFailedOnArray 事件
            │                   │       └─ 成功 → 更新 DB + 审计事件
            │                   └─ 检查 stop_processing
            └─ fire UpdatedSingleTransactionGroup 事件
```

---

## 7. 关键配置文件索引

| 配置文件 | 关键配置项 | 用途 |
|---------|-----------|------|
| [search.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/search.php) | `operators` | 定义搜索操作符、别名、是否需要上下文 |
| [firefly.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/firefly.php) | `rule-actions` | 规则动作类型到类的映射 |

---

## 8. 核心类索引

| 类名 | 文件位置 | 职责 |
|-----|---------|------|
| `TriggerController` | [TriggerController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Rule/TriggerController.php) | API 入口控制器 |
| `SearchRuleEngine` | [SearchRuleEngine.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Engine/SearchRuleEngine.php) | 规则引擎核心 |
| `ActionFactory` | [ActionFactory.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Factory/ActionFactory.php) | 动作类工厂 |
| `Domain` | [Domain.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Domain.php) | 配置桥接层 |
| `SetBudget` | [SetBudget.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/TransactionRules/Actions/SetBudget.php) | 设置预算动作 |
| `RuleActionFailedOnArray` | [RuleActionFailedOnArray.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Events/Model/Rule/RuleActionFailedOnArray.php) | 动作失败事件 |
| `TransactionGroupRequestsAuditLogEntry` | [TransactionGroupRequestsAuditLogEntry.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Events/Model/TransactionGroup/TransactionGroupRequestsAuditLogEntry.php) | 审计日志事件 |
