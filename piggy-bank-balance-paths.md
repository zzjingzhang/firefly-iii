# PiggyBank 余额变更路径分析

本文档分析 Firefly III 中两条会改变 PiggyBank 余额的核心路径：

1. **PUT v1/piggy-banks/{piggyBank}**：直接更新存钱罐属性及账户金额
2. **交易创建中关联 piggy_bank_id / piggy_bank_name**：通过交易自动存入/取出

---

## 路径一：PUT v1/piggy-banks/{piggyBank}

### 1.1 请求入口与验证

#### UpdateController::update

[UpdateController](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/PiggyBank/UpdateController.php#L64-L85)

控制器接收 `UpdateRequest`，调用 `$request->getAll()` 获取数据，再调用 `$this->repository->update($piggyBank, $data)`。

#### UpdateRequest::rules — 三条验证规则的组合

[UpdateRequest::rules](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/PiggyBank/UpdateRequest.php#L73-L94)

```php
return [
    'current_amount'            => ['nullable', new LessThanPiggyTarget(), new IsValidPositiveAmount()],
    'target_amount'             => ['nullable', new IsValidZeroOrMoreAmount()],
    'accounts'                  => 'array',
    'accounts.*'                => 'array',
    'accounts.*.account_id'     => ['required', 'numeric', 'belongsToUser:accounts,id'],
    'accounts.*.current_amount' => ['numeric', 'nullable', new IsValidZeroOrMoreAmount(true), new IsEnoughInAccounts($piggyBank, $this->getAll())],
    // ... 其它字段
];
```

三个自定义规则的协作方式：

| 规则 | 位置 | 作用 |
|------|------|------|
| **LessThanPiggyTarget** | [LessThanPiggyTarget](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Rules/LessThanPiggyTarget.php#L33-L49) | 作用于顶层 `current_amount`，意图确保当前金额不超过目标金额。当前实现中 `validate()` 方法体为空（标注了 TODO），实际上不做校验。 |
| **IsValidPositiveAmount** | [IsValidPositiveAmount](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Rules/IsValidPositiveAmount.php#L34-L98) | 作用于顶层 `current_amount`，确保是合法的正数（非空、非科学计数法、大于0、不超过上限）。 |
| **IsEnoughInAccounts** | [IsEnoughInAccounts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Rules/PiggyBank/IsEnoughInAccounts.php#L34-L73) | 作用于每个 `accounts.*.current_amount`，是核心业务校验。接收 `$piggyBank` 和 `$this->getAll()` 构造，遍历每个账户条目：计算请求金额与该账户已存金额 `savedSoFar` 的差值 `diff = amount - savedSoFar`；若 `diff > 0`（需要增加），则调用 `canAddAmount` 检查账户余额是否充足且目标空间是否足够，不满足则报错 `cannot_add_piggy_amount`。 |
| **IsValidZeroOrMoreAmount** | [IsValidZeroOrMoreAmount](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Rules/IsValidZeroOrMoreAmount.php#L32-L94) | 作用于 `target_amount` 和 `accounts.*.current_amount`（nullable 模式），确保是 ≥0 的合法数值。 |

**组合逻辑总结**：请求中的 `accounts` 数组是真正控制每账户金额的载体；顶层 `current_amount` 的验证已基本失效（LessThanPiggyTarget 为空实现）。`IsEnoughInAccounts` 是唯一执行实际业务校验的规则——它需要先通过 `parseAccounts` 拿到解析后的数据，再逐账户判断"增加的金额"是否可以放入存钱罐。

#### UpdateRequest::getAll 与 parseAccounts

[UpdateRequest::getAll](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/PiggyBank/UpdateRequest.php#L49-L68)

```php
public function getAll(): array
{
    $fields = [
        'name' => ['name', 'convertString'],
        'target_amount' => ['target_amount', 'convertString'],
        // ...
    ];
    $result = $this->getAllData($fields);
    $result['accounts'] = $this->parseAccounts($this->get('accounts'));
    return $result;
}
```

[parseAccounts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Request/ConvertsDataTypes.php#L457-L484) 的处理逻辑：

- 遍历请求中的 `accounts` 数组
- 对每个条目提取 `account_id`（转为整数）和 `current_amount`
- 如果请求中传了 `current_amount` 键且值为 `null`，则 `amount` 设为 `null`（表示"清零"意图）
- 如果请求中没有 `current_amount` 键，则 `amount` 也为 `null`（表示"不改变"）
- 返回结构：`[['account_id' => int, 'current_amount' => ?string]]`

**关键点**：`parseAccounts` 在 `rules()` 和 `getAll()` 中都会被间接调用。`rules()` 里 `IsEnoughInAccounts` 构造时传入 `$this->getAll()`，后者调用 `parseAccounts` 解析出账户数据供校验使用。验证通过后，`getAll()` 再次解析，结果传入 `update()` 方法。

---

### 1.2 ModifiesPiggyBanks::update — 属性、notes、order、object group 的更新

[ModifiesPiggyBanks::update](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/PiggyBank/ModifiesPiggyBanks.php#L280-L348)

更新流程按以下顺序执行：

#### 步骤 1：updateProperties — 更新基本属性

[updateProperties](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/PiggyBank/ModifiesPiggyBanks.php#L367-L395)

| 字段 | 行为 |
|------|------|
| `name` | 非空时触发 `PiggyBankNameIsChanged` 事件（含旧名和新名），然后赋值 |
| `transaction_currency_id` | 整数时更新货币 |
| `target_amount` | 非空字符串赋值；空字符串设为 `'0'` |
| `target_date` / `start_date` | 非空时赋值，同时保存时区 `target_date_tz` / `start_date_tz` |

最后调用 `$piggyBank->save()` 持久化。

#### 步骤 2：updateNote — 更新笔记

[updateNote](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/PiggyBank/ModifiesPiggyBanks.php#L350-L365)

- 空字符串 → 删除已有 Note
- 有内容 → 创建或更新 Note（polymorphic `noteable` 关联）

#### 步骤 3：setOrder — 调整排序

[setOrder](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/PiggyBank/ModifiesPiggyBanks.php#L231-L267)

若请求中的 `order` 与当前不同：
- 新序号 > 旧序号：将范围内其它存钱罐 `order` 递减
- 新序号 < 旧序号：将范围内其它存钱罐 `order` 递增

#### 步骤 4：linkToAccountIds — 同步账户及金额

这是余额变更的核心，详见下节。

#### 步骤 5：removeAmountFromAll — 修剪超额金额

当 `target_amount` 缩小导致当前总额超标时触发，详见下节。

#### 步骤 6：object group — 关联对象组

先尝试按 `object_group_title` 查找或创建 ObjectGroup 并 sync；再尝试按 `object_group_id` 查找并 sync。若标题为空字符串则解除关联。

---

### 1.3 PiggyBankFactory::linkToAccountIds — 保留旧金额、检查可添加性、同步账户、触发事件

[linkToAccountIds](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/PiggyBankFactory.php#L104-L199)

#### 阶段 A：收集旧金额，预填充已关联账户

```php
$oldSavedAmount = $this->piggyBankRepository->getCurrentAmount($piggyBank);
foreach ($piggyBank->accounts as $account) {
    foreach ($accounts as $info) {
        if ((int) $account->id === (int) $info['account_id']) {
            $toBeLinked[$account->id] = ['current_amount' => $account->pivot->current_amount ?? '0'];
        }
    }
}
```

**目的**：遍历当前已关联的账户，若请求中也包含该账户，则先把 pivot 上的 `current_amount` 保留到 `$toBeLinked`，避免后续 `sync()` 操作将旧金额清零。

[getCurrentAmount](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/PiggyBank/PiggyBankRepository.php#L135-L155)：遍历 piggyBank 关联的所有账户（可按单个账户过滤），累加 `pivot->current_amount`。

#### 阶段 B：遍历请求中的账户，处理 current_amount

对每个请求中的账户条目，分三种情况：

**情况 1：请求中 `current_amount` 有值（非 null）**

```php
$previous = $toBeLinked[$account->id]['current_amount'] ?? '0';
$diff = bcsub($info['current_amount'], $previous);
```

- 计算新值与旧值的差 `diff`
- 若 `diff > 0`（需要增加金额），调用 `canAddAmount` 检查：
  - 账户余额是否足够（`leftOnAccount`）
  - 目标空间是否足够（`target_amount - savedSoFar`）
  - 两者取较小值作为上限，`amount` 不得超过此上限
- 若 `canAddAmount` 返回 `false`，跳过该账户（不修改金额）
- 差值不为 0 时原本会触发 `ChangedAmount` 事件，但代码注释显示因 issue #10990 已禁用
- 更新 `$toBeLinked[$account->id] = ['current_amount' => $info['current_amount']]`

**情况 2：请求中 `current_amount` 为 null**

- `diff = 0 - previous`（即减少旧金额那么多）
- 不执行 `canAddAmount` 检查（因为只在增加时才需要）
- 保留旧金额在 `$toBeLinked` 中（`$toBeLinked[$account->id]['current_amount'] ?? '0'`）

**情况 3：请求中没有 `current_amount` 键**

- 仅确保 `$toBeLinked` 中有该账户的条目，不修改金额

#### 阶段 C：sync 并触发 PiggyBankAmountIsChanged

```php
$piggyBank->accounts()->sync($toBeLinked);
$piggyBank->refresh();
$newSavedAmount = $this->piggyBankRepository->getCurrentAmount($piggyBank);
if (0 !== bccomp($oldSavedAmount, $newSavedAmount)) {
    event(new PiggyBankAmountIsChanged($piggyBank, bcsub($newSavedAmount, $oldSavedAmount), null, null));
}
```

1. 调用 Eloquent 的 `sync()` 方法，以 `$toBeLinked` 数组同步多对多关联，pivot 数据中包含 `current_amount`
2. 刷新模型获取最新数据
3. 比较同步前后的总金额 `oldSavedAmount` vs `newSavedAmount`
4. **若总额发生变化，触发 `PiggyBankAmountIsChanged` 事件**，amount 为差值（正为增加，负为减少），journal 和 group 均为 null

---

### 1.4 removeAmountFromAll — target_amount 缩小时的修剪逻辑

[ModifiesPiggyBanks::update](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/PiggyBank/ModifiesPiggyBanks.php#L303-L315)

在 `linkToAccountIds` 之后执行：

```php
$currentAmount = $this->getCurrentAmount($piggyBank);
if (1 === bccomp((string) $currentAmount, (string) $piggyBank->target_amount)
    && 0 !== bccomp((string) $piggyBank->target_amount, '0')) {
    $difference = bcsub((string) $piggyBank->target_amount, (string) $currentAmount);
    $this->removeAmountFromAll($piggyBank, Steam::positive($difference));
}
```

**触发条件**：当前总额 > target_amount 且 target_amount ≠ 0。

`$difference = target_amount - current_amount` 为负数，取绝对值后传入 `removeAmountFromAll`。

[removeAmountFromAll](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/PiggyBank/ModifiesPiggyBanks.php#L172-L186)

```php
public function removeAmountFromAll(PiggyBank $piggyBank, string $amount): void
{
    foreach ($piggyBank->accounts as $account) {
        $current = $account->pivot->current_amount;
        if (1 === bccomp((string) $current, $amount)) {
            $this->removeAmount($piggyBank, $account, $amount);
            return;
        }
        $this->removeAmount($piggyBank, $account, $current);
        $amount = bcsub($amount, (string) $current);
    }
}
```

**算法**：贪心式从关联账户列表中逐个扣除。

1. 遍历存钱罐的所有关联账户
2. 若当前账户的 `current_amount` > 需要移除的 `amount`：
   - 从该账户移除 `amount`，结束
3. 否则，移除该账户的全部 `current_amount`，从 `amount` 中减去已移除的部分，继续下一个账户

每次 `removeAmount` 调用会：
- 更新 pivot 的 `current_amount = currentAmount - amount`
- 更新 `native_current_amount`（如有货币转换需求）
- 触发 `PiggyBankAmountIsChanged` 事件（amount 为负值）

---

### 1.5 路径一完整流程图

```
PUT /v1/piggy-banks/{piggyBank}
  │
  ├─ UpdateRequest::rules()
  │   ├─ LessThanPiggyTarget (空实现)
  │   ├─ IsValidPositiveAmount (顶层 current_amount)
  │   ├─ IsValidZeroOrMoreAmount (target_amount, accounts.*.current_amount)
  │   └─ IsEnoughInAccounts (逐账户检查 canAddAmount)
  │
  ├─ UpdateRequest::getAll()
  │   └─ parseAccounts() → [{account_id, current_amount}]
  │
  └─ ModifiesPiggyBanks::update()
      ├─ updateProperties() → name, target_amount, dates, currency
      ├─ updateNote() → notes
      ├─ setOrder() → order
      ├─ PiggyBankFactory::linkToAccountIds()
      │   ├─ 保留旧 current_amount 到 toBeLinked
      │   ├─ 逐账户: 计算diff → canAddAmount检查(仅增加时) → 更新toBeLinked
      │   ├─ sync(toBeLinked) → 更新pivot
      │   └─ 若总额变化 → PiggyBankAmountIsChanged事件
      ├─ removeAmountFromAll() (仅当 currentAmount > target_amount)
      │   └─ 贪心扣除 → 每次removeAmount触发PiggyBankAmountIsChanged
      └─ object group sync
```

---

## 路径二：交易创建中关联 PiggyBank

### 2.1 入口：TransactionJournalFactory::storePiggyEvent

[storePiggyEvent](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/TransactionJournalFactory.php#L607-L620)

```php
private function storePiggyEvent(TransactionJournal $journal, NullArrayObject $data): void
{
    $piggyBank = $this->piggyRepository->findPiggyBank(
        (int) $data['piggy_bank_id'],
        $data['piggy_bank_name']
    );
    if ($piggyBank instanceof PiggyBank) {
        $this->piggyEventFactory->create($journal, $piggyBank);
        return;
    }
}
```

在 `createJournal()` 中，当 journal 和两条 transaction 都创建完成后，调用 `storePiggyEvent`：
1. 通过 `piggy_bank_id` 或 `piggy_bank_name` 查找存钱罐
2. 若找到，委托给 `PiggyBankEventFactory::create`

### 2.2 PiggyBankEventFactory::create

[PiggyBankEventFactory::create](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/PiggyBankEventFactory.php#L38-L59)

```php
public function create(TransactionJournal $journal, ?PiggyBank $piggyBank): void
{
    $piggyRepos = app(PiggyBankRepositoryInterface::class);
    $piggyRepos->setUser($journal->user);

    $amount = $piggyRepos->getExactAmount($piggyBank, $journal);
    if (0 === bccomp($amount, '0')) {
        return;
    }
    $piggyRepos->addAmountToPiggyBank($piggyBank, $amount, $journal);
}
```

1. 调用 `getExactAmount` 计算本次交易应该对存钱罐增加或减少的金额
2. 若金额为 0，不操作
3. 调用 `addAmountToPiggyBank` 执行增加或移除

### 2.3 PiggyBankRepository::getExactAmount — 精确计算金额

[getExactAmount](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/PiggyBank/PiggyBankRepository.php#L185-L287)

该方法是交易路径的核心，决定金额的正负和大小。逻辑分以下阶段：

#### 阶段 A：确定操作方向（正/负）和匹配的货币

```php
$source = $journal->transactions()->where('amount', '<', 0)->first();
$destination = $journal->transactions()->where('amount', '>', 0)->first();

foreach ($piggyBank->accounts as $account) {
    if ($account->id === $source->account_id) {
        $operator = 'negative';
        $currency = $accountRepos->getAccountCurrency($source->account) ?? $primaryCurrency;
    }
    if ($account->id === $destination->account_id) {
        $operator = 'positive';
        $currency = $accountRepos->getAccountCurrency($destination->account) ?? $primaryCurrency;
    }
}
```

| 匹配情况 | 操作方向 | 含义 |
|----------|----------|------|
| 存钱罐账户 = **source** 账户 | `negative` | 钱从存钱罐关联的账户流出 → 从存钱罐**取出** |
| 存钱罐账户 = **destination** 账户 | `positive` | 钱流入存钱罐关联的账户 → 向存钱罐**存入** |
| 同时匹配 source 和 destination（`$hits > 1`） | 返回 `'0'` | 转账两端都在存钱罐中，无法判断方向 |

#### 阶段 B：确定使用普通金额还是外币金额

```php
if ((int) $source->transaction_currency_id === $currency->id) {
    $amount = Steam::{$operator}($source->amount);
}
if ((int) $source->foreign_currency_id === $currency->id) {
    $amount = Steam::{$operator}($source->foreign_amount);
}
```

- 先检查交易的主货币（`transaction_currency_id`）是否与存钱罐关联账户的货币匹配
- 再检查外币（`foreign_currency_id`）是否匹配
- `Steam::positive()` 或 `Steam::negative()` 将金额转为正/负值
- 若两种货币都不匹配，返回 `'0'`

#### 阶段 C：目标空间限制

```php
$currentAmount = $this->getCurrentAmount($piggyBank);
$room = bcsub($piggyBank->target_amount, $currentAmount);

if (0 === bccomp($piggyBank->target_amount, '0')) {
    $room = Steam::positive($amount);  // target=0 表示无上限
}
```

**正向金额（存入）限制**：

```php
if (1 === bccomp($amount, '0') && -1 === bccomp($room, $amount)) {
    return $room;  // 存入金额超过剩余空间，只返回 room
}
```

- 若 `target_amount > 0`：`room = target_amount - currentAmount`
- 若存入金额 > room，则实际只存入 room（不超过目标）

**负向金额（取出）限制**：

```php
$compare = bcmul($currentAmount, '-1');
if (-1 === bccomp($amount, '0') && 1 === bccomp($compare, $amount)) {
    return $compare;  // 取出金额超过已存金额，最多取出 currentAmount
}
```

- 负数的 compare = -currentAmount
- 若取出金额的绝对值 > currentAmount，则最多取出 currentAmount

### 2.4 addAmountToPiggyBank — 执行增加或移除

[addAmountToPiggyBank](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/PiggyBank/ModifiesPiggyBanks.php#L75-L90)

```php
public function addAmountToPiggyBank(PiggyBank $piggyBank, string $amount, TransactionJournal $journal): void
{
    if (-1 === bccomp($amount, '0')) {
        $source = $journal->transactions()->where('amount', '<', 0)->first();
        $this->removeAmount($piggyBank, $source->account, bcmul($amount, '-1'), $journal);
    }
    if (1 === bccomp($amount, '0')) {
        $destination = $journal->transactions()->where('amount', '>', 0)->first();
        $this->addAmount($piggyBank, $destination->account, $amount, $journal);
    }
}
```

| 金额符号 | 操作 | 账户来源 |
|----------|------|----------|
| 负数 | `removeAmount(piggyBank, source.account, |amount|, journal)` | 交易的 source 账户 |
| 正数 | `addAmount(piggyBank, destination.account, amount, journal)` | 交易的 destination 账户 |

#### addAmount 的内部逻辑

[addAmount](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/PiggyBank/ModifiesPiggyBanks.php#L52-L73)

```php
$currentAmount = $this->getCurrentAmount($piggyBank, $account);
$pivot = $piggyBank->accounts()->where('accounts.id', $account->id)->first()->pivot;
$pivot->current_amount = bcadd((string) $currentAmount, $amount);
$pivot->native_current_amount = null;

// 若用户主货币 ≠ 存钱罐货币，转换并保存 native_current_amount
$userCurrency = Amount::getPrimaryCurrencyByUserGroup($this->user->userGroup);
if ($userCurrency->id !== $piggyBank->transaction_currency_id) {
    $converter = new ExchangeRateConverter();
    $converter->setIgnoreSettings(true);
    $pivot->native_current_amount = $converter->convert(...);
}
$pivot->save();
event(new PiggyBankAmountIsChanged($piggyBank, $amount, $journal, null));
```

- 更新 pivot 的 `current_amount`（加上新增金额）
- 处理货币转换（`native_current_amount`）
- 触发 `PiggyBankAmountIsChanged` 事件，**携带了 `$journal`**

#### removeAmount 的内部逻辑

[removeAmount](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/PiggyBank/ModifiesPiggyBanks.php#L149-L170)

与 `addAmount` 对称：
- `pivot->current_amount = bcsub(currentAmount, amount)`
- 同样处理 `native_current_amount`
- 触发 `PiggyBankAmountIsChanged` 事件，amount 为取反后的负值，**携带了 `$journal`**

### 2.5 路径二完整流程图

```
创建交易（split 中含 piggy_bank_id / piggy_bank_name）
  │
  └─ TransactionJournalFactory::createJournal()
      │  创建 journal + 两条 transaction
      │
      └─ storePiggyEvent(journal, row)
          │
          ├─ piggyRepository->findPiggyBank(id, name)
          │
          └─ PiggyBankEventFactory::create(journal, piggyBank)
              │
              ├─ getExactAmount(piggyBank, journal)
              │   ├─ 匹配 source/destination → 确定正/负方向
              │   ├─ 匹配 transaction_currency_id/foreign_currency_id → 确定金额
              │   └─ target room 限制 → 截断金额
              │
              └─ addAmountToPiggyBank(piggyBank, amount, journal)
                  ├─ amount < 0 → removeAmount(piggyBank, source.account, |amount|, journal)
                  └─ amount > 0 → addAmount(piggyBank, destination.account, amount, journal)
                      │
                      └─ 更新 pivot + 触发 PiggyBankAmountIsChanged(journal!=null)
```

---

## CreatesPiggyBankEventForChangedAmount — 避免重复事件

### 3.1 事件监听注册

`CreatesPiggyBankEventForChangedAmount` 实现了 `ShouldQueue` 接口，通过 Laravel 的事件自动发现机制注册。Laravel 根据 `handle` 方法的类型提示 `PiggyBankAmountIsChanged` 自动将监听器绑定到该事件。

### 3.2 去重逻辑

[CreatesPiggyBankEventForChangedAmount](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/PiggyBank/CreatesPiggyBankEventForChangedAmount.php#L35-L60)

```php
public function handle(PiggyBankAmountIsChanged $event): void
{
    $journal = $event->transactionJournal;
    if ($event->transactionGroup instanceof TransactionGroup) {
        $journal = $event->transactionGroup->transactionJournals()->first();
    }
    $date = $journal->date ?? today(config('app.timezone'));

    if (null !== $journal) {
        $exists = PiggyBankEvent::query()
            ->where('piggy_bank_id', $event->piggyBank->id)
            ->where('transaction_journal_id', $journal->id)
            ->exists();
        if ($exists) {
            Log::warning('Already have event for this journal and piggy, will not create another.');
            return;
        }
    }

    PiggyBankEvent::create([
        'piggy_bank_id'          => $event->piggyBank->id,
        'transaction_journal_id' => $journal?->id,
        'date'                   => $date->format('Y-m-d'),
        'date_tz'                => $date->format('e'),
        'amount'                 => $event->amount,
    ]);
}
```

**去重机制**：

1. 从事件中提取 `transactionJournal`（若事件携带的是 `transactionGroup`，则取 group 的第一个 journal）
2. **若 journal 不为 null**：查询 `piggy_bank_events` 表，检查是否已存在相同的 `(piggy_bank_id, transaction_journal_id)` 组合
3. 若已存在，打印警告日志并 **直接返回**，不创建新记录
4. 若不存在（或 journal 为 null），创建新的 `PiggyBankEvent` 记录

### 3.3 为什么需要去重

在路径二中，`PiggyBankEventFactory::create` 已经通过 `addAmount`/`removeAmount` 改变了存钱罐余额并触发了 `PiggyBankAmountIsChanged` 事件。此事件的监听器 `CreatesPiggyBankEventForChangedAmount` 会创建 `PiggyBankEvent` 记录。

而在路径一中（`linkToAccountIds`），当总额发生变化时也会触发 `PiggyBankAmountIsChanged`，但此时 journal 为 null，所以不会触发去重检查，允许创建无 journal 关联的事件。

去重的核心场景是：**同一笔交易 journal 不应对同一个存钱罐产生两条 PiggyBankEvent**。例如：
- 交易创建路径中，`addAmountToPiggyBank` 先触发了一次事件
- 如果有其它地方也因同一 journal 触发事件，去重会阻止重复记录

---

## 两条路径对比总结

| 维度 | 路径一：PUT 更新存钱罐 | 路径二：交易关联存钱罐 |
|------|----------------------|----------------------|
| **触发方式** | API 直接请求 | 创建交易时传入 piggy_bank_id/name |
| **金额确定方式** | 请求中 `accounts.*.current_amount` 直接指定 | `getExactAmount` 根据交易方向和货币自动计算 |
| **增加金额的校验** | `IsEnoughInAccounts` + `canAddAmount` | `getExactAmount` 内的 room 限制 |
| **减少金额的方式** | `removeAmountFromAll` 贪心扣除 | `removeAmount` 按账户精确扣除 |
| **事件携带 journal** | 通常为 null | 始终携带 `$journal` |
| **PiggyBankEvent 去重** | journal=null 时不去重 | journal 存在时按 (piggy_bank_id, journal_id) 去重 |
| **target_amount 缩小处理** | `removeAmountFromAll` 修剪超额 | `getExactAmount` 中 room 截断，不会超额 |
