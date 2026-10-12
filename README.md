# 接口测试 Skills

给 Java / Spring Boot（Maven）项目用的三组 Cursor skill。生成接口测试、运行测试、检查偏差。只追加测试，不改已有用例和业务代码。

中文版在 `zh/`，英文版在 `en/`。装其中一套即可。两套的指令名相同：`/api-test-gen`、`/api-test-run`、`/api-test-audit`。

## 安装

四个目录必须一起放进 Cursor 的 skills 目录，不要改名：

- `api-test-gen`
- `api-test-run`
- `api-test-audit`
- `api-test-shared`

Windows：`C:\Users\<用户名>\.cursor\skills\`

macOS / Linux：`~/.cursor/skills/`

手动安装：把 `zh` 或 `en` 里面的四个目录复制进去，覆盖同名目录。装中文就复制 `zh`，装英文就复制 `en`。

让 AI 安装时，把下面这段发给它：

```text
把本仓库 zh 目录下的 api-test-gen、api-test-run、api-test-audit、api-test-shared
复制到我的 Cursor skills 目录（Windows 是 %USERPROFILE%\.cursor\skills，macOS/Linux 是 ~/.cursor/skills）。
四个目录名不要改。如果要英文版，改成复制 en 目录下的同名四个目录。
```

装好后新开一个对话，用 `/` 调用。这三组 skill 不会被自动触发。

## 指令

| 指令 | 作用 |
|---|---|
| `/api-test-gen` | 按代码改动或你指定的 Controller 生成测试。每个接口 1 条连 daily 的真实用例，其余是替身用例。有待确认项时先列行为清单，你确认后才写代码。 |
| `/api-test-run` | 运行测试并分类失败原因。默认只跑替身用例。说 daily 时才跑真实用例。只报告，不改代码。 |
| `/api-test-audit` | 检查缺测试、样本来源、脱敏、真实用例是否会写数据。只报告，不改代码。 |

替身用例分两层，都会进默认 `mvn test`：

- `*ApiTest`：契约层。只加载 Controller，校验参数和返回格式，不连环境。
- `*FlowTest`：边界层。业务代码真实执行，下游（Feign、数据库、Redis、HTTP、MQ）换成假的。

`*IT` 是真实用例，连 daily，不进默认 `mvn test`。

## 日志插件

从日志里取样本时，默认用腾讯云 CLS。这只是默认插件，可以换成别的日志插件。

在项目画像 `.api-test/profile.yaml` 里：

```yaml
sampling:
  cls:
    enabled: false          # true 才查日志
    provider: cls           # 默认腾讯云 CLS。改成其他插件名即停用 CLS
    region: ap-guangzhou    # 仅 provider 为 cls 时使用
    topicId: "{{daily 主题，禁止填生产主题}}"
    app: your-service-name  # 服务名，不是业务 appId
    notes: ""               # provider 不是 cls 时必填：插件名、查询工具、如何只查测试环境
```

`enabled: false` 时不查任何日志。`provider` 不是 `cls` 时，按 `notes` 调用你指定的插件，不调用腾讯云 `SearchLog`。主题或查询范围带生产标识时停止。

## 使用时注意

- 首次生成会在项目根写 `.api-test/profile.yaml`。是否提交由项目自己决定。
- 原始录制在 `<模块>/target/api-test-samples/`，未脱敏，不要提交。多数项目的 `.gitignore` 已忽略 `target/`。
- 真实用例可能产生调用费用或写入测试库。运行前会先列出并等你确认。
- 只连 daily 或测试环境，不连生产。
