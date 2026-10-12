---
name: api-test-gen
description: 为 Java/Spring Boot（Maven）项目按代码改动增量生成接口测试。先输出行为清单请用户确认，再生成替身用例（MockMvc + Mockito，进入 mvn test）和连 daily 的真实用例（IT），并录制、脱敏、保存样本文件；只追加，不修改已有测试。在用户要求生成接口测试、补测试用例、为改动写 JUnit、生成 mock 用例或真实用例时使用。
disable-model-invocation: true
---

# 接口测试生成

为 Controller 接口增量生成测试：每个接口 1 条连 daily 的真实用例，其余为替身用例；真实用例运行时录制下游返回值，脱敏后作为替身用例的样本。

## 术语

| 术语 | 含义 |
|---|---|
| 画像 | `<项目根>/.api-test/profile.yaml`，记录各模块的版本、命令、边界依赖、采样与脱敏配置 |
| 边界依赖 | 本进程之外的调用：Feign / Proxy、Mapper、Redis、HTTP 客户端、MQ、当前时间等，类型列在画像 `boundaries` |
| 替身用例 | 边界依赖全部替换为 Mockito 替身，不连任何环境，进入默认 `mvn test`。分契约层 `*ApiTest` 和边界层 `*FlowTest` |
| 真实用例 | 类名以 `IT` 结尾，带 daily 参数运行，真实调用下游，只录制不替换；不进入默认 `mvn test` |
| 样本 | 大段请求体和替身返回值的 JSON 文件，位于 `src/test/resources/fixtures/`。每个 Controller 一份 `<接口名>.sources.yaml` 登记来源。小 JSON 直接写在测试方法里 |
| 行为清单 | 本次要生成的用例列表，写清期望从哪来、数据从哪来，用户确认后才写代码 |
| 现状快照 | 只能从实现代码读出的期望值，必须经用户确认才能作为断言 |

## 产物位置

```
<项目根>/.api-test/
├── profile.yaml                                 画像
└── behaviors/<模块名>/<Controller 类名>.md        确认过的行为清单，按日期追加
<模块>/src/test/
├── java/<basePackage 路径>/apitest/              本 skill 生成的测试和支持类，不放进手写单测所在的业务包
└── resources/fixtures/                           大段样本，以及每个 Controller 一份 sources.yaml
<模块>/target/api-test-samples/                   原始录制样本，不提交
```

`.api-test/` 与项目规范目录（`harness/`、`AGENTS.md`、`.cursor/rules` 等）相互独立：项目没有规范目录时照常运行；有规范目录时也不往里写。`.api-test/` 是否提交由项目决定，本 skill 不改 `.gitignore` 中与它相关的规则。

## 共享文件

位于本 skill 同级目录 `api-test-shared/`，按需读取：

| 文件 | 何时读 |
|---|---|
| [profile-template.yaml](../api-test-shared/profile-template.yaml) | 第 1 步，项目没有画像时 |
| [behavior-list-template.md](../api-test-shared/behavior-list-template.md) | 第 4 步 |
| [fixture-spec.md](../api-test-shared/fixture-spec.md) | 第 5 步 |
| [support-classes.md](../api-test-shared/support-classes.md) | 第 6 步，模块缺少 `TestFixtures` / `FixtureRecorder` 时 |
| [java21-boot3.md](../api-test-shared/java21-boot3.md) | 第 6 步，画像 `styleGuide: java21-boot3` |
| [java8-boot2.md](../api-test-shared/java8-boot2.md) | 第 6 步，画像 `styleGuide: java8-boot2` |
| [api-test-run](../api-test-run/SKILL.md) | 第 5、7 步需要运行测试时，按其规则执行 |

## 硬性规则

1. 有待确认项时，行为清单经用户确认前不写测试代码、不写样本文件。没有待确认项时按第 4 步直接生成。
2. 只追加：不修改、不删除、不重排已有测试类和测试方法；不改业务代码和 `pom.xml`。需要改动时写进行为清单，由用户决定。
3. 每个期望值都要有来源。现状快照必须经用户确认；自检失败时不为通过而改断言，报告为“疑似缺陷”。
4. 真实用例不得触发写操作或计费调用，除非在行为清单中单独列出并经确认。画像 `dailyRun.forbiddenCalls` 中调用链会经过的方法，必须加禁止调用守卫。
5. 只连 daily 或测试环境，只读；不读取、不使用 prod 数据。
6. 原始录制样本只留在 `target/api-test-samples/`，不提交、不复制进 `src`。写入 `src/test/resources` 前必须脱敏，并给用户过目。
7. 测试代码严格按模块对应的写法文件生成；Java 8 模块逐条核对禁用的语法和 API。
8. 不新增依赖，除非列入行为清单并经确认；对话和报告中不贴个人信息原文。
9. 只写“产物位置”中列出的路径。不修改 `harness/`、`AGENTS.md`、`.cursor/rules` 等项目规范文件；它们只作参考读取。
10. 版本和写法以项目实际为准：JUnit、Mockito、Spring Boot 版本取依赖和现有测试。规范文档与实际依赖不一致时，按实际依赖生成，在报告中说明，不为此提问。

## 工作流

复制下面的清单跟踪进度：

```
- [ ] 1. 读取或生成画像
- [ ] 2. 确定范围
- [ ] 3. 分析接口
- [ ] 4. 行为清单确认
- [ ] 5. 准备样本
- [ ] 6. 写测试代码
- [ ] 7. 自检
- [ ] 8. 报告
```

### 1. 读取或生成画像

读取 `.api-test/profile.yaml`。存在则使用；字段为 `unknown` 的，在用到时探测或询问。

不存在时，按 [profile-template.yaml](../api-test-shared/profile-template.yaml) 扫描项目生成：

| 字段 | 取值依据 |
|---|---|
| `modules` | 根 `pom.xml` 的 `<modules>`；只保留有 Controller 或被要求测试的模块 |
| `java` | 模块及父 pom 的 `maven.compiler.release` / `source` / `target`、`maven-compiler-plugin` 配置 |
| `springBoot`、`mockito`、`junit` | `mvn -pl <模块> dependency:tree "-Dincludes=<groupId>:<artifactId>"`，不加 `-q`（会吞掉输出） |
| `servlet` | 主代码 import 的是 `jakarta.servlet` 还是 `javax.servlet` |
| `junit`（有测试时） | 现有测试 import 的是 `org.junit.jupiter` 还是 `org.junit.Test`，以现有测试为准 |
| `json` | 边界依赖返回的 DTO 上用的注解（`@JSONField` 等）和业务代码解析它们用的库 |
| `styleGuide` | 编译目标 17 及以上且 Spring Boot 3.x 为 `java21-boot3`；编译目标 8 且 Spring Boot 2.x 为 `java8-boot2`；其他组合询问用户 |
| `exceptionHandler` | 源码中的 `@RestControllerAdvice` / `@ControllerAdvice`；源码中没有时，在依赖 jar 中查找（公司框架常自带）；都找不到填 `unknown` |
| `sliceConfig` | 启动类（`@SpringBootApplication` 所在类）上另有 `@ComponentScan`、`@MapperScan`、带 `basePackages` 的 `@EnableFeignClients` 时为 `explicit`，否则为 `auto` |
| `boundaries` | Feign 包装类（如 `*Proxy`）和业务代码直接注入的 `@FeignClient` 接口（含依赖 jar 中的，如 `DictClient`）、`*Mapper`、Redis 工具类、HTTP 客户端、MQ 模板的实际类名 |
| `sampling`、`dailyRun` | 询问用户：是否从日志采样、用哪个日志插件（默认腾讯云 CLS，可改成其他日志插件）、测试环境的主题或查询方式、请求记录表、哪些方法禁止在真实用例中调用。插件不是 CLS 时，把用法写入 `sampling.cls.notes`，`provider` 填插件名 |
| `commands.dailyArgs`、`dailyRun.extraArgs` | 询问用户：本地跑真实用例要追加哪些 `-D`。用户说不需要就留空数组。不要写入 `-Denv=daily`，也不要把模板里的例子直接抄进去。只有用户确认需要的参数才写入 |

取不到证据的值写 `unknown`，不猜。项目有测试规范文档时可参考，但版本类字段以依赖和现有测试为准。生成后整份给用户确认，再写入 `.api-test/profile.yaml`。

同时检查 `git check-ignore <模块>/target/api-test-samples/x.json`：没有被忽略时，提示用户在 `.gitignore` 中加入 `target/`，经同意后再改。

### 2. 确定范围

| 用户说法 | 范围 |
|---|---|
| 未指定 | 当前改动：`git diff --name-only <baseBranch>...HEAD` 加 `git status --porcelain` 中的未提交文件 |
| 指定 Controller、接口路径或文件 | 只处理指定的部分 |
| 全量 | 列出所有没有测试的接口，让用户勾选，不一次全部生成 |

把改动文件映射到接口：Controller 改动对应其接口；Service、Proxy、DTO 等改动，向上查找引用它的 Controller，最多 3 层。跳过画像 `exclude` 中的路径。接口超过 10 个时，让用户勾选本次处理哪些。

### 3. 分析接口

对每个接口：

1. 读 Controller 方法：HTTP 方法、完整路径（类上与方法上的映射拼接）、参数绑定方式、返回包装。
2. 读请求 DTO 的校验注解（`@NotBlank`、`@NotNull`、`@Size` 等）和 `@Validated` / `@Valid` 的位置。
3. 沿调用链读到边界依赖为止，记录：分支条件、每个边界调用是读、写还是计费、返回值类型。
4. 查找已有测试：在测试目录中搜索接口路径和 Controller 类名，读出已覆盖的行为。
5. 读取 `.api-test/behaviors/<模块名>/<Controller 类名>.md`（如有），了解以前确认过的行为。

每个接口的用例组合：

- 契约层替身用例：参数校验拒绝（每个校验注解一条）、正常返回格式。
- 边界层替身用例：只在有业务分支时生成。分支指下游为空、业务失败、解析失败、结果被丢弃，以及写操作要核对入参。只有参数校验和原样返回的接口不生成 `FlowTest`。
- 真实用例：每个接口 1 条正常路径。断言只写调用成功和关键字段存在，不写大模型或下游的完整正文。只读接口才生成，写接口或计费接口需单独确认。用户明确要求第二条路径时才再加 1 条。

### 4. 行为清单确认

期望全部来自注解、注释、文档或已有用例，且没有写操作、计费调用、要用户提供的数据时：在对话里列出用例后直接生成，不写行为清单文件，不为每条用例单独提问。

有待确认项时，按 [behavior-list-template.md](../api-test-shared/behavior-list-template.md) 输出行为清单，然后：

1. 只对“待确认”里的项提问。现状快照的选项为“作为期望”或“不作为期望（疑似缺陷）”。
2. 真实用例需要的数据（如 daily 上存在的 userId）由用户提供；用户没有时，该用例放入“不生成”。数据确定后，对照 daily 配置中的白名单和开关（如 appId 白名单）检查一遍；用 Apollo 的项目可读本地缓存 `C:\opt\data\<app.id>\config-cache\`（Linux / macOS 为 `/opt/data/<app.id>/config-cache/`）。
3. 确认后把清单追加到 `.api-test/behaviors/<模块名>/<Controller 类名>.md`。没有待确认项时不创建这个文件。

### 5. 准备样本

按 [fixture-spec.md](../api-test-shared/fixture-spec.md) 的来源顺序为每个样本找数据。需要录制时：

1. 先写真实用例（第 6 步的真实用例部分），在录制点用 `FixtureRecorder` 包住。
2. 按 [api-test-run](../api-test-run/SKILL.md) 的 daily 模式运行这些真实用例。
3. 从 `target/api-test-samples/` 取出录制结果，按规则脱敏。
4. 按 [fixture-spec.md](../api-test-shared/fixture-spec.md) 判断：小的写进测试方法；大的脱敏后写入 `fixtures/`，来源登记到该 Controller 的 `sources.yaml`。落盘前把样本和来源给用户过目。同一依赖方法已有录制样本时复用，不再录、不再存。

daily 连不上时，告诉用户原因，改用下一种来源，并在报告中注明。

### 6. 写测试代码

1. 模块缺少 `TestFixtures` 或 `FixtureRecorder` 时，按 [support-classes.md](../api-test-shared/support-classes.md) 创建；已存在则直接使用。
2. 按画像 `styleGuide` 读对应写法文件，套用模板生成用例。
3. 测试类已存在（以前生成的同名类）时，在类末尾追加方法，补上缺少的字段和 import，其他内容不动。人工写的其他测试类不动，本 skill 的类只建在 `<basePackage>.apitest`。
4. 方法名冲突时换一个更具体的名字，不覆盖。
5. 每个方法上一行注释写行为清单编号和期望来源，如 `// #3 期望来源：用户确认`。
6. Java 8 模块逐条核对 [java8-boot2.md](../api-test-shared/java8-boot2.md) 的禁用表。
7. 画像 `boot` 中为 `daily` 的层，测试类加 daily 启动条件，只在带 daily 参数时运行，默认 `mvn test` 跳过；写法见写法文件的“daily 启动条件”。
8. 同一模块的 `FlowTest` 继承同一个边界基类，替身都声明在基类上。见写法文件的“边界层共用上下文”。三个测试类保持独立：`ApiTest`、`FlowTest`、`IT` 不合并到一个类里。

### 7. 自检

按 [api-test-run](../api-test-run/SKILL.md) 的规则执行：

1. `testCompile` 通过。编译失败只修本次新增的代码。
2. 新增替身用例连续运行 3 次，3 次都通过才算通过；结果不一致标为“不稳定”，修正写法（如固定时间、去掉顺序依赖）后重跑。
3. 新增真实用例在 daily 模式下运行 1 次。
4. 启动方式为 `daily` 的层，再不带任何参数运行一次：应全部显示为跳过；报错说明缺少 daily 启动条件，会让默认 `mvn test` 失败。
5. 断言失败时先判断：测试写错（选错 JSON 路径、样本不对）就修测试；行为与确认的期望不符就保留断言，报告为“疑似缺陷”。

### 8. 报告

```markdown
## 接口测试生成结果

- 范围：{{当前改动 / 指定 / 全量勾选}}，共 {{n}} 个接口
- 画像：{{已存在 / 本次新生成并已确认}}
- 行为清单：{{.api-test/behaviors/... 路径}}

### 新增与追加

| 文件 | 新增方法数 | 类型 |
|---|---|---|
| {{路径}} | {{n}} | {{替身-契约 / 替身-边界 / 真实}} |

### 样本

| 样本 | 来源 | 脱敏字段 |
|---|---|---|
| {{fixtures/... 路径}} | {{recorded / cls / synthetic ...}} | {{字段路径}} |

### 自检

- 编译：{{通过 / 失败原因}}
- 替身用例：{{x}}/{{y}} 通过（各运行 3 次）；不带参数运行：{{daily 启动的层已跳过 / 不适用}}
- 真实用例：{{x}}/{{y}} 通过；{{未运行原因}}

### 疑似缺陷

- #{{n}} {{接口}}：期望 {{期望}}，实际 {{实际}}，期望来源 {{来源}}

### 未生成

- {{行为}}：{{原因}}

### 需要你处理

- {{如：画像 rejectResponse 待确认、.gitignore 待补充、需新增的测试依赖}}
```
