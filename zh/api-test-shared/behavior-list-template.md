# 行为清单模板

api-test-gen 第 4 步仅在有待确认项时按本模板输出。确认后的清单追加保存到
`<项目根>/.api-test/behaviors/<模块名>/<Controller 类名>.md`，同一文件按日期追加，不覆盖旧内容。
没有待确认项时不创建这个文件。

## 填写规则

**类型**

- `替身`：边界依赖全部替换，进入默认 `mvn test`。
- `真实`：连 daily，不替换依赖，只录制返回值；类名以 `IT` 结尾，不进入默认 `mvn test`。

**层级**（仅替身用例）

- `契约`：`<Controller>ApiTest`，`@WebMvcTest`，替换 Controller 直接依赖的 Bean，验证路由、参数校验、返回格式。
- `边界`：`<Controller>FlowTest`，`@SpringBootTest` + MockMvc，业务代码真实执行，只替换画像 `boundaries` 中的类型。

**期望来源**，只能取以下值：

| 值 | 含义 |
|---|---|
| 需求 | 用户给出的需求或规则原文 |
| 注释 | 方法、类、字段上的注释 |
| 注解 | 校验注解，如 `@NotBlank` |
| 文档 | 接口文档 |
| 已有用例 | 现有测试已表达的同类规则 |
| 用户确认 | 本次确认时用户给出 |
| 现状快照 | 只能从实现代码读出；必须同时列入“待确认” |

**数据来源**：请求体与替身返回值的来源，按 [fixture-spec.md](fixture-spec.md) 的顺序填写：`已有样本`、`录制`、`日志`、`记录表`、`文档`、`用户提供`、`编造`。日志插件由画像 `sampling.cls.provider` 决定，默认腾讯云 CLS，可换成其他日志插件。不需要样本时填 `不需要`。

已被现有测试覆盖的行为不列入用例表，写在“已覆盖”中。

## 模板

```markdown
## {{yyyy-MM-dd}} {{HTTP 方法}} {{接口路径}}

- 模块：{{模块名}}（写法：{{styleGuide}}）
- 范围来源：{{git diff <基线>...HEAD / 用户指定 / 全量勾选}}
- Controller：{{类名}}#{{方法名}}
- 调用链边界：{{类#方法（读 / 写 / 计费）}}，……
- 已有测试：{{测试类名，或 无}}

### 用例

| # | 类型 | 层级 | 行为 | 请求体来源 | 替身与返回值来源 | 期望来源 |
|---|---|---|---|---|---|---|
| 1 | {{替身 / 真实}} | {{契约 / 边界 / -}} | {{一句话：什么条件下，预期什么结果}} | {{来源}} | {{类#方法 → 来源}} | {{来源}} |

### 已覆盖

- {{行为}}：{{测试类#方法}}

### 待确认

- #{{n}} 期望来源为现状快照：{{代码当前的行为}}，是否作为正式期望？
- #{{n}} 真实用例需要的数据：{{如 daily 上存在简历的 userId}}
- #{{n}} 会触发写操作或计费调用：{{类#方法}}

### 需要新增的测试依赖

{{无 / 依赖坐标 + 原因}}

### 不生成

| 行为 | 原因 |
|---|---|
| {{行为}} | {{如：静态调用无法替换，需要改业务代码}} |
```

## 示例

```markdown
## 2026-10-08 POST /address/recognize

- 模块：platform-ai-service（写法：java21-boot3）
- 范围来源：git diff master...HEAD
- Controller：AddressController#recognize
- 调用链边界：ResourceIdValidUtils#validate（读），AddressBusiness#recognize（计费：调用大模型）
- 已有测试：无

### 用例

| # | 类型 | 层级 | 行为 | 请求体来源 | 替身与返回值来源 | 期望来源 |
|---|---|---|---|---|---|---|
| 1 | 替身 | 契约 | 缺 textContent 时被参数校验拦下，不调用 AddressBusiness | 编造（按 DTO） | AddressBusiness → 不需要 | 注解 @NotBlank |
| 2 | 替身 | 契约 | resourceId 非空且格式不对时返回“resourceID格式不正确”，不调用 AddressBusiness | 编造（按 DTO） | ResourceIdValidUtils#validate → 编造（返回 false） | 现状快照 |
| 3 | 替身 | 契约 | 正常请求时，AddressBusiness 返回的 data 原样放进返回体 data | 编造（按 DTO） | AddressBusiness#recognize → 录制 | 现状快照 |
| 4 | 真实 | - | 输入“新疆乌鲁木齐市天山区团结路”返回成功，province 为“新疆维吾尔自治区” | 用户提供 | 不替换；录制 AddressBusiness#recognize | 用户确认 |

### 已覆盖

- 无

### 待确认

- #2 期望来源为现状快照：resourceId 校验失败时提示文案为“resourceID格式不正确”，是否作为正式期望？
- #3 期望来源为现状快照：Controller 不加工 data，是否作为正式期望？
- #4 会触发计费调用：AddressBusiness#recognize 调用大模型，每次运行产生费用。

### 需要新增的测试依赖

无

### 不生成

| 行为 | 原因 |
|---|---|
| 大模型返回内容不同的各种地址 | 输出不稳定，真实用例只断言关键字段 |
```
