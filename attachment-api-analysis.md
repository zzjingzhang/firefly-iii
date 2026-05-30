# Firefly III 附件 API 两阶段模型深度分析

## 一、API 总体架构

Firefly III 附件 API 采用**两阶段提交模型**：

| 阶段 | 方法 | 端点 | 作用 |
|------|------|------|------|
| 第一阶段 | `POST` | `/v1/attachments` | 创建附件元数据（metadata），数据库中创建记录 |
| 第二阶段 | `POST` | `/v1/attachments/{attachment}/upload` | 上传原始文件内容（raw body） |
| 下载 | `GET` | `/v1/attachments/{attachment}/download` | 下载附件文件 |

路由定义在 [api.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/routes/api.php#L326-L342)。

---

## 二、请求验证层：StoreRequest::rules

### 2.1 attachable_type 白名单生成

在 [StoreRequest::rules()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Requests/Models/Attachment/StoreRequest.php#L59-L73) 中，验证规则通过以下步骤从配置生成：

```php
public function rules(): array
{
    // 1. 从配置读取完整的类名数组
    $models = config('firefly.valid_attachment_models');
    
    // 2. 去除命名空间前缀，只保留类名
    $models = array_map(static fn (string $className): string => 
        str_replace('FireflyIII\Models\\', '', $className), $models);
    
    // 3. 拼接为逗号分隔的字符串
    $models = implode(',', $models);
    
    return [
        // ...
        'attachable_type' => sprintf('required|in:%s', $models),
        // ...
    ];
}
```

### 2.2 valid_attachment_models 配置

配置在 [firefly.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/firefly.php#L214-L224) 中定义：

```php
'valid_attachment_models' => [
    Account::class,
    Bill::class,
    Budget::class,
    Category::class,
    PiggyBank::class,
    Tag::class,
    Transaction::class,
    TransactionJournal::class,
    Recurrence::class,
],
```

经过 `str_replace` 处理后，实际白名单为：
`Account,Bill,Budget,Category,PiggyBank,Tag,Transaction,TransactionJournal,Recurrence`

### 2.3 IsValidAttachmentModel 保护 attachable_id

[IsValidAttachmentModel](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Rules/IsValidAttachmentModel.php) 是一个自定义验证规则，确保 `attachable_id` 对应的模型：
1. **存在**于数据库中
2. **属于当前用户**（通过各自的 Repository 查询并 `setUser()`）

关键代码流程：

```php
public function validate(string $attribute, mixed $value, Closure $fail): void
{
    // 先通过 normalizeModel 补全命名空间
    $result = match ($this->model) {
        Account::class      => $this->validateAccount((int) $value),
        Bill::class         => $this->validateBill((int) $value),
        // ... 其他类型
        Transaction::class  => $this->validateTransaction((int) $value),
        TransactionJournal::class => $this->validateJournal((int) $value),
        default             => false
    };
    if (false === $result) {
        $fail('validation.model_id_invalid')->translate();
    }
}

// 以 Account 为例，每个验证方法都确保 setUser
private function validateAccount(int $value): bool
{
    $repository = app(AccountRepositoryInterface::class);
    $repository->setUser(auth()->user());
    return null !== $repository->find($value);
}
```

> **设计意图**：两层安全防护
> - 第一层 `in:白名单` 确保类型合法
> - 第二层 `IsValidAttachmentModel` 确保 ID 真实存在且属于当前用户

---

## 三、工厂层：AttachmentFactory::create

### 3.1 Transaction 改挂 TransactionJournal

[AttachmentFactory::create()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Factory/AttachmentFactory.php#L44-L84) 中有一个关键的类型转换逻辑：

```php
public function create(array $data): ?Attachment
{
    $model = str_contains((string) $data['attachable_type'], 'FireflyIII')
        ? $data['attachable_type']
        : sprintf('FireflyIII\Models\%s', $data['attachable_type']);

    // 核心：Transaction → TransactionJournal 转换
    if (Transaction::class === $model) {
        $transaction = $this->user->transactions()->find((int) $data['attachable_id']);
        if (null === $transaction) {
            throw new FireflyException('Unexpectedly could not find transaction');
        }
        // 替换为 TransactionJournal 的 ID 和类名
        $data['attachable_id'] = $transaction->transaction_journal_id;
        $model                 = TransactionJournal::class;
    }
    
    // 创建 Attachment ...
}
```

**设计原因**：
- `Transaction` 是交易的单边分录（debit/credit），而 `TransactionJournal` 才是完整的交易记录
- 附件应该挂载在完整的交易（Journal）上，而非单边分录
- API 允许用户传入 `Transaction` 类型以简化使用，但内部统一存储为 `TransactionJournal`

### 3.2 初始 Attachment 与 Note 创建

创建的 Attachment 初始状态为**占位符记录**：

```php
$attachment = Attachment::create([
    'user_id'         => $this->user->id,
    'attachable_id'   => $data['attachable_id'],
    'attachable_type' => $model,
    'md5'             => '',        // 空
    'filename'        => $data['filename'],
    'title'           => '' === $data['title'] ? null : $data['title'],
    'description'     => null,
    'mime'            => '',        // 空
    'size'            => 0,         // 大小为 0
    'uploaded'        => 0,         // 未上传
]);
```

如果提供了 `notes`，同时创建关联的 Note：

```php
$notes = (string) ($data['notes'] ?? '');
if ('' !== $notes) {
    $note       = new Note();
    $note->noteable()->associate($attachment);
    $note->text = $notes;
    $note->save();
}
```

> **两阶段设计的体现**：第一阶段只创建空壳记录，第二阶段 `upload` 成功后才会填充 `md5`、`mime`、`size`、`uploaded` 字段。

---

## 四、上传流程：StoreController::upload → AttachmentHelper::saveAttachmentFromApi

### 4.1 StoreController::upload

[StoreController::upload()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Attachment/StoreController.php#L97-L121)：

```php
public function upload(Request $request, Attachment $attachment): JsonResponse
{
    // demo 用户检查（见第五节）
    // ...
    
    $body = $request->getContent();
    
    // 空 body 检查
    if ('' === $body) {
        Log::error('Body of attachment is empty.');
        return response()->json([], 422);  // 422 Unprocessable Entity
    }
    
    $result = $helper->saveAttachmentFromApi($attachment, $body);
    if (false === $result) {
        return response()->json([], 422);
    }
    
    return response()->json([], 204);  // 204 No Content
}
```

### 4.2 AttachmentHelper::saveAttachmentFromApi

[AttachmentHelper::saveAttachmentFromApi()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Helpers/Attachments/AttachmentHelper.php#L128-L190)：

```php
public function saveAttachmentFromApi(Attachment $attachment, string $content): bool
{
    // 1. 创建临时文件
    try {
        $resource = tmpfile();
    } catch (FilesystemException $e) {
        return false;
    }
    
    // 2. 再次检查空内容（双重保护）
    if ('' === $content) {
        Log::error('Cannot upload empty file.');
        return false;
    }
    
    // 3. 写入临时文件
    $path = stream_get_meta_data($resource)['uri'];
    try {
        $result = fwrite($resource, $content);
    } catch (FilesystemException $e) {
        return false;
    }
    
    // 4. MIME 类型检测与白名单验证
    try {
        $finfo = finfo_open(FILEINFO_MIME_TYPE);
    } catch (FileinfoException $e) {
        return false;
    }
    $mime = (string) finfo_file($finfo, $path);
    $allowedMime = config('firefly.allowedMimes');
    if (!in_array($mime, $allowedMime, true)) {
        Log::error(sprintf('Mime type %s is not allowed...', $mime));
        fclose($resource);
        return false;
    }
    
    // 5. 存储到 upload 磁盘
    $parts = explode('/', $attachment->fileName());
    $file  = $parts[count($parts) - 1];
    $this->uploadDisk->put($file, $content);
    
    // 6. 更新 Attachment 记录
    $attachment->md5      = md5_file($path);
    $attachment->mime     = $mime;
    $attachment->size     = strlen($content);
    $attachment->uploaded = true;
    $attachment->save();
    
    return true;
}
```

### 4.3 upload 磁盘配置

在 [filesystems.php](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/config/filesystems.php#L62-L65) 中定义：

```php
'upload' => [
    'driver' => 'local',
    'root'   => storage_path('upload'),
],
```

### 4.4 Attachment::fileName()

[Attachment::fileName()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Models/Attachment.php#L97-L100) 定义文件存储命名规则：

```php
public function fileName(): string
{
    return sprintf('at-%s.data', (string) $this->id);
}
```

即附件 ID 为 123 的文件路径为：`storage/upload/at-123.data`

---

## 五、下载流程：AttachmentRepository → ShowController::download

### 5.1 AttachmentRepository::exists

[AttachmentRepository::exists()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Attachment/AttachmentRepository.php#L69-L75)：

```php
public function exists(Attachment $attachment): bool
{
    $disk = Storage::disk('upload');
    return $disk->exists($attachment->fileName());
}
```

### 5.2 AttachmentRepository::getContent

[AttachmentRepository::getContent()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Repositories/Attachment/AttachmentRepository.php#L82-L101)：

```php
public function getContent(Attachment $attachment): string
{
    $disk = Storage::disk('upload');
    $file = $attachment->fileName();
    $unencryptedContent = '';
    
    if ($disk->exists($file)) {
        $encryptedContent = (string) $disk->get($file);
        try {
            $unencryptedContent = Crypt::decrypt($encryptedContent);
        } catch (DecryptException $e) {
            // 解密失败时回退使用原始内容（向后兼容未加密的旧文件）
            $unencryptedContent = $encryptedContent;
        }
    }
    
    return $unencryptedContent;
}
```

### 5.3 ShowController::download 错误处理链

[ShowController::download()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Attachment/ShowController.php#L76-L114) 是最复杂的错误处理节点：

```php
public function download(Attachment $attachment): LaravelResponse
{
    // 检查 1: demo 用户（见第五节）
    if (true === auth()->user()->hasRole('demo')) {
        throw new NotFoundHttpException();
    }
    
    // 检查 2: uploaded 标志
    if (false === $attachment->uploaded) {
        throw new FireflyException('200000: File has not been uploaded (yet).');
    }
    
    // 检查 3: size 为 0
    if (0 === $attachment->size) {
        throw new FireflyException('200000: File has not been uploaded (yet).');
    }
    
    // 检查 4: 文件在磁盘上存在
    if ($this->repository->exists($attachment)) {
        $content = $this->repository->getContent($attachment);
        
        // 检查 5: 内容为空
        if ('' === $content) {
            throw new FireflyException('200002: File is empty (zero bytes).');
        }
        
        // 正常下载响应
        $response = response($content);
        $response
            ->header('Content-Type', 'application/octet-stream')
            ->header('Content-Disposition', 'attachment; filename='.$quoted)
            ->header('Content-Length', (string) strlen($content));
        return $response;
    }
    
    // 检查 4 失败：文件不存在
    throw new FireflyException('200003: File does not exist.');
}
```

### 5.4 各种异常情况汇总表

| 异常场景 | 发生位置 | 错误码/状态码 | 错误信息 |
|----------|----------|--------------|----------|
| **空 body 上传** | [StoreController::upload()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Attachment/StoreController.php#L108-L112) | 422 | 空 JSON `[]` |
| **非法 MIME** | [AttachmentHelper::saveAttachmentFromApi()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Helpers/Attachments/AttachmentHelper.php#L167-L172) | 422 | 空 JSON `[]`（通过 `return false` 传递） |
| **文件不存在** | [ShowController::download()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Attachment/ShowController.php#L113) | FireflyException | `200003: File does not exist.` |
| **uploaded=false** | [ShowController::download()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Attachment/ShowController.php#L83-L85) | FireflyException | `200000: File has not been uploaded (yet).` |
| **size=0** | [ShowController::download()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Attachment/ShowController.php#L86-L88) | FireflyException | `200000: File has not been uploaded (yet).` |
| **空内容** | [ShowController::download()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Attachment/ShowController.php#L91-L93) | FireflyException | `200002: File is empty (zero bytes).` |

---

## 六、Demo 用户保护机制

### 6.1 三层防护

Demo 用户在 store/upload/download 三个端点受到 **NotFoundHttpException** 保护，有三层防护：

#### 第一层：Middleware - ApiDemoUser

在 [StoreController](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Attachment/StoreController.php#L55) 和 [ShowController](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Controllers/Models/Attachment/ShowController.php#L57) 构造函数中：

```php
$this->middleware(ApiDemoUser::class)->except(['delete', 'download', 'show', 'index']);
```

[ApiDemoUser](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Api/V1/Middleware/ApiDemoUser.php#L40-L53) 中间件返回 **403 Forbidden**：

```php
public function handle(Request $request, Closure $next)
{
    $user = $request->user();
    if (null === $user) {
        return $next($request);
    }
    if ($user->hasRole('demo')) {
        return response('', 403);  // 403 禁止访问
    }
    return $next($request);
}
```

> 注意：`download`、`show`、`index` 被 `except` 排除，中间件不对它们生效。

#### 第二层：方法内显式检查 - NotFoundHttpException

在 `store`、`upload`、`download`、`index`、`show` 每个方法内部都有显式检查：

```php
// StoreController::store (line 76-80)
// StoreController::upload (line 99-103)
// ShowController::download (line 78-82)
// ShowController::index (line 126-130)
// ShowController::show (line 160-164)
if (true === auth()->user()->hasRole('demo')) {
    Log::channel('audit')->warning(sprintf('Demo user tries to access attachment API in %s', __METHOD__));
    throw new NotFoundHttpException();  // 404 未找到
}
```

#### 第三层：Route Model Binding - Attachment::routeBinder

[Attachment::routeBinder()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/firefly-iii/app/Models/Attachment.php#L65-L84) 确保即使绕过前面检查，也只能访问自己的附件：

```php
public static function routeBinder(self|string $value): self
{
    if (auth()->check()) {
        $attachmentId = (int) $value;
        $user = auth()->user();
        // 关键：通过 $user->attachments() 查询，确保属于当前用户
        $attachment = $user->attachments()->find($attachmentId);
        if (null !== $attachment) {
            return $attachment;
        }
    }
    throw new NotFoundHttpException();
}
```

### 6.2 保护机制对比

| 位置 | 防护类型 | 响应码 | 覆盖方法 |
|------|----------|--------|----------|
| ApiDemoUser Middleware | 403 | `store`、`upload`、`update`、`destroy` |
| 方法内检查 | 404 | **所有**方法（store/upload/download/index/show） |
| routeBinder | 404 | 所有需要 {attachment} 参数的方法 |

> **设计意图**：
> - 对写入操作（store/upload/update/destroy）双重防护（middleware + 方法内检查）
> - 对读取操作（download/show/index）单独特意抛 404（而不是 403），避免暴露资源存在性
> - routeBinder 是最后一道防线，确保用户越权时看到的也是 404

---

## 七、中间状态测试指南

"metadata 已创建但文件未上传"是两阶段模型的关键中间状态。以下是测试策略：

### 7.1 中间状态特征

| 字段 | 值 |
|------|----|
| `uploaded` | `false` / `0` |
| `md5` | `''`（空字符串） |
| `mime` | `''`（空字符串） |
| `size` | `0` |
| 磁盘文件 | 不存在（`storage/upload/at-{id}.data` 未创建） |

### 7.2 测试方法

#### 步骤 1：创建 metadata（第一阶段）
```bash
curl -X POST "https://firefly.example.com/api/v1/attachments" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "test.pdf",
    "attachable_type": "TransactionJournal",
    "attachable_id": 123
  }'
```

**预期响应**：200 OK，返回包含 `uploaded: false` 的 Attachment JSON。

#### 步骤 2：验证中间状态
```bash
# GET /show 应该正常返回
curl "https://firefly.example.com/api/v1/attachments/{id}" \
  -H "Authorization: Bearer {token}"
```

**预期响应**：200 OK，attachment 对象的 `uploaded: false`、`size: 0`

#### 步骤 3：尝试下载（验证错误码）
```bash
curl -v "https://firefly.example.com/api/v1/attachments/{id}/download" \
  -H "Authorization: Bearer {token}"
```

**预期响应**：抛出 `FireflyException 200000: File has not been uploaded (yet).`

#### 步骤 4：验证文件不存在
```bash
# 在服务器上检查
ls storage/upload/at-{id}.data
# 预期：No such file or directory
```

#### 步骤 5：完成第二阶段上传
```bash
curl -X POST "https://firefly.example.com/api/v1/attachments/{id}/upload" \
  -H "Authorization: Bearer {token}" \
  --data-binary "@test.pdf"
```

**预期响应**：204 No Content

#### 步骤 6：验证上传后状态
```bash
# GET /show 应该返回 uploaded: true，size > 0，md5 和 mime 有值
# GET /download 应该正常返回文件内容
```

### 7.3 可测试的场景

1. **中断场景**：创建 metadata 后不调用 upload，验证系统的一致性
2. **重复上传**：对同一 attachment 调用多次 upload，验证是否覆盖或报错
3. **并发上传**：同时对同一 attachment 发起多个 upload 请求
4. **部分失败**：upload 过程中模拟网络中断，验证数据库状态与磁盘状态一致性
5. **元数据查询**：在 upload 之前，通过 GET /{attachment} 获取 metadata，验证字段正确性
6. **更新 metadata**：在 upload 之前，通过 PUT /{attachment} 更新 filename 等字段

### 7.4 自动化测试建议

```php
// 示例测试用例（伪代码）
public function test_middle_state_after_store()
{
    // 1. 创建 metadata
    $response = $this->postJson('/api/v1/attachments', [
        'filename' => 'test.pdf',
        'attachable_type' => 'TransactionJournal',
        'attachable_id' => $journal->id,
    ]);
    
    $attachmentId = $response->json('data.id');
    
    // 2. 断言数据库状态
    $this->assertDatabaseHas('attachments', [
        'id' => $attachmentId,
        'uploaded' => false,
        'size' => 0,
        'md5' => '',
        'mime' => '',
    ]);
    
    // 3. 断言文件不存在
    Storage::disk('upload')->assertMissing("at-{$attachmentId}.data");
    
    // 4. 断言下载抛出预期异常
    $this->expectException(FireflyException::class);
    $this->expectExceptionMessage('200000: File has not been uploaded');
    $this->get("/api/v1/attachments/{$attachmentId}/download");
}
```

---

## 八、总结

Firefly III 附件 API 的两阶段模型设计体现了良好的**关注点分离**和**容错性**：

1. **第一阶段（Metadata）**：快速创建记录，支持后续更新元数据，不依赖文件大小和上传时间
2. **第二阶段（Upload）**：专注于文件内容处理和验证，可独立失败重试
3. **安全防护**：多层验证（类型白名单 + 所有权验证 + Demo 用户隔离）
4. **错误处理**：精细的错误分级，不同异常场景返回不同错误码，便于客户端优雅处理
5. **可观测性**：中间状态明确，便于测试和调试

这种设计特别适合大文件上传场景，允许客户端在 metadata 创建失败时快速失败，而不会浪费带宽上传大文件。
