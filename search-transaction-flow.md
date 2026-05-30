# Firefly III 交易搜索全链路追踪

> 以查询 `coffee tag_contains:food -has_attachments amount_more:100 account_id:1,2 date:2026-05 before:2026-06-01` 为例，从 `GET v1/search/transactions` 一直追踪到最终的交易组分页 JSON 响应。

---

## 1. 入口：API 路由与请求对象

### 1.1 TransactionSearchRequest 如何组合 PaginationRequest 与 SearchQueryRequest

路由 `GET v1/search/transactions` 指向 [TransactionController::search()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Search/TransactionController.php#L94-L123)，其第一个参数类型为 `TransactionSearchRequest`。

[TransactionSearchRequest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Search/TransactionSearchRequest.php#L1-L43) 继承自 [AggregateFormRequest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/AggregateFormRequest.php#L32-L106)，核心逻辑在 `getRequests()` 方法：

```php
protected function getRequests(): array
{
    return [
        [PaginationRequest::class, 'sort_class' => TransactionJournal::class],
        SearchQueryRequest::class,
    ];
}
```

`AggregateFormRequest::initialize()` 在请求初始化时遍历 `getRequests()` 返回的数组，将每个子请求实例化并**共享同一个 `attributes` bag**：

```php
foreach ($this->getRequests() as $config) {
    $requestClass         = is_array($config) ? array_shift($config) : $config;
    $instance             = new $requestClass();
    $instance->attributes = $this->attributes;  // 共享 attributes
    if ($instance instanceof ApiRequest) {
        $instance->handleConfig(is_array($config) ? $config : []);
    }
}
```

这意味着：

- **[PaginationRequest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/PaginationRequest.php#L33-L82)** 验证 `page`、`limit`、`sort` 参数，并将它们写入 `$request->attributes`（如 `page=1`, `limit=50`）。它还通过 `handleConfig()` 接收 `sort_class` 配置用于排序验证。
- **[SearchQueryRequest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Search/SearchQueryRequest.php#L30-L47)** 验证 `query` 参数（min:0, max:500），在 `withValidator` 后将 query 字符串写入 `$request->attributes->set('query', $query)`。

两者写入的是同一个 `attributes` bag，所以在控制器中可以通过同一个 `$request` 取出所有值：

```php
$fullQuery = (string) $request->attributes->get('query');   // 来自 SearchQueryRequest
$page      = $request->attributes->get('page');              // 来自 PaginationRequest
$pageSize  = $request->attributes->get('limit');             // 来自 PaginationRequest
```

### 1.2 控制器主流程

[TransactionController::search()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Search/TransactionController.php#L94-L123) 的执行步骤：

```
1. $searcher->parseQuery($fullQuery)     → 解析搜索字符串
2. $searcher->setPage($page)             → 设置分页页码
3. $searcher->setLimit($pageSize)        → 设置每页数量
4. $groups = $searcher->searchTransactions() → 执行搜索，返回 LengthAwarePaginator
5. TransactionGroupEnrichment->enrich()  → 丰富交易数据
6. TransactionGroupTransformer->transform() → 转换为 API 响应格式
7. Fractal Manager 输出 JSON
```

---

## 2. SearchServiceProvider：绑定 QueryParserInterface 与 SearchInterface

[SearchServiceProvider](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Providers/SearchServiceProvider.php#L1-L62) 在 `register()` 中做了两个关键绑定：

```php
// 绑定 1: QueryParserInterface → QueryParser
$this->app->bind(static fn (): QueryParserInterface => app(QueryParser::class));

// 绑定 2: SearchInterface → OperatorQuerySearch（带用户注入）
$this->app->bind(static function (Application $app): SearchInterface {
    $search = app(OperatorQuerySearch::class);
    if ($app->auth->check()) {
        $search->setUser(auth()->user());
    }
    return $search;
});
```

- **QueryParserInterface** → [QueryParser](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/QueryParser/QueryParser.php#L38-L212)：负责将搜索字符串解析为 AST（`NodeGroup`）。
- **SearchInterface** → [OperatorQuerySearch](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php#L62-L2981)：负责遍历 AST、解析操作符、构建查询条件、执行搜索。

控制器通过方法注入 `SearchInterface $searcher` 获取到的就是 `OperatorQuerySearch` 实例，且已通过 `setUser()` 初始化了用户上下文和 `GroupCollector`。

---

## 3. OperatorQuerySearch::parseQuery 与节点处理

### 3.1 整体流程

[parseQuery()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php#L175-L200) 方法：

```php
public function parseQuery(string $query): void
{
    $parser = app(QueryParserInterface::class);  // 实际是 QueryParser
    $parsedQuery = $parser->parse($query);       // 返回 NodeGroup

    $this->handleSearchNode($parsedQuery, $parsedQuery->isProhibited(false));

    $this->collector->withBillInformation();
    $this->collector->setSearchWords($this->words);
    $this->collector->excludeSearchWords($this->prohibitedWords);
}
```

### 3.2 QueryParser 解析示例查询

对于 `coffee tag_contains:food -has_attachments amount_more:100 account_id:1,2 date:2026-05 before:2026-06-01`，[QueryParser](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/QueryParser/QueryParser.php#L41-L212) 的单趟扫描产出一个顶层 `NodeGroup`：

```
NodeGroup(prohibited=false, nodes=[
    StringNode(value="coffee", prohibited=false),
    FieldNode(operator="tag_contains", value="food", prohibited=false),
    FieldNode(operator="has_attachments", value="true", prohibited=true),   // -has_attachments
    FieldNode(operator="amount_more", value="100", prohibited=false),
    FieldNode(operator="account_id", value="1,2", prohibited=false),
    FieldNode(operator="date", value="2026-05", prohibited=false),         // alias for date_on
    FieldNode(operator="before", value="2026-06-01", prohibited=false),    // alias for date_before
])
```

### 3.3 三种节点类型的处理

[handleSearchNode()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php#L326-L351) 通过 `switch(true)` 分派到三种处理方法：

#### StringNode — [handleStringNode()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php#L353-L367)

```php
$prohibited = $node->isProhibited($flipProhibitedFlag);
if ($prohibited) {
    $this->prohibitedWords[] = $string;   // 排除词
} else {
    $this->words[] = $string;             // 搜索词
}
```

示例中 `"coffee"` 的 `prohibited=false`，所以 `$this->words = ['coffee']`。

#### FieldNode — [handleFieldNode()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php#L281-L310)

```php
$operator   = strtolower($node->getOperator());
$value      = $node->getValue();
$prohibited = $node->isProhibited($flipProhibitedFlag);
$context    = config(sprintf('search.operators.%s.needs_context', $operator));

// needs_context=false 且 value="false" 时的翻转逻辑
if ('false' === $value && in_array($operator, $this->validOperators, true) && false === $context && !$prohibited) {
    $prohibited = true;
    $value = 'true';
}

if ($inArray && $this->updateCollector($operator, $value, $prohibited)) {
    $this->operators->push(['type' => self::getRootOperator($operator), 'value' => $value, 'prohibited' => $prohibited]);
}
```

关键点：
1. 先从 `config/search.php` 查询操作符的 `needs_context` 属性。
2. 对于 `needs_context=false` 的操作符（如 `has_attachments`），值可以是 `true`/`false`；如果值为 `false`，会翻转 `prohibited` 标记。
3. 调用 `getRootOperator()` 解析 alias，将 `date` → `date_on`、`before` → `date_before`。
4. 调用 `updateCollector()` 执行具体的 collector 操作。

#### NodeGroup — [handleNodeGroup()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php#L312-L319)

```php
$prohibited = $node->isProhibited($flipProhibitedFlag);
foreach ($node->getNodes() as $subNode) {
    $this->handleSearchNode($subNode, $prohibited);
}
```

NodeGroup 的 `prohibited` 标记会作为 `flipProhibitedFlag` 传递给子节点。这使得 `-(amount:100 category:food)` 这样的禁止子查询内部的每个 FieldNode 的 prohibited 状态都被翻转。

### 3.4 prohibited 标记传递链

[Node::isProhibited(bool $flipFlag)](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/QueryParser/Node.php#L82-L91)：

```php
public function isProhibited(bool $flipFlag): bool
{
    if ($flipFlag) {
        return !$this->prohibited;  // 翻转
    }
    return $this->prohibited;
}
```

对于示例中的 `-has_attachments`：
- QueryParser 产出 `FieldNode(operator="has_attachments", value="true", prohibited=true)`
- 在 `handleFieldNode` 中，`has_attachments` 的 `needs_context=false`，值为 `"true"`，不触发翻转逻辑
- `prohibited=true`，所以走 `-has_attachments` 分支 → `collector->hasNoAttachments()`

---

## 4. config/search.php 中的 Alias 与 needs_context

### 4.1 date 和 before 的 Alias

在 [config/search.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/search.php#L160-L167) 中：

| 用户输入操作符 | alias | alias_for | needs_context |
|---|---|---|---|
| `date_on` | false | — | true |
| `date` | **true** | `date_on` | true |
| `date_is` | **true** | `date_on` | true |
| `on` | **true** | `date_on` | true |
| `date_before` | false | — | true |
| `before` | **true** | `date_before` | true |

当用户输入 `date:2026-05` 时，`handleFieldNode` 中 `$operator = "date"`，然后 `getRootOperator("date")` 查到 `date` 是 alias，返回 `date_on`。同理 `before:2026-06-01` 被解析为 `date_before`。

在 `updateCollector` 中：
```php
case 'date_on':
    $range = $this->parseDateRange($operator, $value);
    $this->setExactDateParams($range, $prohibited);
    return false;  // 注意：return false，不 push 到 operators

case 'date_before':
    $range = $this->parseDateRange($operator, $value);
    $this->setDateBeforeParams($range);
    return false;
```

`parseDateRange()` 使用 `ParseDateString` 解析值。`2026-05` 会被解析为一个月份范围（exact 日期或 year+month 组合），`2026-06-01` 会被解析为精确日期。

### 4.2 has_attachments 的 needs_context=false

```php
'has_attachments' => ['alias' => false, 'needs_context' => false],
```

`needs_context=false` 意味着该操作符不需要用户输入值（或用户输入的值只是 `true`/`false`），它本身就是一个布尔开关。

在 `handleFieldNode` 中，特殊逻辑处理 `needs_context=false` 的操作符：

```php
// 如果值为 "false" 且操作符合法且 needs_context=false 且未禁止 → 翻转为禁止
if ('false' === $value && in_array($operator, $this->validOperators, true) && false === $context && !$prohibited) {
    $prohibited = true;
    $value = 'true';
}
// 如果已禁止但值为 "false" → 反翻转
if ('false' === $value && $prohibited && in_array($operator, $this->validOperators, true) && false === $context) {
    $prohibited = false;
    $value = 'true';
}
```

因此：
- `has_attachments:true` → `prohibited=false` → 走 `case 'has_attachments'` → `collector->hasAttachments()`
- `has_attachments:false` → 翻转为 `prohibited=true, value=true` → 走 `case '-has_attachments'` → `collector->hasNoAttachments()`
- `-has_attachments` → `prohibited=true` → 走 `case '-has_attachments'` → `collector->hasNoAttachments()`
- `-has_attachments:false` → 反翻转为 `prohibited=false, value=true` → 走 `case 'has_attachments'` → `collector->hasAttachments()`

---

## 5. updateCollector 中各操作符的不同分支

[updateCollector()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php) 是一个超大的 switch-case 方法，根据操作符名调用 GroupCollector 的不同方法。以下针对示例查询中的操作符逐一分析。

### 5.1 amount_more:100

```php
case '-amount_less':
case 'amount_more':
    $value  = str_replace(',', '.', $value);
    $amount = Steam::positive($value);
    $this->collector->amountMore($amount);
    break;
```

- 将逗号替换为点号以标准化数字格式
- 使用 `Steam::positive()` 确保金额为正值
- 直接调用 `collector->amountMore("100")`，在 SQL 层面加 `WHERE amount > 100`

### 5.2 account_id:1,2

```php
case 'account_id':
    $parts      = explode(',', $value);       // ["1", "2"]
    $collection = new Collection();
    foreach ($parts as $accountId) {
        $account = $this->accountRepository->find((int) $accountId);
        if (null !== $account) {
            $collection->push($account);
        }
    }
    if ($collection->count() > 0) {
        $this->collector->setAccounts($collection);  // OR 语义：源或目标账户
    }
    if (0 === $collection->count()) {
        $this->collector->findNothing();  // 找不到任何账户 → 不返回结果
    }
    break;
```

- 支持逗号分隔的多个 ID
- 通过 `accountRepository->find()` 逐个查找，找到的放入 Collection
- `setAccounts()` 表示"源或目标账户在列表中"（OR 语义），即交易涉及账户 1 **或** 账户 2
- 如果一个账户都找不到，调用 `findNothing()` 使查询返回空

### 5.3 tag_contains:food

详见第 6 节专门分析。

### 5.4 date:2026-05（alias → date_on）

```php
case 'date_on':
    $range = $this->parseDateRange($operator, $value);
    $this->setExactDateParams($range, $prohibited);
    return false;  // 返回 false，operator 由 setExactDateParams 内部 push
```

[setExactDateParams()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php#L780-L865) 根据 `parseDateRange()` 的解析结果分派：

| 解析结果 key | 含义 | collector 调用 |
|---|---|---|
| `exact` | 精确日期 | `collector->setRange($start, $end)` |
| `year` | 仅年份 | `collector->yearIs($value)` |
| `month` | 年+月 | `collector->monthIs($value)` |
| `day` | 仅日 | `collector->dayIs($value)` |

`2026-05` 会被 `ParseDateString` 解析为 `month` 类型（`year=2026, month=05`），调用 `collector->monthIs("05")` + `collector->yearIs("2026")` 设置日期范围。

### 5.5 before:2026-06-01（alias → date_before）

```php
case 'date_before':
    $range = $this->parseDateRange($operator, $value);
    $this->setDateBeforeParams($range);
    return false;
```

[setDateBeforeParams()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php#L724-L773) 同样按解析结果分派：

| 解析结果 key | 含义 | collector 调用 |
|---|---|---|
| `exact` | 精确日期 | `collector->setBefore($carbon)` |
| `year` | 年份之前 | `collector->yearBefore($value)` |
| `month` | 月份之前 | `collector->monthBefore($value)` |
| `day` | 日之前 | `collector->dayBefore($value)` |

`2026-06-01` 会被解析为 `exact` 类型，调用 `collector->setBefore(Carbon::parse("2026-06-01"))`，在 SQL 中加 `WHERE date < "2026-06-01"`。

### 5.6 -has_attachments

```php
case 'has_no_attachments':
case '-has_attachments':
    $this->collector->hasNoAttachments();
    break;
```

直接调用 collector 的过滤方法，排除有附件的交易。

---

## 6. tag_contains 的三种命中场景

[tag_contains](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php#L2175-L2197) 的处理比其他操作符更复杂，因为它需要搜索 tag 名称来匹配，且匹配数量不同时走不同逻辑：

```php
case 'tag_contains':
    $tags = $this->tagRepository->searchTag($value);
    if (0 === $tags->count()) {
        // 场景 A：0 个命中
        $this->collector->findNothing();
    }
    if (1 === $tags->count()) {
        // 场景 B：1 个命中
        $ids   = array_values($tags->pluck('id')->toArray());
        $index = count($this->includeAllTags);
        $this->includeAllTags[$index] = array_unique(array_merge($this->includeAllTags[$index] ?? [], $ids));
    }
    if ($tags->count() > 1) {
        // 场景 C：多个命中
        $ids   = array_values($tags->pluck('id')->toArray());
        $index = count($this->includeAnyTags);
        $this->includeAnyTags[$index] = array_unique(array_merge($this->includeAnyTags[$index] ?? [], $ids));
    }
    break;
```

### 场景 A：0 个 tag 命中（findNothing）

当 `searchTag("food")` 返回空集合时，说明数据库中不存在任何名称包含 "food" 的 tag。此时没有任何交易可能满足条件，因此调用 `collector->findNothing()`：

```php
public function findNothing(): GroupCollectorInterface
{
    $this->query->where('transaction_groups.id', -1);  // 永远不可能满足的条件
    return $this;
}
```

这保证 SQL 查询返回零结果。

### 场景 B：1 个 tag 命中（includeAllTags）

当恰好匹配到 1 个 tag 时，该 tag 的 ID 被加入 `includeAllTags` 数组。之后在 `parseTagInstructions()` 中：

```php
if (count($this->includeAllTags) > 0) {
    foreach ($this->includeAllTags as $set) {
        $collection = new Collection();
        foreach ($set as $tagId) {
            $tag = $this->tagRepository->find($tagId);
            if (null !== $tag) {
                $collection->push($tag);
            }
            $this->collector->setAllTags($collection);
        }
    }
}
```

[setAllTags()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Helpers/Collector/Extensions/MetaCollection.php#L465-L517) 添加一个 **postFilter**，要求交易的 tag 列表必须包含所有指定 tag：

```php
$filter = static function (array $object) use ($list): bool|array {
    $expectedTagCount = count($list);
    $foundTagCount    = 0;
    foreach ($object['transactions'] as $transaction) {
        foreach ($transaction['tags'] as $tag) {
            if (in_array(strtolower((string) $tag['name']), $list, true)) {
                ++$foundTagCount;
            }
        }
    }
    return $foundTagCount >= $expectedTagCount;  // 必须找到所有期望的 tag
};
```

当只有 1 个 tag 时，`includeAllTags` 和 `includeAnyTags` 语义等价——都是"交易必须包含这个 tag"。

### 场景 C：多个 tag 命中（includeAnyTags）

当匹配到多个 tag 时（如搜索 "food" 匹配到了 "food"、"seafood"、"foodie"），这些 tag 的 ID 被加入 `includeAnyTags`：

```php
if (count($this->includeAnyTags) > 0) {
    foreach ($this->includeAnyTags as $set) {
        $collection = new Collection();
        foreach ($set as $tagId) {
            $tag = $this->tagRepository->find($tagId);
            if (null !== $tag) {
                $collection->push($tag);
            }
        }
        $this->collector->setTags($collection);
    }
}
```

[setTags()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Helpers/Collector/Extensions/MetaCollection.php#L656-L688) 添加的 postFilter 是 **OR 语义**——只要交易包含列表中的任意一个 tag 即可：

```php
$filter = static function (array $object) use ($list): bool {
    foreach ($object['transactions'] as $transaction) {
        foreach ($transaction['tags'] as $tag) {
            if (in_array(strtolower((string) $tag['name']), $list, true)) {
                return true;   // 找到任意一个匹配即可
            }
        }
    }
    return false;
};
```

### 三种场景总结

| 命中数 | 存储位置 | 后续 collector 方法 | 语义 | 为什么这样设计 |
|---|---|---|---|---|
| 0 | — | `findNothing()` | 不可能匹配 | 数据库不存在此 tag，查询必然为空 |
| 1 | `includeAllTags` | `setAllTags()` | 交易必须包含此 tag | 单 tag 时 ALL 和 ANY 等价，选 ALL 语义更严格、更高效 |
| >1 | `includeAnyTags` | `setTags()` | 交易包含任一匹配 tag | 多 tag 时 ALL 语义（必须同时有所有 tag）过严，改用 ANY 语义更符合"包含"直觉 |

> 注：`tag_contains` 从 `includeTags` 改为 `includeAnyTags` 是为修复 #8632，多个 tag 命中时应为 ANY 而非 ALL。`includeAllTags` 的引入是为修复 #11473，确保多个独立 tag 搜索条件（如 `tag_is:A tag_is:B`）正确作为 ALL 处理。

---

## 7. searchTransactions 为空条件时返回空 Paginator

[searchTransactions()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php#L207-L225)：

```php
public function searchTransactions(): LengthAwarePaginator
{
    $this->parseTagInstructions();
    if (
        0 === count($this->excludeTags)
        && 0 === count($this->includeAnyTags)
        && 0 === count($this->includeAllTags)
        && 0 === count($this->getWords())
        && 0 === count($this->getExcludedWords())
        && 0 === count($this->getOperators())
    ) {
        Log::warning('No need to search for anything');
        return new LengthAwarePaginator([], 0, 5, 1);
    }
    return $this->collector->getPaginatedGroups();
}
```

**为什么返回空 Paginator 而不是空数组或 null？**

1. **类型一致性**：控制器代码期望 `searchTransactions()` 返回 `LengthAwarePaginator`，后续 `$groups->setPath($url)`、`$groups->getCollection()` 等方法调用要求返回值必须是 paginator 实例。返回空数组会导致方法调用失败。

2. **Fractal 适配器兼容**：控制器使用 `IlluminatePaginatorAdapter` 包装 paginator：`$resource->setPaginator(new IlluminatePaginatorAdapter($groups))`。Fractal 需要一个合法的 paginator 对象来生成分页元数据（`meta.pagination`）。

3. **空结果也要有分页信息**：即使是空结果，API 响应也需要包含 `pagination` 结构（total=0, count=0, per_page=5, current_page=1, total_pages=1），这样客户端代码可以统一处理。

4. **避免全表扫描**：如果查询字符串完全为空，不执行搜索可以避免返回用户的所有交易数据（安全考虑），也避免了不必要的数据库查询。

`new LengthAwarePaginator([], 0, 5, 1)` 表示：空数据集、0 条总记录、每页 5 条、第 1 页。

---

## 8. setPage/setLimit 与 with*Information 方法

### 8.1 setPage 与 setLimit

在 [OperatorQuerySearch](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php#L232-L242) 中：

```php
public function setLimit(int $limit): void
{
    $this->limit = $limit;
    $this->collector->setLimit($this->limit);
}

public function setPage(int $page): void
{
    $this->page = $page;
    $this->collector->setPage($this->page);
}
```

直接传递给 [GroupCollector](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Helpers/Collector/GroupCollector.php#L553-L582)。`setLimit` 和 `setPage` 控制 `getPaginatedGroups()` 的分页行为：

```php
public function getPaginatedGroups(): LengthAwarePaginator
{
    $limit = $this->limit ?? 1;
    if (0 === $this->limit) {
        $this->setLimit(50);
    }
    $set = $this->getGroups();
    return new LengthAwarePaginator($set, $this->total, $limit, $this->page);
}
```

### 8.2 withAccountInformation / withCategoryInformation / withBudgetInformation

这三个方法在 `setUser()` 中自动调用：

```php
public function setUser(User $user): void
{
    // ...
    $this->collector = app(GroupCollectorInterface::class);
    $this->collector->setUser($user);
    $this->collector->withAccountInformation()->withCategoryInformation()->withBudgetInformation();
}
```

它们的作用是**在 SQL 查询中 LEFT JOIN 对应的表并 SELECT 额外字段**，使搜索结果中包含关联数据：

| 方法 | JOIN 行为 | 新增 SELECT 字段 |
|---|---|---|
| [withAccountInformation()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Helpers/Collector/Extensions/AccountCollection.php#L284-L313) | LEFT JOIN `accounts` (source/dest) + `account_types` | `source_account_name`, `source_account_iban`, `source_account_type`, `destination_account_name`, `destination_account_iban`, `destination_account_type` |
| [withCategoryInformation()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Helpers/Collector/Extensions/MetaCollection.php#L799-L815) | LEFT JOIN `category_transaction_journal` + `categories` | `category_id`, `category_name` |
| [withBudgetInformation()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Helpers/Collector/Extensions/MetaCollection.php#L764-L785) | LEFT JOIN `budget_transaction_journal` + `budgets` | `budget_id`, `budget_name` |

此外，`parseQuery()` 末尾还会追加 `withBillInformation()`：

```php
$this->collector->withBillInformation();
```

这 LEFT JOIN `bills` 表，新增 `bill_id` 和 `bill_name` 字段。

---

## 9. TransactionGroupEnrichment 与 TransactionGroupTransformer

### 9.1 TransactionGroupEnrichment

[TransactionGroupEnrichment](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/JsonApi/Enrichments/TransactionGroupEnrichment.php#L1-L267) 在 collector 查询结果之上**批量补充额外数据**，避免 N+1 查询：

```php
public function enrich(Collection $collection): Collection
{
    $this->collection = $collection;
    $this->collectJournalIds();    // 收集所有 journal ID
    $this->collectNotes();         // 批量查 notes
    $this->collectTags();          // 批量查 tags
    $this->collectMetaData();      // 批量查 meta
    $this->collectLocations();     // 批量查 locations
    $this->collectAttachmentCount();// 批量查附件计数
    $this->appendCollectedData();  // 将收集的数据挂载到每条交易上
    return $this->collection;
}
```

`appendCollectedData()` 为每个 transaction 补充以下字段：

| 字段 | 来源 | 说明 |
|---|---|---|
| `notes` | `notes` 表 | 交易备注文本 |
| `tags` | `tags` + `tag_transaction_journal` | 交易的标签列表（字符串数组） |
| `attachment_count` | `attachments` 表 COUNT | 附件数量 |
| `location` | `locations` 表 | 经纬度和缩放级别 |
| `primary_currency` | 系统主货币 | 用户的主要货币信息 |
| `meta` / `meta_date` | `journal_meta` 表 | SEPA、内部引用、外部 ID 等元数据及日期元数据 |

### 9.2 TransactionGroupTransformer

[TransactionGroupTransformer](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Transformers/TransactionGroupTransformer.php#L1-L523) 将 enrich 后的数据转换为 Fractal API 格式。

[transform()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Transformers/TransactionGroupTransformer.php#L81-L96) 方法输出顶层结构：

```php
return [
    'id'           => (int) $first['transaction_group_id'],
    'created_at'   => $first['created_at']->toAtomString(),
    'updated_at'   => $first['updated_at']->toAtomString(),
    'user'         => (string) $data['user_id'],
    'user_group'   => (string) $data['user_group_id'],
    'group_title'  => $data['title'],
    'transactions' => $this->transformTransactions($data),
    'links'        => [['rel' => 'self', 'uri' => '/transactions/'.$first['transaction_group_id']]],
];
```

[transformTransaction()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Transformers/TransactionGroupTransformer.php#L380-L513) 为每条交易输出完整字段，包括：

- 基本信息：`type`, `date`, `order`, `description`
- 金额：`amount`, `foreign_amount`, `pc_amount`（主货币金额）
- 源账户：`source_id`, `source_name`, `source_iban`, `source_type`
- 目标账户：`destination_id`, `destination_name`, `destination_iban`, `destination_type`
- 分类/预算/账单：`budget_id`, `budget_name`, `category_id`, `category_name`, `bill_id`, `bill_name`
- 余额变化：`source_balance_after`, `destination_balance_after`
- 附加信息：`reconciled`, `notes`, `tags`（来自 enrichment）
- 元数据：`internal_reference`, `external_id`, `external_url`, `import_hash_v2`, `recurrence_id`
- SEPA 字段：`sepa_cc`, `sepa_ct_op`, `sepa_ct_id`, `sepa_db`, `sepa_country`, `sepa_ep`, `sepa_ci`, `sepa_batch_id`
- 日期元数据：`interest_date`, `book_date`, `process_date`, `due_date`, `payment_date`, `invoice_date`

---

## 10. 完整链路总结

以 `coffee tag_contains:food -has_attachments amount_more:100 account_id:1,2 date:2026-05 before:2026-06-01` 为例，完整执行链路如下：

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. HTTP GET /v1/search/transactions?query=...&page=1&limit=50      │
└────────────────────┬─────────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 2. TransactionSearchRequest (AggregateFormRequest)                   │
│    ├─ PaginationRequest → attributes['page']=1, ['limit']=50        │
│    └─ SearchQueryRequest → attributes['query']="coffee tag_..."     │
└────────────────────┬─────────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 3. TransactionController::search()                                   │
│    $searcher (SearchInterface → OperatorQuerySearch, via SP)         │
│    $searcher->parseQuery($fullQuery)                                 │
└────────────────────┬─────────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 4. OperatorQuerySearch::parseQuery()                                 │
│    ├─ QueryParser::parse() → NodeGroup AST                          │
│    ├─ handleSearchNode() 递归遍历 AST:                               │
│    │   ├─ StringNode("coffee")     → words[] = ["coffee"]           │
│    │   ├─ FieldNode("tag_contains", "food")                         │
│    │   │   └─ updateCollector → searchTag → includeAllTags/AnyTags  │
│    │   ├─ FieldNode("has_attachments", prohibited=true)             │
│    │   │   └─ updateCollector → collector->hasNoAttachments()       │
│    │   ├─ FieldNode("amount_more", "100")                           │
│    │   │   └─ updateCollector → collector->amountMore("100")        │
│    │   ├─ FieldNode("account_id", "1,2")                            │
│    │   │   └─ updateCollector → collector->setAccounts(...)         │
│    │   ├─ FieldNode("date", "2026-05") [alias→date_on]             │
│    │   │   └─ updateCollector → setExactDateParams → monthIs+yearIs │
│    │   └─ FieldNode("before", "2026-06-01") [alias→date_before]    │
│    │       └─ updateCollector → setDateBeforeParams → setBefore()   │
│    ├─ collector->withBillInformation()                              │
│    ├─ collector->setSearchWords(["coffee"])                         │
│    └─ collector->excludeSearchWords([])                             │
└────────────────────┬─────────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 5. $searcher->setPage(1)  → collector->setPage(1)                   │
│    $searcher->setLimit(50) → collector->setLimit(50)                │
└────────────────────┬─────────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 6. $searcher->searchTransactions()                                   │
│    ├─ parseTagInstructions() → collector->setAllTags/setTags(...)   │
│    ├─ 条件非空 → collector->getPaginatedGroups()                    │
│    └─ 返回 LengthAwarePaginator                                      │
└────────────────────┬─────────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 7. TransactionGroupEnrichment::enrich()                              │
│    批量补充: notes, tags, meta, locations, attachment_count,        │
│    primary_currency 到每条交易                                       │
└────────────────────┬─────────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 8. TransactionGroupTransformer::transform()                          │
│    将 enriche d 数据转为 Fractal API 格式                           │
│    包含完整字段: 金额、账户、分类、预算、账单、元数据等             │
└────────────────────┬─────────────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 9. Fractal Manager + IlluminatePaginatorAdapter                      │
│    输出 JSON: { data: [...], meta: { pagination: {...} } }          │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 关键源码索引

| 组件 | 文件路径 |
|---|---|
| TransactionSearchRequest | [TransactionSearchRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Search/TransactionSearchRequest.php) |
| PaginationRequest | [PaginationRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/PaginationRequest.php) |
| SearchQueryRequest | [SearchQueryRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Search/SearchQueryRequest.php) |
| AggregateFormRequest | [AggregateFormRequest.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/AggregateFormRequest.php) |
| SearchServiceProvider | [SearchServiceProvider.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Providers/SearchServiceProvider.php) |
| TransactionController | [TransactionController.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Search/TransactionController.php) |
| OperatorQuerySearch | [OperatorQuerySearch.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/OperatorQuerySearch.php) |
| QueryParser | [QueryParser.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/QueryParser/QueryParser.php) |
| Node (base) | [Node.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/QueryParser/Node.php) |
| StringNode | [StringNode.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/QueryParser/StringNode.php) |
| FieldNode | [FieldNode.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/QueryParser/FieldNode.php) |
| NodeGroup | [NodeGroup.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/Search/QueryParser/NodeGroup.php) |
| config/search.php | [search.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/search.php) |
| GroupCollector | [GroupCollector.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Helpers/Collector/GroupCollector.php) |
| MetaCollection (setAllTags/setTags) | [MetaCollection.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Helpers/Collector/Extensions/MetaCollection.php) |
| AccountCollection (withAccountInformation) | [AccountCollection.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Helpers/Collector/Extensions/AccountCollection.php) |
| TransactionGroupEnrichment | [TransactionGroupEnrichment.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Support/JsonApi/Enrichments/TransactionGroupEnrichment.php) |
| TransactionGroupTransformer | [TransactionGroupTransformer.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Transformers/TransactionGroupTransformer.php) |
