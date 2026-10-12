<p align="right"><a href="README.md">English</a> · <strong>简体中文</strong></p>

# 接口测试 Skills

面向 Maven 构建的 Java / Spring Boot 服务，按改动增量生成接口测试。

把一个 Controller 交给 agent。它会写出三层测试，并写明每条期望、每份 JSON 样本从哪来。已有测试和业务代码保持原样。注解里读不到的期望，由你确认。agent 不会为了让测试变绿去改断言。

指令遵循开放的 [Agent Skills](https://agentskills.io/specification.md) 规范。同一套目录可以装进 Cursor、Claude Code、Codex、GitHub Copilot、VS Code、Gemini CLI、OpenCode，以及[客户端列表](https://agentskills.io/clients.md)里的其他产品。各产品的安装目录不同，技能正文不变。

## 你得到什么

三条指令。四个目录要一起安装。`api-test-shared` 是另外三条按步骤读取的参考材料，不是第四条指令。

| 指令 | 什么时候用 | 允许改什么 |
|---|---|---|
| `/api-test-gen` | Controller 有改动，或你点名了某个 Controller | 在 `<basePackage>.apitest` 追加测试，在 `src/test/resources/fixtures/` 写样本，在 `.api-test/` 写项目画像。不改已有测试方法，不改业务代码，不改 `pom.xml`。 |
| `/api-test-run` | 要执行测试，并把失败原因分开 | 什么都不改。它输出一份分类报告。画像修改只作为建议，你接受之后才写入。 |
| `/api-test-audit` | 评审之前，或你怀疑有缺口 | 什么都不改。它报告缺测试的接口、编造或过期的样本、未脱敏字段，以及会写库或产生费用的真实用例。 |

中文指令在 `zh/`，英文指令在 `en/`。只安装一套。两套的指令名相同，同时安装会出现两条同名命令。

## 适合什么项目

Maven 上的 Java 服务：Spring Boot 2.x（Java 8，`javax.servlet`）或 Spring Boot 3.x（Java 17 或 21，`jakarta.servlet`）。JUnit 4 和 JUnit 5 都按该模块已经在用的版本生成。样本使用的 JSON 库跟服务现有解析方式一致：fastjson2、fastjson 或 Jackson。

它针对的是这种测试：生成很快，但评审时无法判断能不能信。断言只是生成器对实现的一种读法，JSON 是编出来的成功路径，而一条所谓的真实用例会在你没注意到的时候调用计费模型或写入测试库。

## 三层测试

每个接口都是这个形状。类名后缀跟 Maven Surefire 的默认包含规则对齐：`*Test` 进入普通 `mvn test`，`*IT` 不进入。

| 层 | 类 | Spring | 下游 | 怎样才算通过 |
|---|---|---|---|---|
| 契约 | `<Controller>ApiTest` | `@WebMvcTest` 切片。不连配置中心，不连注册中心。 | Controller 直接依赖的对象都是替身。 | 缺字段或格式不对时，返回项目真实的校验文案，并且业务方法没有被调用。成功用例检查：测试自己打桩的返回值，按约定的结构回到了响应里。 |
| 边界 | `<Controller>FlowTest` | 完整的 `@SpringBootTest`，但 Feign、Mapper、Redis、HTTP、MQ 都是替身。 | 业务代码真实执行。 | 确认过的分支按预期发生：下游为空、解析失败、结果被丢弃，或写操作的入参正确。只有参数校验和原样返回的接口不生成这个类。 |
| 真实 | `<Controller>IT` | 完整应用，daily 或测试环境。 | 真实调用。需要录制的依赖套上 spy。 | 调用成功，关键字段存在且非空。不断言大模型返回的整段正文。每个接口一条正常路径。你明确要求时才加第二条。 |

同一模块的 `FlowTest` 继承同一个 `BoundarySpring`。Spring 上下文在一次 Maven 进程里只启动一次，而不是每个测试类启动一次。执行指令会把这些类放进同一次 `mvn`，就是为了这件事。

通过不等于“方法没抛异常”。每条断言都有来源：注解、注释、接口文档、已有用例、你确认过的值，或你明确接受的当前实现快照。运行结果和已确认的期望不一致时，断言保留，报告里记为疑似缺陷。

## 一次生成怎么走

1. **画像。** 没有 `.api-test/profile.yaml` 时，agent 扫描根 POM、模块编译配置和依赖树，把草稿给你看。没有证据的字段写 `unknown`。版本以依赖和仓库里已有测试为准，不以一份已经过时的规范文档为准。
2. **范围。** 你没指定路径时，它把当前 git diff 映射到 Controller；Service、客户端、DTO 的改动最多向上找三层。点名 Controller 就只处理那一个。说全量时，先列出还没有测试的接口，等你勾选。超过十个接口同样等你勾选。
3. **行为清单。** 期望都能从注解、注释、文档或已有用例读出，并且没有写操作、计费调用、需要你提供的 daily 数据时，agent 在对话里列出用例后直接写代码。有待确认项时先出清单，只问这些项。当前实现的快照属于待确认：由你决定它算不算正式期望。
4. **样本。** 大段 JSON 落成文件，登记在该 Controller 的一份 `<接口名>.sources.yaml` 里。请求体不超过 5 个字段，或替身返回值格式化后不超过 15 行，直接写在测试方法里。每份样本都记录来源。
5. **代码。** 新类放在 `<basePackage>.apitest`，和手写测试相邻，但不进手写测试的包。以前生成过的同名类只在末尾追加方法。手写测试类不动。
6. **自检。** 先编译。新增的替身用例要连续通过 3 次。时过时不过标为不稳定，然后改写法（固定时间、去掉顺序依赖），不放宽断言。新增的真实用例跑 1 次，写操作和计费调用要你确认之后才跑。需要 daily 参数才能启动的层，在不带这些参数时必须显示为跳过，这样笔记本上的普通 `mvn test` 仍然是绿的。

## 样本允许从哪来

agent 按这张表往下找，找到就停。它不会为了拿到样本去调用写接口。

| 顺序 | 来源 | 什么时候用 |
|---|---|---|
| 1 | 已有测试 | JSON 已经在仓库里。复制出来，不改原测试。 |
| 2 | 录制 | 只读的 daily 真实用例跑过，`FixtureRecorder` 把返回值写到了 `target/api-test-samples/`。 |
| 3 | 日志 | `sampling.cls.enabled` 为 true，并且代码打印了这个返回值。用哪个插件可以换，见下一节。 |
| 4 | 请求记录表 | 画像里配置了 daily 的请求记录表，只读查询。 |
| 5 | 厂商文档 | 依赖是有官方示例的公开 API。 |
| 6 | 你 | 上面都没有。agent 说明要调哪个接口、什么参数，你粘贴返回。 |
| 7 | 编造 | 最后的兜底，按返回 DTO 构造。失败分支（null、空列表、业务错误、抛异常）直接从这里开始，因为这些值不值得打一次真实调用。 |

原始录制没有脱敏。它们留在 `target/`，直到 agent 按画像替换手机号、身份证号、邮箱、token 和指定字段名，把结果给你看过，才复制进 `src/test/resources/fixtures/`。

## 日志插件

日志采样默认关闭。画像里的键仍然叫 `sampling.cls`。这是上一张表里的来源编号，不是“只能用腾讯云 CLS”。

```yaml
sampling:
  cls:
    enabled: false
    provider: cls                 # cls，或其他日志插件的名字
    region: ap-guangzhou          # 仅 CLS 使用，否则填 unused
    topicId: "<daily 主题 id>"     # 仅 CLS 使用。禁止填生产主题
    app: your-service-name        # 服务名，取 spring.application.name。不是业务 appId
    notes: ""                     # provider 不是 cls 时必填
```

`provider: cls` 时走腾讯云日志服务。查询语句是 `app:"<服务名>"` 加上这条日志的固定文本，通过 CLS 的 `SearchLog` 查 daily 或测试主题。时间窗口从最近 1 小时开始，取不到再放到 7 天。主题名或 TopicId 里带 `prod`、`pro` 或生产字样时，不查询。

把 `provider` 改成其他名字，就不再使用 CLS。这时 `notes` 必须写清：调用哪个插件或工具、传什么参数、哪个字段是服务名、怎样把查询限制在测试环境。这种模式下不会调用 `SearchLog`。当前会话里没有这个工具时，agent 会告诉你，然后改用下一种来源。它不会编造一条日志。

`enabled: false` 时整段日志来源跳过，不调用任何日志插件。

## 格式，以及哪些软件能加载

一个 skill 就是一个含有 `SKILL.md` 的目录：YAML frontmatter（`name`、`description`）加上 Markdown 指令。这个布局是 [Agent Skills 规范](https://agentskills.io/specification.md)。旁边的参考文件只在某一步用到时才读取。规范要求这些链接离 `SKILL.md` 只有一层。本仓库把共用模板放在同级目录 `api-test-shared/`，每份 `SKILL.md` 用相对路径指向它。

规范规定的是文件格式，不是安装目录。把四个目录复制到你的产品文档里写的 skills 目录。下面是这些产品目前文档中的位置：

| 软件 | 个人（本机全部项目） | 项目（当前仓库） | 怎么调用 |
|---|---|---|---|
| [Cursor](https://cursor.com/docs/context/skills) | `~/.cursor/skills/` 或 `~/.agents/skills/` | `.cursor/skills/` 或 `.agents/skills/` | `/api-test-gen`、`/api-test-run`、`/api-test-audit` |
| [Claude Code](https://code.claude.com/docs/en/skills) | `~/.claude/skills/` | `.claude/skills/` | 同样用这些 `/` 名字 |
| [Codex](https://developers.openai.com/codex/skills) | `~/.agents/skills/` | `.agents/skills/` | `/skills`，或 `$api-test-gen` |
| [Gemini CLI](https://geminicli.com/docs/cli/skills/) | `~/.gemini/skills/` 或 `~/.agents/skills/` | `.gemini/skills/` 或 `.agents/skills/` | 按 description 匹配后需你确认；`/skills list` 查看已发现的技能 |
| [VS Code](https://code.visualstudio.com/docs/copilot/customization/agent-skills) / [GitHub Copilot](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills) | `~/.copilot/skills/` 或 `~/.agents/skills/` | `.github/skills/`、`.agents/skills/` 或 `.claude/skills/` | 该产品的 skill 选择器 |

Windows 上个人目录把 `~` 换成 `%USERPROFILE%`。`.agents/skills/` 是规范里的跨产品目录，Codex、Cursor、Gemini CLI 和 Copilot 都会读它。Claude Code 读的是 `.claude/skills/`，不读 `.agents/skills/`。Cursor 还会扫描 Claude 和 Codex 的目录，所以本机往往放一份就能给多个产品用。Junie、OpenCode、Goose、Amp、Roo Code，以及[客户端列表](https://agentskills.io/clients.md)里的其他产品，用的是同一套 `SKILL.md` 格式；表里没有的，按其自己的文档选目录。路径若有变动，以产品文档为准。

安装前有两个 frontmatter 细节：

- `name` 就是命令名。目录名必须和它一致：`api-test-gen`、`api-test-run`、`api-test-audit`。不要改名。四个目录之间的相对链接会断，斜杠命令也会变。
- `disable-model-invocation: true` 让 Cursor 和 Claude Code 只在你输入命令时执行。没有实现这个字段的产品，仍可能在你的话和 description 对得上时自动选用。无论哪种触发方式，执行的指令正文相同。

`api-test-shared` 没有 `SKILL.md`。把它跟另外三个目录放在一起。不要指望它出现在斜杠菜单里。

## 安装

先选语言，再复制四个目录。克隆本仓库之后，英文指令是：

```text
en/api-test-gen
en/api-test-run
en/api-test-audit
en/api-test-shared
```

希望 agent 按中文指令工作，就复制 `zh/` 下同名的四个目录。目标位置是上一张表里的个人目录或项目目录。Windows 上装到 Cursor、使用英文指令的例子：

```text
%USERPROFILE%\.cursor\skills\api-test-gen
%USERPROFILE%\.cursor\skills\api-test-run
%USERPROFILE%\.cursor\skills\api-test-audit
%USERPROFILE%\.cursor\skills\api-test-shared
```

也可以把安装交给当前已经打开的 agent：

```text
把本仓库 en 目录下的 api-test-gen、api-test-run、api-test-audit、api-test-shared
复制到我正在使用的这个 agent 的个人 skills 目录。四个目录名保持不变。
它们必须放在同一层，因为技能里用相对路径引用 ../api-test-shared。
如果我要中文指令，改为复制 zh 目录下的这四个目录。
不要两种语言都装。
```

复制之后新开一个会话。Cursor 和 Claude Code 里输入 `/` 再加命令名。Codex 里打开 `/skills`，或提及 `$api-test-gen`。

装在项目里（服务仓库中的 `.cursor/skills/`、`.claude/skills/` 或 `.agents/skills/`）时，技能只对这个仓库生效，也可以提交给团队。装在个人目录时，对本机所有仓库生效。

## 服务仓库里会出现什么

```text
<项目>/.api-test/profile.yaml                         模块版本、启动方式、边界依赖、采样和脱敏
<项目>/.api-test/behaviors/<模块>/<Controller>.md      只有存在待确认项时才有
<模块>/src/test/java/<basePackage>/apitest/           生成的测试、BoundarySpring、TestFixtures、FixtureRecorder
<模块>/src/test/resources/fixtures/                   大段样本，以及每个 Controller 一份 sources.yaml
<模块>/target/api-test-samples/                       未脱敏的原始录制，不提交
```

`.api-test/` 和服务里已有的 `AGENTS.md`、`harness/`、规则目录互不干扰。技能只把那些文件当参考读，不往里面写。`.api-test/` 交不提交，由项目自己决定。正常的 Java `.gitignore` 已经忽略 `target/`。如果没有忽略，生成指令会先问你，再改 `.gitignore`。

第一次运行还会确认契约层怎么启动。有的服务在 Apollo 和注册中心开着时起不来 `@WebMvcTest`，因为启动类会扫描全部 Feign 客户端。画像会为契约层记下离线参数（`apollo.bootstrap.enabled=false`、关闭 eager load、关闭 Eureka 和 discovery），为边界层记下 daily 参数。这些参数加在 Maven 命令行上，不写进测试类。

在 Windows PowerShell 里，执行指令会给每个 `-D` 参数加引号。不加引号时，PowerShell 会在 `.` 处把参数拆开，Maven 报未知的 lifecycle phase。

## 不会放开的规则

- 只连 daily 或测试环境。启动参数里出现 `prod`、`pro` 或其他生产标识时，直接停止。
- 会写库或计费的真实用例单独列出，等你确认。`dailyRun.forbiddenCalls` 里的方法会加守卫：调用链走到它们时，测试以断言失败结束，而不是真的执行。
- 会向共享测试环境注册、或占用本机端口的定时任务执行器和队列消费者，用项目里现成的 `-D` 开关关掉。常见的是调度执行器。
- 不新增 Maven 依赖，除非行为清单里写了并且你接受。
- 报告里不贴个人信息原文。命中项只写文件、字段路径和类别。
- 生成指令不会为了把红的测试改绿而放宽断言。

## 仓库结构

```text
README.md                 英文说明；GitHub 默认展示这一份
README.zh-CN.md           本文件
en/                       英文 SKILL.md
zh/                       中文 SKILL.md
```

<p align="right"><a href="README.md">English</a> · <a href="README.zh-CN.md"><strong>简体中文</strong></a></p>
