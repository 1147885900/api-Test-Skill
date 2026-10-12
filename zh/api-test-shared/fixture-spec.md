# 样本规范

样本是测试里用到的请求体和替身返回值。测试代码只通过 `TestFixtures` 读取样本（见 [support-classes.md](support-classes.md)），不在测试里内嵌大段 JSON。

## 目录与命名

```
<模块>/src/test/resources/fixtures/
├── <接口名>.sources.yaml                              该 Controller 全部落盘样本的来源，一份
├── request/<接口名>/<方法名>-<场景>.json               仅大段请求体
└── <依赖名>/<方法名>-<场景>.json                       仅大段替身返回值
```

- **接口名**：Controller 类名去掉 `Controller` 后缀，首字母小写。`AddressController` → `address`。
- **依赖名**：类名去掉 `Proxy`、`Client`、`Business`、`Service`、`Mapper` 后缀，首字母小写。`ZhideganProxy` → `zhidegan`。
- **场景**：小写驼峰，常用 `ok`、`empty`、`bizError`、`missingTextContent`。
- **写在测试方法里，不建文件**：请求体不超过 5 个字段；替身返回值格式化后不超过 15 行。缺字段、空列表、一句话的失败文案都属于这类。
- **同一依赖方法已有 `source: recorded` 的样本时直接复用**，不为新用例再录一份、再存一份。

## 来源说明（每个 Controller 一份）

路径：`fixtures/<接口名>.sources.yaml`。只登记落盘的样本，写在测试方法里的不登记。

```yaml
- path: relay/relay-recognizeOk.json
  source: recorded                 # existing | recorded | cls | table | doc | user | synthetic
  type: com.shebao.ai.dto.response.RelayAiChatCompletionResponse
  from: "AddressControllerIT#recognize_realAddress_returnsProvinceCityDistrict" 
  env: daily                       # synthetic 填 none
  capturedAt: 2026-10-08
  redacted: []                     # 已替换的字段路径
  notes: ""                        # synthetic 必须写明编造依据
```

落盘样本在这份清单里没有对应 `path`，或 `source`、`type` 为空，`api-test-audit` 视为来源不明。

## 来源顺序

为每个样本按顺序查找，找到即停：

| 顺序 | source | 做法 | 适用 |
|---|---|---|---|
| 1 | existing | 现有测试里的 JSON 常量或资源文件，复制成样本文件，不改原测试 | 以前测过的依赖 |
| 2 | recorded | 运行真实用例时由 `FixtureRecorder` 录制到 `target/api-test-samples/`，脱敏后转存 | 只读调用，能连 daily |
| 3 | cls | 代码打印了该返回值时，按画像 `sampling.cls` 查测试环境日志。`provider: cls` 用腾讯云 CLS；其他 `provider` 按 `notes` 用指定的日志插件 | 画像 `sampling.cls.enabled: true` |
| 4 | table | 画像 `sampling.recordTables` 配置的请求记录表，只读查询 daily | 项目有请求记录表 |
| 5 | doc | 公开第三方 API 的官方文档示例 | 模型厂商等公开接口 |
| 6 | user | 告诉用户具体调用哪个接口、什么参数，由用户调用后粘贴返回 | 以上都取不到 |
| 7 | synthetic | 按返回类型的 DTO 编造 | 兜底 |

以下情况直接用 synthetic，不采样：

- 写操作（创建、更新、删除、发送）的返回值：不为采样去 daily 写数据。
- 异常分支要用的值：返回 null、空列表、业务失败码、抛异常。按代码分支编造，`notes` 写明对应哪个分支。

## 录制（recorded）

1. 真实用例用 spy 包住依赖，把返回值交给 `FixtureRecorder.record(依赖名, 方法名, 返回值)`。业务代码直接注入 `@FeignClient` 接口、没有包装类时，不 spy 这个接口（Bean 是 JDK 动态代理，spy 可能不生效），改为 spy 调用它的项目内类，或换用下一种来源。
   依赖把结果写进入参而不是返回（如写入 `HttpServletResponse`）时，用 `doAnswer` 先 `callRealMethod()`，再从入参取出写入的内容录制。
2. 运行后原始样本位于 `<模块>/target/api-test-samples/<依赖名>/<方法名>-<序号>.json`，同名 `.meta.yaml` 记录类型和时间。
3. 每个依赖方法保留 1 到 3 份：成功、空结果、业务失败各一份即可。
4. 原始样本未脱敏，不得复制到 `src`，不得提交。

## 日志采样（cls）

画像字段名是 `sampling.cls`，表示从日志取样本。默认插件是腾讯云 CLS，可以改成其他日志插件。生成画像时问用户是否开启，以及用哪一个插件。

`enabled` 不是 `true` 时跳过这一来源，不调用任何日志插件。

`provider` 决定插件：

- `cls`：腾讯云 CLS。只查 daily 或测试主题。主题名或 TopicId 带 prod、pro、生产时停止，不查询。
  1. 查询必须带项目名：`app:"<项目名>"`。项目名是服务名（如 `platform-ai-service`），取 `spring.application.name` 或 Apollo 的 `app.id`，不是接口入参里的业务 `appId`。
  2. 再加这条日志的固定文本，例：`log.info("getResumeByUserId response {}", ...)` → `getResumeByUserId response`。
  3. 按 CLS 工具说明：先换算时间戳、生成查询语句，再调用 `SearchLog`。`Region`、`TopicId` 取画像。时间范围从近 1 小时开始，取不到再逐步放宽到 7 天。
  4. 当前窗口没有 CLS 工具时，告诉用户把 CLS 插件接到这个项目。不改用别的环境，不编造样本。
- 其他值：按 `notes` 使用用户指定的日志插件，不调用 CLS 的 `SearchLog`。`notes` 必须写清插件或工具名、查询参数、用哪个字段表示项目名、如何只查测试环境。当前窗口没有该工具时，告诉用户接入它。不编造样本，不查询生产。

两种插件都遵守：

1. 只取 1 到 3 条，从日志正文中截出 JSON。
2. JSON 解析失败（多半是日志被截断）则放弃这条，换下一条或下一种来源。
3. 日志里是序列化后的 Java 对象。打印的是下游返回值时，可以反序列化成替身返回值；不能当作 HTTP 原始响应体。

## 脱敏

样本写入 `src/test/resources` 前必须脱敏，并把结果给用户过目。

1. **按字段名**（忽略大小写）：与画像 `redact.exactFieldNames` 中某项完全相同，或以 `redact.suffixFieldNames` 中某项结尾（`userPhone` 命中 `phone`），即命中；`keepFieldNames` 中的字段除外。
2. **按内容**：所有字符串值都用正则扫描，命中即替换：
   - 手机号：`(?<![0-9A-Za-z])1[3-9]\d{9}(?![0-9A-Za-z])`
   - 身份证号：`(?<![0-9A-Za-z])\d{17}[\dXx](?![0-9A-Za-z])`
   - 邮箱：`[\w.+-]+@[\w-]+\.[\w.]+`

   前后的边界用来排除 id、traceId、哈希值等长串中的数字片段。仍命中这类字段时判为误报、不替换，过目时向用户说明。
3. **替换值**保持类型和格式：

| 类别 | 替换为 |
|---|---|
| 手机号 | `13800000000` |
| 身份证号 | `000000000000000000` |
| 姓名、昵称 | `测试用户` |
| 地址 | `测试地址` |
| 邮箱 | `test@example.com` |
| 银行卡号 | `6222000000000000` |
| token、密码、密钥 | `REDACTED` |
| 其他命中的字符串 | `测试数据` |

4. 简历正文、聊天内容等自由文本，正则可能漏掉个人信息：过目时单独提示用户检查这些字段。
5. 在 `<接口名>.sources.yaml` 对应条目的 `redacted` 中列出所有替换过的字段路径。写在测试方法里的样本同样要先脱敏。
