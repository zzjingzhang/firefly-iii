# Budget Limit 预算限额流程分析

## 概述

本文档详细分析 Firefly III 中 Budget Limit（预算限额）的两条创建路径：
1. **手动创建路径**：通过 API `POST v1/budgets/{budget}/limits` 创建
2. **自动生成路径**：通过 Cron 任务 `firefly-iii:cron --create-auto-budgets` 自动生成

最后分析 `ProcessesBudgetLimits` 监听器如何处理后续的 AvailableBudget 重算和 Webhook 触发。

---

## 一、手动创建路径：POST v1/budgets/{budget}/limits

### 1.1 路由定义

路由定义在 [routes/api.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/routes/api.php#L437-L437)：

```php
Route::post('{budget}/limits', ['uses' => 'BudgetLimit\StoreController@store', 'as' => 'limits.store']);
```

### 1.2 StoreRequest::withValidator - 防重复验证

请求验证类 [StoreRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/BudgetLimit/StoreRequest.php#L89-L128) 的 `withValidator` 方法在验证规则执行后进行防重复检查。

**核心逻辑：**

```php
public function withValidator(Validator $validator): void
{
    $validator->after(static function (Validator $validator) use ($budget): void {
        // 1. 解析币种（优先级：currency_id > currency_code > 默认币种）
        $factory  = app(TransactionCurrencyFactory::class);
        $currency = $factory->find($data['currency_id'] ?? null, $data['currency_code'] ?? null);
        if (null === $currency) {
            $currency = Amount::getPrimaryCurrency();
        }
        $repository = app(CurrencyRepositoryInterface::class);
        $repository->enable($currency);  // 启用币种

        // 2. 解析日期
        $start = Carbon::parse($data['start'], config('app.timezone'));
        $end   = Carbon::parse($data['end'], config('app.timezone'));

        // 3. 防重复检查：同一 budget + 同一日期范围 + 同一币种
        $limit = $budget
            ->budgetlimits()
            ->where('budget_limits.start_date', $start->format('Y-m-d'))
            ->where('budget_limits.end_date', $end->format('Y-m-d'))
            ->where('budget_limits.transaction_currency_id', $currency->id)
            ->first(['budget_limits.*']);

        if (null !== $limit) {
            $validator->errors()->add('start', trans('validation.limit_exists'));
        }
    });
}
```

**防重复的三个关键维度：**
- `budget_id`（从路由参数获取）
- `start_date` + `end_date`（精确到天）
- `transaction_currency_id`

### 1.3 StoreController::store - 数据转换

控制器 [StoreController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/BudgetLimit/StoreController.php#L66-L91) 的 `store` 方法处理请求数据。

**核心逻辑：**

```php
public function store(StoreRequest $request, Budget $budget): JsonResponse
{
    $data               = $request->getAll();
    $data['start_date'] = $data['start'];           // start → start_date
    $data['end_date']   = $data['end'];             // end → end_date
    $data['fire_webhooks'] ??= true;                // 默认触发 webhook
    $data['budget_id']  = $budget->id;              // 设置 budget_id

    $budgetLimit = $this->blRepository->store($data);
    // ... 后续转换和响应
}
```

**关键转换：**
- 将请求的 `start` / `end` 字段重命名为 Repository 期望的 `start_date` / `end_date`
- `fire_webhooks` 默认为 `true`，可通过请求参数覆盖
- 从路由参数注入的 `$budget` 对象中获取 `budget_id`

### 1.4 BudgetLimitRepository::store - 持久化与事件触发

Repository [BudgetLimitRepository.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Budget/BudgetLimitRepository.php#L298-L358) 的 `store` 方法执行实际的持久化操作。

**核心逻辑：**

```php
public function store(array $data): BudgetLimit
{
    // 1. 币种选择（与 StoreRequest 相同逻辑）
    $factory  = app(TransactionCurrencyFactory::class);
    $currency = $factory->find($data['currency_id'] ?? null, $data['currency_code'] ?? null);
    if (null === $currency) {
        $currency = Amount::getPrimaryCurrencyByUserGroup($this->user->userGroup);
    }
    $currency->enabled = true;
    $currency->save();  // 启用并保存币种

    // 2. 验证 Budget 存在
    $budget = $this->user->budgets()->find((int) $data['budget_id']);
    if (null === $budget) {
        throw new FireflyException('200004: Budget does not exist.');
    }

    // 3. 再次防重复检查（冗余但安全）
    $limit = $budget
        ->budgetlimits()
        ->where('budget_limits.start_date', $data['start_date']->format('Y-m-d'))
        ->where('budget_limits.end_date', $data['end_date']->format('Y-m-d'))
        ->where('budget_limits.transaction_currency_id', $currency->id)
        ->first(['budget_limits.*']);
    if (null !== $limit) {
        throw new FireflyException('200027: Budget limit already exists.');
    }

    // 4. 通过单例传递 fire_webhooks 标志给 Observer
    $singleton = PreferencesSingleton::getInstance();
    $singleton->setPreference('fire_webhooks_bl_store', $data['fire_webhooks'] ?? true);

    // 5. 创建 BudgetLimit
    $limit = new BudgetLimit();
    $limit->budget()->associate($budget);
    $limit->start_date              = $data['start_date']->format('Y-m-d');
    $limit->start_date_tz           = $data['start_date']->format('e');  // 时区
    $limit->end_date                = $data['end_date']->format('Y-m-d');
    $limit->end_date_tz             = $data['end_date']->format('e');
    $limit->amount                  = $data['amount'];
    $limit->generated               = $data['generated'] ?? false;       // 自动生成标志
    $limit->period                  = $data['period'] ?? '';              // 周期类型
    $limit->transaction_currency_id = $currency->id;
    $limit->save();

    // 6. 保存 Notes
    $noteText = (string) ($data['notes'] ?? '');
    if ('' !== $noteText) {
        $this->setNoteText($limit, $noteText);
    }

    // 7. 触发事件
    $createWebhookMessages = $data['fire_webhooks'] ?? true;
    event(new CreatedBudgetLimit($limit, $createWebhookMessages));
    event(new WebhookMessagesRequestSending());

    return $limit;
}
```

**关键要点：**

| 字段 | 说明 |
|------|------|
| `generated` | 布尔值，标识是否自动生成。手动创建时为 `false`，自动创建时为 `true` |
| `period` | 周期类型，如 `daily`、`weekly`、`monthly` 等。手动创建时空字符串 |
| `start_date_tz` / `end_date_tz` | 保存时区信息，格式如 `Asia/Shanghai` |
| `fire_webhooks` | 通过单例 `PreferencesSingleton` 传递给 Observer，同时通过事件参数传递 |

**触发的两个事件：**
1. `CreatedBudgetLimit` - 携带 BudgetLimit 对象和是否触发 webhook 的标志
2. `WebhookMessagesRequestSending` - 触发 webhook 消息发送任务

---

## 二、自动生成路径：firefly-iii:cron --create-auto-budgets

### 2.1 命令入口

Cron 命令定义在 [Cron.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Console/Commands/Tools/Cron.php#L53-L53)：

```php
protected $signature = 'firefly-iii:cron
    {--create-auto-budgets : Create auto budgets. Other tasks will be skipped unless also requested.}
';
```

调用链：`Cron::handle()` → `Cron::autoBudgetCronJob()` → `AutoBudgetCronjob::fire()`

### 2.2 AutoBudgetCronjob::fire - 执行频率控制

Cronjob [AutoBudgetCronjob.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Cronjobs/AutoBudgetCronjob.php#L39-L67) 控制执行频率。

**核心逻辑：**

```php
public function fire(): void
{
    // 1. 检查上次执行时间
    $config   = FireflyConfig::get('last_ab_job', 0);
    $lastTime = (int) $config->data;
    $diff     = now(config('app.timezone'))->getTimestamp() - $lastTime;

    // 2. 43200秒 = 12小时内不重复执行（除非 force=true）
    if ($lastTime > 0 && $diff <= 43_200) {
        if (false === $this->force) {
            $this->message = sprintf('It has been %s since the auto budget cron-job has fired.', $diffForHumans);
            return;
        }
    }

    // 3. 执行自动预算生成
    $this->fireAutoBudget();
    Preferences::mark();
}

private function fireAutoBudget(): void
{
    $job = app(CreateAutoBudgetLimits::class, [$this->date]);
    $job->setDate($this->date);
    $job->handle();

    // 记录本次执行时间
    FireflyConfig::set('last_ab_job', (int) $this->date->format('U'));
}
```

**频率控制规则：**
- 正常间隔：至少 12 小时（43200秒）才能执行一次
- 强制执行：使用 `--force` 参数可跳过间隔检查
- 记录执行时间到配置表 `last_ab_job`

### 2.3 CreateAutoBudgetLimits::handleAutoBudget - 核心生成逻辑

Job [CreateAutoBudgetLimits.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/CreateAutoBudgetLimits.php#L268-L334) 是自动预算生成的核心。

**执行流程：**

```
handle()
  └── 遍历所有 AutoBudget
       └── handleAutoBudget($autoBudget)
            ├── 检查 Budget 是否存在且激活
            ├── isMagicDay() 判断今天是否是生成日
            ├── 计算当前周期的 start / end
            ├── 检查当前周期是否已有 BudgetLimit
            └── 根据 auto_budget_type 分支处理
```

**核心代码：**

```php
private function handleAutoBudget(AutoBudget $autoBudget): void
{
    // 1. 有效性检查
    if (null === $autoBudget->budget) { /* 删除无效的 AutoBudget */ }
    if (false === $autoBudget->budget->active) { /* 跳过非激活 Budget */ }

    // 2. Magic Day 检查
    if (!$this->isMagicDay($autoBudget)) {
        Log::info(sprintf('Today is not a magic day for %s auto-budget #%d', $autoBudget->period, $autoBudget->id));
        return;
    }

    // 3. 计算当前周期范围
    $start = Navigation::startOfPeriod($this->date, $autoBudget->period);
    $end   = Navigation::endOfPeriod($start, $autoBudget->period);

    // 4. 检查当前周期是否已有 BudgetLimit
    $budgetLimit = $this->findBudgetLimit($autoBudget->budget, $start, $end);
    if ($budgetLimit instanceof BudgetLimit) {
        return;  // 已有则跳过
    }

    // 5. 根据类型分支处理
    switch ((int) $autoBudget->auto_budget_type) {
        case AutoBudgetType::AUTO_BUDGET_RESET->value:
            $this->createBudgetLimit($autoBudget, $start, $end);
            break;
        case AutoBudgetType::AUTO_BUDGET_ROLLOVER->value:
            $this->createRollover($autoBudget);
            break;
        case AutoBudgetType::AUTO_BUDGET_ADJUSTED->value:
            $this->createAdjustedLimit($autoBudget);
            break;
    }
}
```

### 2.4 isMagicDay - 周期触发日判断

[CreateAutoBudgetLimits.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/CreateAutoBudgetLimits.php#L339-L372) 的 `isMagicDay` 方法判断今天是否应该生成预算。

| 周期 | 判断条件 |
|------|----------|
| `daily` | 始终返回 `true`（每天都生成） |
| `weekly` | `$this->date->isMonday()`（每周一） |
| `monthly` | `1 === $this->date->day`（每月1号） |
| `quarterly` | 日期为 `01-01`、`04-01`、`07-01`、`10-01`（每季度第一天） |
| `half_year` | 日期为 `01-01`、`07-01`（每半年第一天） |
| `yearly` | 日期为 `01-01`（每年第一天） |

### 2.5 AutoBudgetType 分支详解

AutoBudgetType 枚举定义在 [AutoBudgetType.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Enums/AutoBudgetType.php#L30-L34)：

```php
enum AutoBudgetType: int
{
    case AUTO_BUDGET_RESET    = 1;  // 重置模式
    case AUTO_BUDGET_ROLLOVER = 2;  // 滚存模式
    case AUTO_BUDGET_ADJUSTED = 3;  // 调整模式
}
```

#### 分支1：AUTO_BUDGET_RESET（重置模式）

最简单的模式，直接创建一个新的 BudgetLimit，金额为 AutoBudget 配置的金额。

```php
// 调用链：handleAutoBudget → createBudgetLimit
private function createBudgetLimit(AutoBudget $autoBudget, Carbon $start, Carbon $end, ?string $amount = null): void
{
    $repository = app(BudgetLimitRepositoryInterface::class);
    $repository->store([
        'currency_id' => $autoBudget->transaction_currency_id,
        'budget_id'   => $autoBudget->budget->id,
        'start_date'  => clone $start,
        'end_date'    => clone $end,
        'amount'      => $amount ?? $autoBudget->amount,  // 使用配置金额
        'period'      => $autoBudget->period,              // 记录周期类型
        'generated'   => true,                             // 标记为自动生成
    ]);
}
```

#### 分支2：AUTO_BUDGET_ROLLOVER（滚存模式）

上一周期未花完的金额可以滚存到下一周期。

**核心逻辑**（[CreateAutoBudgetLimits.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/CreateAutoBudgetLimits.php#L187-L249)）：

```php
private function createRollover(AutoBudget $autoBudget): void
{
    // 1. 计算上一周期范围
    $start         = Navigation::startOfPeriod($this->date, $autoBudget->period);
    $end           = Navigation::endOfPeriod($start, $autoBudget->period);
    $previousStart = Navigation::subtractPeriod($start, $autoBudget->period);
    $previousEnd   = Navigation::endOfPeriod($previousStart, $autoBudget->period);

    // 2. 上一周期没有 BudgetLimit，则按标准金额创建
    $budgetLimit = $this->findBudgetLimit($autoBudget->budget, $previousStart, $previousEnd);
    if (!$budgetLimit instanceof BudgetLimit) {
        $this->createBudgetLimit($autoBudget, $start, $end);
        return;
    }

    // 3. 计算上一周期实际花费
    $repository = app(OperationsRepositoryInterface::class);
    $repository->setUser($autoBudget->budget->user);
    $spent = $repository->sumExpenses(
        $previousStart,
        $previousEnd,
        null,
        new Collection()->push($autoBudget->budget),
        $autoBudget->transactionCurrency
    );
    $spentAmount = $spent[$currencyId]['sum'] ?? '0';  // 注意：spentAmount 是负数（支出）

    // 4. 计算剩余金额：上一周期限额 + 实际花费（负数）
    $budgetLeft  = bcadd($budgetLimit->amount, $spentAmount);
    $totalAmount = $autoBudget->amount;

    // 5. 根据剩余金额决定本期金额
    if (-1 !== bccomp('0', $budgetLeft)) {
        // 剩余金额 <= 0（超支），重置为配置金额
        Log::info(sprintf('The amount left is negative, so it will be reset to %s.', $totalAmount));
    }
    if (1 !== bccomp('0', $budgetLeft)) {
        // 剩余金额 > 0（有结余），本期金额 = 剩余金额 + 配置金额
        $totalAmount = bcadd($budgetLeft, $totalAmount);
        Log::info(sprintf('The amount left is positive, so the new amount will be %s.', $totalAmount));
    }

    // 6. 创建本期 BudgetLimit
    $this->createBudgetLimit($autoBudget, $start, $end, $totalAmount);
}
```

**ROLLOVER 模式金额计算：**

| 上一周期情况 | 计算方式 | 本期金额 |
|-------------|----------|----------|
| 没有 BudgetLimit | - | `autoBudget->amount`（配置金额） |
| 有 BudgetLimit，`budgetLeft <= 0`（超支） | 重置 | `autoBudget->amount`（配置金额） |
| 有 BudgetLimit，`budgetLeft > 0`（结余） | 滚存 | `budgetLeft + autoBudget->amount` |

**注意**：`sumExpenses` 返回的 `spentAmount` 是**负数**（因为是支出），所以 `bcadd(budgetLimit->amount, spentAmount)` 等价于 `预算限额 - 实际支出`。

#### 分支3：AUTO_BUDGET_ADJUSTED（调整模式）

根据上一周期的实际花费调整本期预算，超支严重时设置最低限额为 1。

**核心逻辑**（[CreateAutoBudgetLimits.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/CreateAutoBudgetLimits.php#L91-L159)）：

```php
private function createAdjustedLimit(AutoBudget $autoBudget): void
{
    // 1. 计算上一周期范围（与 ROLLOVER 相同）
    $start         = Navigation::startOfPeriod($this->date, $autoBudget->period);
    $end           = Navigation::endOfPeriod($start, $autoBudget->period);
    $previousStart = Navigation::subtractPeriod($start, $autoBudget->period);
    $previousEnd   = Navigation::endOfPeriod($previousStart, $autoBudget->period);

    // 2. 上一周期没有 BudgetLimit，则按标准金额创建
    $budgetLimit = $this->findBudgetLimit($autoBudget->budget, $previousStart, $previousEnd);
    if (!$budgetLimit instanceof BudgetLimit) {
        $this->createBudgetLimit($autoBudget, $start, $end);
        return;
    }

    // 3. 计算上一周期实际花费（与 ROLLOVER 相同）
    $repository  = app(OperationsRepositoryInterface::class);
    $repository->setUser($autoBudget->budget->user);
    $spent       = $repository->sumExpenses(...);
    $spentAmount = $spent[$currencyId]['sum'] ?? '0';  // 负数

    // 4. 计算可用金额：上一周期限额 + 本期配置金额 + 实际花费（负数）
    $budgetAvailable = bcadd(bcadd($budgetLimit->amount, $autoBudget->amount), $spentAmount);
    $totalAmount     = $autoBudget->amount;

    // 5. 三种情况分支
    if (-1 !== bccomp($budgetAvailable, $totalAmount)) {
        // 情况1: budgetAvailable >= totalAmount
        // 没有超支，使用计算出的可用金额
        $this->createBudgetLimit($autoBudget, $start, $end, $budgetAvailable);
    }
    if (1 !== bccomp($budgetAvailable, $totalAmount) && 1 === bccomp($budgetAvailable, '0')) {
        // 情况2: budgetAvailable < totalAmount 且 budgetAvailable > 0
        // 有超支但还能覆盖部分，使用计算出的金额
        $this->createBudgetLimit($autoBudget, $start, $end, $budgetAvailable);
    }
    if (1 !== bccomp($budgetAvailable, $totalAmount) && -1 === bccomp($budgetAvailable, '0')) {
        // 情况3: budgetAvailable < totalAmount 且 budgetAvailable <= 0
        // 严重超支，设置最低限额为 1
        $this->createBudgetLimit($autoBudget, $start, $end, '1');
    }
}
```

**ADJUSTED 模式金额计算：**

| 条件 | 说明 | 本期金额 |
|------|------|----------|
| 上一周期无 BudgetLimit | - | `autoBudget->amount` |
| `budgetAvailable >= totalAmount` | 无超支，有结余 | `budgetAvailable` |
| `0 < budgetAvailable < totalAmount` | 有超支但未耗尽 | `budgetAvailable` |
| `budgetAvailable <= 0` | 严重超支 | `'1'`（最低限额） |

**公式说明**：
- `budgetAvailable = 上期限额 + 本期配置金额 + 上期花费（负数）`
- 实质上是：`上期限额 - 上期实际花费 + 本期配置金额`

---

## 三、ProcessesBudgetLimits - 后续处理监听器

监听器 [ProcessesBudgetLimits.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/BudgetLimit/ProcessesBudgetLimits.php#L40-L86) 监听 BudgetLimit 的创建、更新和删除事件。

### 3.1 事件监听

```php
class ProcessesBudgetLimits implements ShouldQueue
{
    public function handle(CreatedBudgetLimit|DestroyedBudgetLimit|UpdatedBudgetLimit $event): void
    {
        // ... 处理逻辑
    }
}
```

监听三种事件：
- `CreatedBudgetLimit` - 创建后
- `UpdatedBudgetLimit` - 更新后
- `DestroyedBudgetLimit` - 删除后

### 3.2 AvailableBudget 重算

使用 `AvailableBudgetCalculator` 重新计算可用预算。

**核心逻辑：**

```php
public function handle($event): void
{
    if ($event instanceof DestroyedBudgetLimit) {
        // 删除事件：重算用户所有相关可用预算
        $calculator = new AvailableBudgetCalculator();
        $calculator->setUser($event->user);
        $calculator->setStart($event->start->clone());
        $calculator->setEnd($event->end->clone());
        $calculator->setCreate(false);           // 删除时不创建新的 AvailableBudget
        $calculator->setCurrency(Amount::getPrimaryCurrencyByUserGroup(...));
        $calculator->recalculateByRange();
    } else {
        // 创建/更新事件：重算该 BudgetLimit 覆盖范围内的可用预算
        $calculator = new AvailableBudgetCalculator();
        $calculator->setUser($event->budgetLimit->budget->user);
        $calculator->setStart($event->budgetLimit->start_date->clone());
        $calculator->setEnd($event->budgetLimit->end_date->clone());
        $calculator->setCreate(true);            // 创建/更新时需要创建 AvailableBudget
        $calculator->setCurrency($event->budgetLimit->transactionCurrency);
        $calculator->recalculateByRange();
    }

    // Webhook 触发（见下文）
}
```

### 3.3 AvailableBudgetCalculator::recalculateByRange

[AvailableBudgetCalculator.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Models/AvailableBudgetCalculator.php#L56-L81) 按用户视图范围（viewRange）循环重算。

```php
public function recalculateByRange(): void
{
    // 1. 根据用户 viewRange 对齐日期范围
    $start = Navigation::startOfPeriod($this->start, $this->viewRange);
    $end   = Navigation::startOfPeriod($this->end, $this->viewRange);
    $end   = Navigation::endOfPeriod($end, $this->viewRange);

    // 2. 按 viewRange 循环，逐段刷新 AvailableBudget
    $current = clone $start;
    while ($current <= $end) {
        $this->refreshAvailableBudget($current);
        $current = Navigation::addPeriod($current, $this->viewRange);
    }
}

private function refreshAvailableBudget(Carbon $start): void
{
    $end = Navigation::endOfPeriod($start, $this->viewRange);

    // 查找并更新现有 AvailableBudget
    $availableBudgets = $this->abRepository->findInRange($this->currency, $start, $end);
    foreach ($availableBudgets as $item) {
        $this->abRepository->recalculateAmount($item);
    }

    // 需要时创建新的 AvailableBudget
    if ($this->create) {
        $availableBudget = $this->abRepository->find($this->currency, $start, $end);
        if (null === $availableBudget) {
            $availableBudget = $this->abRepository->store([...]);
            $this->abRepository->recalculateAmount($availableBudget);
        }
    }
}
```

### 3.4 Webhook 触发

[ProcessesBudgetLimits.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/BudgetLimit/ProcessesBudgetLimits.php#L56-L86) 中根据事件参数决定是否触发 Webhook。

```php
public function handle($event): void
{
    // ... 先重算 AvailableBudget

    if ($event instanceof DestroyedBudgetLimit) {
        if ($event->createWebhookMessages) {
            $this->createWebhookMessages($event->user, $event->budget, WebhookTrigger::STORE_UPDATE_BUDGET_LIMIT);
        }
        return;
    }

    if ($event->createWebhookMessages) {
        $this->createWebhookMessages(
            $event->budgetLimit->budget->user,
            $event->budgetLimit->budget,
            WebhookTrigger::STORE_UPDATE_BUDGET_LIMIT
        );
    }
}

private function createWebhookMessages(User $user, Budget $budget, WebhookTrigger $trigger): void
{
    $engine = app(MessageGeneratorInterface::class);
    $engine->setUser($user);
    $engine->setObjects(new Collection()->push($budget));
    $engine->setTrigger($trigger);
    $engine->generateMessages();
}
```

**关键要点：**

| 项目 | 说明 |
|------|------|
| Webhook 触发类型 | `WebhookTrigger::STORE_UPDATE_BUDGET_LIMIT`（值为 230） |
| 触发条件 | 事件的 `createWebhookMessages` 属性为 `true` |
| 关联对象 | 传递 `Budget` 对象（而非 BudgetLimit） |
| 触发时机 | AvailableBudget 重算完成后 |

`WebhookTrigger` 枚举定义在 [WebhookTrigger.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Enums/WebhookTrigger.php#L39-L39)：

```php
case STORE_UPDATE_BUDGET_LIMIT = 230;
```

---

## 四、完整流程图

### 4.1 手动创建流程

```
POST v1/budgets/{budget}/limits
        │
        ▼
StoreRequest::rules()           基本验证（必填、日期、金额格式）
        │
        ▼
StoreRequest::withValidator()   防重复检查（budget+日期+币种）
        │  ├─ 币种解析与启用
        │  └─ 重复查询与错误
        ▼
StoreController::store()        数据转换
        │  ├─ start → start_date
        │  ├─ end → end_date
        │  ├─ fire_webhooks 默认为 true
        │  └─ budget_id 设置
        ▼
BudgetLimitRepository::store()  持久化
        │  ├─ 币种选择与启用
        │  ├─ Budget 存在性检查
        │  ├─ 再次防重复检查
        │  ├─ 创建 BudgetLimit（generated=false, period=''）
        │  ├─ 保存 Notes
        │  ├─ event(CreatedBudgetLimit)
        │  └─ event(WebhookMessagesRequestSending)
        ▼
ProcessesBudgetLimits::handle() 后续处理
        │  ├─ AvailableBudgetCalculator::recalculateByRange()
        │  └─ createWebhookMessages(STORE_UPDATE_BUDGET_LIMIT)
```

### 4.2 自动生成流程

```
firefly-iii:cron --create-auto-budgets
        │
        ▼
Cron::autoBudgetCronJob()
        │
        ▼
AutoBudgetCronjob::fire()       频率控制（12小时间隔）
        │
        ▼
CreateAutoBudgetLimits::handle()
        │
        ▼
遍历每个 AutoBudget
        │
        ▼
handleAutoBudget()
        ├─ Budget 有效性检查
        ├─ isMagicDay() 判断
        │   ├─ daily: always true
        │   ├─ weekly: Monday
        │   ├─ monthly: day == 1
        │   ├─ quarterly: 01-01/04-01/07-01/10-01
        │   ├─ half_year: 01-01/07-01
        │   └─ yearly: 01-01
        ├─ 当前周期是否已有 BudgetLimit？
        │   └─ 有则跳过
        └─ AutoBudgetType 分支
             ├─ RESET: createBudgetLimit(amount = autoBudget->amount)
             ├─ ROLLOVER:
             │   ├─ 上一周期花费计算
             │   ├─ budgetLeft = 上期限额 + 花费（负数）
             │   ├─ budgetLeft <= 0: 重置为配置金额
             │   └─ budgetLeft > 0: 本期 = 剩余 + 配置金额
             └─ ADJUSTED:
                 ├─ 上一周期花费计算
                 ├─ budgetAvailable = 上期 + 本期 + 花费
                 ├─ >= 配置金额: 使用 budgetAvailable
                 ├─ 0 < x < 配置金额: 使用 budgetAvailable
                 └─ <= 0: 设置为 '1'
        │
        ▼
BudgetLimitRepository::store()  持久化（generated=true, period='xxx'）
        │
        ▼
ProcessesBudgetLimits::handle() 后续处理（同上）
```

---

## 五、关键代码位置汇总

| 功能 | 文件 | 行号 |
|------|------|------|
| API 路由定义 | [routes/api.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/routes/api.php#L437-L437) | L437 |
| 手动创建请求验证 | [StoreRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/BudgetLimit/StoreRequest.php#L89-L128) | L89-L128 |
| 手动创建控制器 | [StoreController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/BudgetLimit/StoreController.php#L66-L91) | L66-L91 |
| BudgetLimit 仓储 | [BudgetLimitRepository.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Budget/BudgetLimitRepository.php#L298-L358) | L298-L358 |
| Cron 命令入口 | [Cron.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Console/Commands/Tools/Cron.php#L110-L118) | L110-L118 |
| 自动预算 Cronjob | [AutoBudgetCronjob.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Cronjobs/AutoBudgetCronjob.php#L39-L86) | L39-L86 |
| 自动预算生成 Job | [CreateAutoBudgetLimits.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/CreateAutoBudgetLimits.php#L74-L373) | L74-L373 |
| Magic Day 判断 | [CreateAutoBudgetLimits.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/CreateAutoBudgetLimits.php#L339-L372) | L339-L372 |
| ROLLOVER 逻辑 | [CreateAutoBudgetLimits.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/CreateAutoBudgetLimits.php#L187-L249) | L187-L249 |
| ADJUSTED 逻辑 | [CreateAutoBudgetLimits.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/CreateAutoBudgetLimits.php#L91-L159) | L91-L159 |
| 后续处理监听器 | [ProcessesBudgetLimits.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/Model/BudgetLimit/ProcessesBudgetLimits.php#L40-L86) | L40-L86 |
| 可用预算计算器 | [AvailableBudgetCalculator.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Models/AvailableBudgetCalculator.php#L56-L176) | L56-L176 |
| AutoBudgetType 枚举 | [AutoBudgetType.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Enums/AutoBudgetType.php#L30-L34) | L30-L34 |
| WebhookTrigger 枚举 | [WebhookTrigger.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Enums/WebhookTrigger.php#L30-L39) | L30-L39 |
| CreatedBudgetLimit 事件 | [CreatedBudgetLimit.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Events/Model/BudgetLimit/CreatedBudgetLimit.php#L32-L42) | L32-L42 |
