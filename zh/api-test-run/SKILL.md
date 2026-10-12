---
name: api-test-run
description: 执行 Java/Spring Boot（Maven）项目的接口测试并分类失败原因。支持替身模式和 daily 模式（带 -Denv=daily 等参数运行 IT 真实用例并收集录制样本），按模块自动切换 JDK、探测 Spring 启动方式、重跑识别不稳定用例；只报告，不修改代码。在用户要求运行测试、跑接口测试、跑 IT、验证改动、跑 daily 用例时使用。
disable-model-invocation: true
---

# 接口测试执行

按画像拼出 Maven 命令运行测试，读取结果并分类失败原因。术语（画像、替身用例、真实用例、样本）与 [api-test-gen](../api-test-gen/SKILL.md) 相同。

## 硬性规则

1. 只报告，不修改测试代码、业务代码和 `pom.xml`。画像的修改建议经用户确认后才写入。
2. Windows PowerShell 下所有 `-D` 参数必须整体加双引号，如 `"-Dspring.profiles.active=local"`；不加引号时 PowerShell 会在 `.` 处拆开参数，Maven 报 `Unknown lifecycle phase`。`-Dtest=A,B` 中的逗号同样需要引号保护。
3. 替身模式不带 daily 参数，除非画像 `boot` 中该层为 `daily`。真实用例（`*IT`）只在 daily 模式下运行。
4. 只连 daily 或测试环境。启动参数中出现 `-Denv=prod`、`-Denv=pro` 或其他生产环境标识时，停止并告知用户。
5. 录制样本未脱敏：不把内容贴进对话，不复制到其他位置。

## 模式与范围

| 模式 | 运行内容 | 启动参数 |
|---|---|---|
| 替身（默认） | `*ApiTest`、`*FlowTest`，以及范围内的其他已有测试 | 按画像 `boot.contract` / `boot.boundary`：`offline` 用 `boot.offlineArgs`，`daily` 用 `commands.dailyArgs` 加 `dailyRun.extraArgs` |
| daily | `*IT` | `commands.dailyArgs` 加 `dailyRun.extraArgs` |
| 全部 | 先替身，后 daily | 分别同上 |

| 范围 | 确定测试类的方法 |
|---|---|
| 当前改动（默认） | `git diff --name-only <baseBranch>...HEAD` 和 `git status --porcelain` 中的测试文件；改动的主代码类按命名（`<类名>ApiTest` / `FlowTest` / `IT` / `Test`）和测试目录中对类名的引用找到测试 |
| 指定 | 用户给出的类或方法，方法写成 `类名#方法名` |
| 模块 | 该模块全部测试 |
| 全量 | 画像中所有模块 |

## 工作流

```
- [ ] 1. 读画像，确定模式、范围和测试类
- [ ] 2. 选择 JDK
- [ ] 3. 探测启动方式（画像 boot 为 unknown 时）
- [ ] 4. 拼命令并运行
- [ ] 5. 读取结果、分类失败
- [ ] 6. 重跑失败用例
- [ ] 7. 收集录制样本（daily 模式）
- [ ] 8. 报告
```

### 1. 读画像

读取 `<项目根>/.api-test/profile.yaml`。不存在时告诉用户先运行 `/api-test-gen` 生成画像；用户只想跑现有测试时，按模块默认命令 `mvn -pl <模块> test` 运行，跳过第 3 步。

daily 模式运行前，列出将运行的真实用例；`.api-test/behaviors/` 中标为写操作或计费的用例，提醒用户并确认后再运行。

### 2. 选择 JDK

Maven 用 `JAVA_HOME` 指向的 JDK 编译和运行测试，必须不低于模块的编译目标（画像 `modules[].java`）。

本机 JDK 路径记录在 `~/.cursor/api-test-local.yaml`（本机文件，不放进项目）：

```yaml
# 本机 JDK 路径，供 api-test-run 按模块切换
jdks:
  "8": "{{JDK 8 安装目录，如 C:/Program Files/Java/jdk1.8.0_202}}"
  "17": "{{JDK 17 安装目录，没有可删除此行}}"
  "21": "{{JDK 21 安装目录}}"
```

文件不存在或缺少所需版本时：

1. 按顺序收集候选目录：
   - 当前 `JAVA_HOME`，以及 `where java`（bash 下 `which -a java`）找到的目录。
   - Windows 注册表：`HKLM\SOFTWARE\JavaSoft\JDK\*` 与 `HKLM\SOFTWARE\JavaSoft\Java Development Kit\*` 的 `JavaHome`。
   - Maven 工具链 `~/.m2/toolchains.xml` 中的 `jdkHome`。
   - 常见安装位置：`C:/Program Files/Java/`、`C:/Program Files/Eclipse Adoptium/`、`C:/Program Files/Zulu/`、`C:/Program Files/Amazon Corretto/`、`C:/Program Files/Microsoft/`、`~/.jdks/`、`/Library/Java/JavaVirtualMachines/`、`/usr/lib/jvm/`。
   - 自定义目录：`JAVA_HOME` 的上一级目录（如 `D:/software/jdk/`），以及各盘符下的 `software/jdk`、`tools/jdk`、`java` 目录。
2. 对候选目录运行 `<目录>/bin/java -version` 确认版本。
3. 把结果告诉用户，询问是否写入该文件；用户不写入时，只在本次对话中使用。都找不到时请用户提供路径。

每条命令前单独设置，不依赖上一条命令留下的环境：

```powershell
$env:JAVA_HOME = '{{JDK 目录}}'; $env:Path = "$env:JAVA_HOME\bin;$env:Path"; {{mvn 命令}}
```

bash 下写作 `JAVA_HOME='{{JDK 目录}}' PATH="$JAVA_HOME/bin:$PATH" {{mvn 命令}}`。

### 3. 探测启动方式

画像 `boot.contract` 或 `boot.boundary` 为 `unknown`，且范围内有对应层的用例时：

1. 取该层一个测试类，带 `boot.offlineArgs` 运行。上下文能启动（不论断言是否通过）记为 `offline`。
2. 启动失败且原因是配置拉取或连接问题时，改带 `commands.dailyArgs` 加 `dailyRun.extraArgs` 再运行一次。能启动记为 `daily`。配置中心里的开关缺失也算配置问题，常见表现是 `BeanDefinitionOverrideException`（缺少 `spring.main.allow-bean-definition-overriding`）、占位符无法解析。
3. 两次都失败：报告最后一个 `Caused by`，保持 `unknown`。
4. 把探测结果作为画像修改建议给用户，确认后写入。

命令行 `-D` 参数没有传到测试 JVM 时（表现为离线参数无效、日志中仍连接 Apollo），在报告中说明，建议改用 `"-DargLine=..."` 传参；`argLine` 会覆盖 pom 中已有的 `argLine`（如覆盖率代理），需用户确认。

### 4. 拼命令并运行

以画像 `commands.testClasses` 为基础，追加启动参数，每个 `-D` 参数单独加引号：

```powershell
mvn -pl {{模块}} "-Dtest={{类1}},{{类2}}" "-Dsurefire.failIfNoSpecifiedTests=false" {{启动参数}} test
```

示例（daily 模式运行一个真实用例，最后一个参数来自 `dailyRun.extraArgs`）：

```powershell
mvn -pl platform-ai-service -am "-Dtest=AddressControllerIT" "-Dsurefire.failIfNoSpecifiedTests=false" "-Denv=daily" "-Dspring.profiles.active=local" "-Dxxl.job.executor.appname=false" test
```

- 契约层和边界层启动方式不同时，按启动方式分组，分别运行。同一组里的 `FlowTest` 放在同一次 `mvn` 中，让 Spring 上下文在这个进程里复用。每个类单独起一次 `mvn` 都是新进程，启动成本会重复付。
- `-Dtest` 会覆盖 Surefire 默认的包含规则，写进去的 `*IT` 也会运行，因此替身模式的 `-Dtest` 中不得出现 `*IT`。
- 输出 `Tests are skipped` 时追加 `"-DskipTests=false"`；测试代码没有编译时追加 `"-Dmaven.test.skip=false"`。
- 报 `Could not resolve dependencies` 且缺的是同项目的其他模块时，追加 `-am`。
- 不加 `-q`；运行时间较长时放到后台，定期检查输出。

### 5. 读取结果、分类失败

先看控制台汇总行 `Tests run: x, Failures: y, Errors: z, Skipped: w`，再读 `<模块>/target/surefire-reports/` 中失败类的 `.txt` 和 `TEST-*.xml`，取最后一个 `Caused by` 作为根因。

| 类别 | 判断依据 | 下一步 |
|---|---|---|
| JDK 不符 | `invalid target release`、`release version 21 not supported`、`UnsupportedClassVersionError` | 检查第 2 步的 JDK 选择 |
| 编译失败 | `COMPILATION ERROR`、`cannot find symbol`、`找不到符号` | 列出文件和行号；被替换的方法签名变了也会在这里暴露 |
| 上下文启动失败 | `Failed to load ApplicationContext`、`NoSuchBeanDefinitionException`、`UnsatisfiedDependencyException`、`BeanDefinitionOverrideException` | 缺 Bean 说明测试缺少替身；配置项无法解析或 Bean 重复定义，说明启动方式不对；`@WebMvcTest` 加载了大量无关 Bean，检查画像 `sliceConfig` |
| 端口冲突 | `port[...] is used`、`Address already in use`、`BindException` | 本机已运行的服务占用了端口，多为定时任务执行器；检查 `dailyRun.extraArgs` 是否关闭了执行器，或该层是否缺少 daily 启动条件 |
| 环境不可用 | `Connection refused`、`UnknownHostException`、`connect timed out`、`ApolloConfigException`、`RedisConnectionFailureException`、`CannotGetJdbcConnectionException` | daily 模式：检查 VPN 和 daily 服务；替身模式：说明有边界依赖没被替换 |
| 守卫拦截 | `AssertionError: 真实用例禁止调用` | 调用链经过了禁止调用的方法，检查请求数据或改为替身用例 |
| 断言失败 | `AssertionError`、`Status expected`、`JSON path`、`expected:` | 疑似缺陷或期望待确认，不改断言 |
| 未执行 | `Tests run: 0`、`No tests matching pattern`、`Tests are skipped` | 检查 `-Dtest` 写法和跳过开关 |
| 全部跳过 | `Skipped` 等于 `Tests run`，且该类带 daily 启动条件 | 没带 daily 参数；该层按画像应为 `daily` 时补上参数重跑 |

### 6. 重跑失败用例

断言失败和环境类失败的方法，用 `"-Dtest=类名#方法名"` 再运行最多 2 次：

- 重跑全部通过或时过时不过：标为“不稳定”，可能依赖时间、执行顺序或共享状态。
- 每次都失败：保持原分类。

编译失败、JDK 不符、上下文启动失败不重跑。

### 7. 收集录制样本

daily 模式运行后，列出 `<模块>/target/api-test-samples/` 下的文件，按依赖名统计数量。只列文件名和数量，不展示内容。告诉用户：这些样本未脱敏，需经 `/api-test-gen` 的第 5 步脱敏、过目后才能写入 `src/test/resources/fixtures/`。

### 8. 报告

```markdown
## 接口测试执行结果

- 模式：{{替身 / daily / 全部}}；范围：{{当前改动 / 指定 / 模块 / 全量}}
- JDK：{{版本与目录}}
- 命令：`{{实际执行的命令，每组一行}}`

### 汇总

| 模块 | 运行 | 通过 | 失败 | 跳过 | 不稳定 |
|---|---|---|---|---|---|
| {{模块}} | {{n}} | {{n}} | {{n}} | {{n}} | {{n}} |

### 失败

| 用例 | 类别 | 根因摘要 | 下一步 |
|---|---|---|---|
| {{类名#方法名}} | {{类别}} | {{最后一个 Caused by 的一行摘要}} | {{建议}} |

### 录制样本

- {{依赖名}}：{{n}} 个，位于 `{{target/api-test-samples/... 路径}}`

### 画像修改建议

- {{如：boot.contract: unknown → offline（依据：离线参数下 AddressControllerApiTest 上下文启动成功）}}
```
