# 币种API和外部汇率任务回归分析文档

## 第一部分：币种API流程分析

### 1.1 端点概述

| HTTP方法 | 端点 | Controller方法 |
|---------|------|----------------|
| POST | `/api/v1/currencies/{currency_code}/primary` | [UpdateController::makePrimary](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/TransactionCurrency/UpdateController.php#L124-L143) |
| PUT | `/api/v1/currencies/{currency_code}` | [UpdateController::update](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/TransactionCurrency/UpdateController.php#L153-L183) |
| POST | `/api/v1/currencies/{currency_code}/enable` | [UpdateController::enable](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/TransactionCurrency/UpdateController.php#L105-L122) |
| POST | `/api/v1/currencies/{currency_code}/disable` | [UpdateController::disable](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/TransactionCurrency/UpdateController.php#L72-L97) |

### 1.2 makePrimary 执行流程

**代码位置**: [UpdateController::makePrimary](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/TransactionCurrency/UpdateController.php#L124-L143)

执行顺序：
1. 调用 `$this->repository->enable($currency)` - 先启用币种
2. 调用 `$this->repository->makePrimary($currency)` - 再设置为主币种
3. 调用 `Preferences::mark()` - 标记偏好更新
4. 返回转换后的币种数据

**关键设计**: 设置主币种前必须先启用该币种，确保币种可用。

### 1.3 update 方法 409 状态码分析

**代码位置**: [UpdateController::update](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/TransactionCurrency/UpdateController.php#L153-L183)

返回 409 的两种情况：

#### 情况一：禁用最后一个启用币种
```php
$set = $this->repository->get();
if (array_key_exists('enabled', $data) 
    && false === $data['enabled'] 
    && 1 === count($set) 
    && $set->first()->id === $currency->id) {
    return response()->json([], 409);
}
```
**原因**: 系统必须至少保留一个启用的币种，否则用户将无法进行任何交易操作。

#### 情况二：禁用正在使用的币种
```php
if (array_key_exists('enabled', $data) 
    && false === $data['enabled'] 
    && $this->repository->currencyInUse($currency)) {
    return response()->json([], 409);
}
```
**原因**: 如果币种已被使用（在交易、账户等中），禁用会导致数据不一致。

### 1.4 CurrencyRepository::currencyInUseAt 检查项

**代码位置**: [CurrencyRepository::currencyInUseAt](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Currency/CurrencyRepository.php#L71-L178)

按顺序检查以下各项，任意一项存在则返回对应标识：

| 检查项 | 代码位置 | 说明 |
|-------|---------|------|
| **journals** | [L74-L79](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Currency/CurrencyRepository.php#L74-L79) | 检查交易日记账中是否使用该币种（包括本币和外币） |
| **last_left** | [L82-L86](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Currency/CurrencyRepository.php#L82-L86) | 是否是系统中最后剩下的币种 |
| **account_meta** | [L89-L110](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Currency/CurrencyRepository.php#L89-L110) | 账户元数据中是否引用该币种（字符串和整数两种格式检查） |
| **bills** | [L113-L118](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Currency/CurrencyRepository.php#L113-L118) | 账单中是否使用该币种 |
| **recurring** | [L121-L128](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Currency/CurrencyRepository.php#L121-L128) | 定期交易中是否使用该币种（本币或外币） |
| **account_meta** (二次检查) | [L131-L141](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Currency/CurrencyRepository.php#L131-L141) | 带账户删除状态检查的账户币种引用 |
| **available_budgets** | [L144-L149](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Currency/CurrencyRepository.php#L144-L149) | 可用预算中是否使用该币种 |
| **budget_limits** | [L152-L157](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Currency/CurrencyRepository.php#L152-L157) | 预算限额中是否使用该币种 |
| **current_default** | [L160-L173](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Currency/CurrencyRepository.php#L160-L173) | 是否是用户组的默认币种（两次相同检查） |

### 1.5 CurrencyRepository::makePrimary 流程

**代码位置**: [CurrencyRepository::makePrimary](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Currency/CurrencyRepository.php#L369-L383)

执行步骤：
1. 获取当前主币种 `$current = Amount::getPrimaryCurrencyByUserGroup($this->userGroup)`
2. 分离当前币种（确保干净状态）
3. 遍历所有关联币种，重置 `group_default` 为 `false`
4. 将目标币种关联到用户组，设置 `group_default` 为 `true`
5. **关键**: 如果新旧主币种 ID 不同，触发 `UserGroupChangedPrimaryCurrency` 事件

```php
if ($current->id !== $currency->id) {
    event(new UserGroupChangedPrimaryCurrency($this->userGroup));
}
```

**事件类**: [UserGroupChangedPrimaryCurrency](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Events/Preferences/UserGroupChangedPrimaryCurrency.php)

---

## 第二部分：主币种金额重算机制

### 2.1 RecalculatesPrimaryCurrencyAmounts 监听器

**代码位置**: [RecalculatesPrimaryCurrencyAmounts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Listeners/System/RecalculatesPrimaryCurrencyAmounts.php)

**触发条件**: 监听 `UserGroupChangedPrimaryCurrency` 事件

**控制逻辑**:
```php
public function handle(UserGroupChangedPrimaryCurrency $event): void
{
    if (Amount::convertToPrimary()) {
        $calculator = new PrimaryAmountRecalculationService();
        $calculator->recalculate();
    }
}
```

**关键点**: 重算操作受 `Amount::convertToPrimary()` 控制，该方法需要同时满足：
1. 用户偏好 `convert_to_primary` 为 `true`
2. 系统配置 `enable_exchange_rates` 为 `true`

### 2.2 Amount::convertToPrimary 控制逻辑

**代码位置**: [Amount::convertToPrimary](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Amount.php#L115-L143)

返回 `true` 的条件（AND 关系）：
- `Preferences::get('convert_to_primary', false)->data === true`
- `FireflyConfig::get('enable_exchange_rates', config('cer.enabled'))->data === true`

### 2.3 PrimaryAmountRecalculationService::recalculate 流程

**代码位置**: [PrimaryAmountRecalculationService::recalculate](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Services/Internal/Recalculate/PrimaryAmountRecalculationService.php#L63-L82)

**前置检查**:
```php
if (false === FireflyConfig::get('enable_exchange_rates', config('cer.enabled'))->data) {
    return;
}
```

**执行流程**:
1. **重置阶段（Reset）**
   - `resetGenericTables()`: 重置 accounts、available_budgets、bills 表的原生金额字段
   - `resetPiggyBanks()`: 重置储蓄罐的目标金额和事件金额
   - `resetBudgets()`: 重置预算限额和自动预算的原生金额
   - `resetTransactions()`: 重置交易的原生金额字段

2. **重算阶段（Recalculate）**
   - `recalculateAccounts()`: 重算账户虚拟余额的原生金额
   - `recalculatePiggyBanks()`: 重算储蓄罐的原生金额（包括关联账户和事件）
   - `recalculateBudgets()`: 重算预算限额和自动预算
   - `recalculateAvailableBudgets()`: 重算可用预算
   - `recalculateBills()`: 重算账单
   - `calculateTransactions()`: 通过 touch() 触发交易观察者重算

### 2.4 重算对象清单

| 对象类型 | 重置字段 | 重算方式 |
|---------|---------|---------|
| **Accounts** | `native_virtual_balance` | `touch()` 触发观察者 |
| **Piggy Banks** | `native_target_amount`, `native_current_amount` (pivot), `native_amount` (events) | 直接计算 + `touch()` events |
| **Budgets** | `native_amount` (BudgetLimit), `native_amount` (AutoBudget) | `touch()` 触发观察者 |
| **Available Budgets** | `native_amount` | `touch()` 触发观察者 |
| **Bills** | `native_amount_min`, `native_amount_max` | `touch()` 触发观察者 |
| **Transactions** | `native_amount`, `native_foreign_amount` | `touch()` 触发 TransactionObserver |

---

## 第三部分：外部汇率下载任务

### 3.1 调用链路

```
Cron::handle() 
    └──> exchangeRatesCronJob()
        └──> ExchangeRatesCronjob::fire()
            └──> fireExchangeRateJob()
                └──> DownloadExchangeRates::handle()
                    └──> downloadRates()
                        └──> saveRates()
                            └──> saveRate()
```

### 3.2 Cron::handle 入口控制

**代码位置**: [Cron::handle](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Console/Commands/Tools/Cron.php#L58-L144)

**启用条件**:
```php
if (true === FireflyConfig::get('enable_external_rates', config('cer.download_enabled'))->data 
    && ($doAll || $this->option('download-cer'))) {
    $this->exchangeRatesCronJob($force, $date);
}
```

**关键配置**:
- `enable_external_rates`: 动态配置项
- `config('cer.download_enabled')`: 默认值来自 `ENABLE_EXTERNAL_RATES` 环境变量

### 3.3 ExchangeRatesCronjob::fire 频率控制

**代码位置**: [ExchangeRatesCronjob::fire](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Cronjobs/ExchangeRatesCronjob.php#L39-L68)

**时间控制逻辑**:
1. 读取上次执行时间 `last_cer_job`
2. 计算时间差 `$diff = now() - $lastTime`
3. 如果 `$diff <= 43200` 秒（12小时）且未强制执行，**跳过任务**
4. 超过 12 小时或 `force === true` 时，执行下载

**关键常量**:
- `43200` 秒 = 12 小时 - 最小执行间隔

**配置项**:
- `last_cer_job`: 存储上次执行的 Unix 时间戳

### 3.4 DownloadExchangeRates::handle 执行流程

**代码位置**: [DownloadExchangeRates::handle](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/DownloadExchangeRates.php#L85-L94)

#### 3.4.1 获取完整币种集合

```php
$currencies = $this->repository->getCompleteSet();
```
**代码位置**: [CurrencyRepository::getCompleteSet](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Currency/CurrencyRepository.php#L329-L332)

返回所有 `enabled = true` 的币种，按 code 排序。

#### 3.4.2 下载汇率（downloadRates）

**代码位置**: [DownloadExchangeRates::downloadRates](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/DownloadExchangeRates.php#L107-L139)

**URL 构建**:
```php
$base = sprintf('%s/%s/%s', (string) config('cer.url'), $this->date->year, $this->date->isoWeek);
$url = sprintf('%s/%s.json', $base, $currency->code);
```

**配置来源**: [cer.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/cer.php#L26)
- 默认 URL: `https://ff3exchangerates.z6.web.core.windows.net`

**HTTP 异常处理**:
- `ConnectException`: 连接失败 → 记录 warning，返回
- `RequestException`: 请求异常 → 记录 warning，返回
- 状态码 ≠ 200 → 记录 warning，返回
- JSON 解析失败 → 记录 warning，返回

#### 3.4.3 保存汇率（saveRates + saveRate）

**代码位置**: 
- [saveRates](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/DownloadExchangeRates.php#L181-L193)
- [saveRate](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Jobs/DownloadExchangeRates.php#L169-L179)

**过滤逻辑（saveRates）**:
1. 遍历返回的汇率数据
2. 调用 `getCurrency($code)` 验证目标币种
3. 目标币种必须存在且 `enabled = true` 才保存
4. 未启用的币种跳过处理

**重复检查（saveRate）**:
```php
foreach ($this->users as $user) {
    $this->repository->setUser($user);
    $existing = $this->repository->getExchangeRate($from, $to, $date);
    if (!$existing instanceof CurrencyExchangeRate) {
        $this->repository->setExchangeRate($from, $to, $date, $rate);
    }
}
```

**关键行为**:
- **按用户保存**: 每个用户独立保存汇率记录
- **不覆盖已有数据**: 如果该日期的汇率已存在，不进行更新（幂等性保证）

---

## 测试断言设计

### 4.1 409 状态码测试

**测试场景 1: 禁用最后一个启用币种**
```php
// 给定：系统中只有一个启用的币种 EUR
// 当：尝试 PUT /currencies/EUR 并设置 enabled=false
// 则：响应状态码应为 409
// 且：币种 EUR 仍然保持启用状态
```

**测试场景 2: 禁用正在使用的币种**
```php
// 给定：币种 USD 已在交易中使用（journals）
// 当：尝试 PUT /currencies/USD 并设置 enabled=false
// 则：响应状态码应为 409
// 且：币种 USD 仍然保持启用状态
```

**测试场景 3: currencyInUseAt 各检查项**
```php
// 对于每个使用场景（account_meta, bills, recurring, available_budgets, budget_limits, current_default）
// 给定：币种在该场景被引用
// 当：尝试禁用该币种
// 则：响应状态码应为 409
```

### 4.2 事件触发测试

**测试场景: 主币种变化触发事件**
```php
// 给定：当前主币种为 EUR，有另一个启用的币种 USD
// 当：POST /currencies/USD/primary
// 则：应触发 UserGroupChangedPrimaryCurrency 事件
// 且：事件应携带正确的 userGroup
// 且：USD 应成为新的主币种
```

**测试场景: 相同主币种不触发事件**
```php
// 给定：当前主币种为 EUR
// 当：POST /currencies/EUR/primary（重复设置）
// 则：不应触发 UserGroupChangedPrimaryCurrency 事件
// 且：主币种仍然是 EUR
```

### 4.3 汇率 HTTP 失败测试

**测试场景: 网络连接失败**
```php
// 给定：cer.url 指向不可达地址
// 当：执行 DownloadExchangeRates 任务
// 则：应捕获 ConnectException
// 且：不应抛出异常中断程序
// 且：应记录 warning 日志
// 且：不应保存任何汇率
```

**测试场景: 非 200 响应**
```php
// 给定：cer.url 返回 404/500 状态码
// 当：执行 DownloadExchangeRates 任务
// 则：应记录 warning 日志
// 且：不应保存该币种的任何汇率
```

**测试场景: 无效 JSON 响应**
```php
// 给定：cer.url 返回无效 JSON
// 当：执行 DownloadExchangeRates 任务
// 则：应记录 warning 日志
// 且：不应保存该币种的任何汇率
```

### 4.4 重复汇率不覆盖测试

**测试场景: 已有汇率不被覆盖**
```php
// 给定：2024-01-01 USD->EUR 汇率已存在（rate=0.92）
// 当：下载同日期同币种对，新汇率为 0.93
// 则：数据库中的汇率应保持为 0.92
// 且：不应创建新的汇率记录
// 且：不应更新现有记录
```

**测试场景: 按用户独立保存**
```php
// 给定：用户 A 已有 USD->EUR 汇率
// 当：用户 B 执行下载
// 则：用户 B 应获得独立的汇率记录
// 且：用户 A 的汇率不受影响
```

**测试场景: 只保存启用的目标币种**
```php
// 给定：币种 GBP 存在但 disabled=false
// 当：下载 USD 的汇率，响应中包含 GBP
// 则：不应保存 USD->GBP 的汇率记录
// 且：只保存目标币种为 enabled 的汇率
```

---

## 附录：关键配置汇总

| 配置项 | 位置 | 说明 |
|-------|------|------|
| `cer.url` | [config/cer.php#L26](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/cer.php#L26) | 汇率服务地址 |
| `cer.enabled` | [config/cer.php#L27](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/cer.php#L27) | 汇率功能启用（默认） |
| `cer.download_enabled` | [config/cer.php#L28](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/cer.php#L28) | 外部下载启用（默认） |
| `enable_exchange_rates` | FireflyConfig | 动态汇率功能开关 |
| `enable_external_rates` | FireflyConfig | 动态外部下载开关 |
| `last_cer_job` | FireflyConfig | 上次执行时间戳 |
| `convert_to_primary` | Preferences | 用户转换偏好 |
