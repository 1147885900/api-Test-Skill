<p align="right"><strong>English</strong> · <a href="README.zh-CN.md">简体中文</a></p>

# API Test Skills

Incremental API tests for Java and Spring Boot services built with Maven.

Point an agent at a Controller. It writes three layers of tests, records where every expectation and every JSON sample came from, and leaves your existing tests and business code alone. You confirm the few expectations that cannot be read from an annotation. The agent does not invent a passing assertion to hide a disagreement with the code.

The instructions follow the open [Agent Skills](https://agentskills.io/specification.md) format. The same directories load in Cursor, Claude Code, Codex, GitHub Copilot, VS Code, Gemini CLI, OpenCode, and the other products on the [client list](https://agentskills.io/clients.md). Each product chooses its own install directory. The skill text does not change.

## What you get

Three commands. Install all four directories together. `api-test-shared` is reference material the other three read on demand. It is not a fourth command.

| Command | When you type it | What it is allowed to change |
|---|---|---|
| `/api-test-gen` | After a Controller change, or when you name a Controller | Appends tests under `<basePackage>.apitest`, samples under `src/test/resources/fixtures/`, and a project profile under `.api-test/`. Does not edit an existing test method, business code, or `pom.xml`. |
| `/api-test-run` | When you want the tests executed and the failures sorted | Nothing. It prints a classified report. A profile change is only a suggestion until you accept it. |
| `/api-test-audit` | Before review, or when you suspect a gap | Nothing. It reports missing endpoints, stale or synthetic samples, redaction misses, and real tests that would write or bill. |

Chinese and English instruction sets live in `zh/` and `en/`. Install one set. Both expose the same three command names, so installing both at once gives you two copies of each command.

## Who it is for

Java services on Maven, with Spring Boot 2.x (Java 8, `javax.servlet`) or Spring Boot 3.x (Java 17 or 21, `jakarta.servlet`). JUnit 4 and JUnit 5 are both generated from the version the module already uses. JSON samples follow the library the service already parses with: fastjson2, fastjson, or Jackson.

It is built for the case where a generated test is easy to produce and hard to trust: the assertion encodes the generator's reading of the implementation, the JSON is a made-up happy path, and a "real" test quietly calls a billed model or writes the test database.

## The three layers

Every endpoint gets the same shape. The class names are the contract with Maven Surefire: `*Test` runs in a normal `mvn test`, `*IT` does not.

| Layer | Class | Spring | Downstream | What a pass means |
|---|---|---|---|---|
| Contract | `<Controller>ApiTest` | `@WebMvcTest` slice. No config center, no registry. | The Controller's direct collaborators are fakes. | A missing or invalid field returns the project's real validation message, and the business method is not called. A success case checks that the value the test itself stubbed is returned in the documented shape. |
| Boundary | `<Controller>FlowTest` | Full `@SpringBootTest`, but Feign, Mapper, Redis, HTTP, and MQ are fakes. | Business code runs. | A real branch behaves as confirmed: empty downstream, parse failure, discarded result, or the arguments of a write. Endpoints that only validate and pass data through do not get this class. |
| Real | `<Controller>IT` | Full application, daily or test profile. | Real calls. Selected dependencies are spied so the return value can be recorded. | The call succeeds and the key fields exist and are non-empty. The full model paragraph is not frozen into the assertion. One happy path per endpoint. A second path is added only when you ask. |

`FlowTest` classes in one module extend a single `BoundarySpring`, so the Spring context starts once per Maven process instead of once per class. The run command keeps those classes in one `mvn` invocation for that reason.

A pass is not "the method returned without throwing." Each assertion has a source: an annotation, a comment, an API document, an existing test, a value you confirmed, or a snapshot of the current implementation that you explicitly accepted. When a run disagrees with a confirmed expectation, the assertion stays and the report calls it a suspected defect.

## How a generation run proceeds

1. **Profile.** If `.api-test/profile.yaml` is missing, the agent scans the root POM, module compiler settings, and dependency tree, then shows you the draft. Unknown values stay `unknown`. Versions come from the dependencies and the tests already in the repo, not from a style document that has drifted.
2. **Scope.** With no path named, it maps the current git diff to Controllers, at most three call levels up from a changed Service, client, or DTO. Naming a Controller limits the run to that Controller. "Everything" lists uncovered endpoints and waits for you to pick. More than ten endpoints also waits for a pick.
3. **Behavior list.** When every expectation is an annotation, a comment, a document, or an existing test, and nothing writes, bills, or needs a daily key from you, the agent lists the cases in the chat and writes the code. When something is unconfirmed, it writes a list and asks only about those items. A snapshot of current code is one of those items: you choose whether it becomes the official expectation.
4. **Samples.** Large JSON is stored as a file and registered in one `<endpoint>.sources.yaml` per Controller. A request of at most five fields, or a stub of at most fifteen lines, stays inline in the test method. The source of each sample is recorded.
5. **Code.** New classes go to `<basePackage>.apitest`, next to handwritten tests but not inside their package. If a generated class already exists, methods are appended. Handwritten classes are not edited.
6. **Self-check.** Compile. New mock tests must pass three times in a row. A flip is reported as flaky and the test is rewritten (fixed clock, no order dependence), not by loosening the assertion. New real tests run once, after you have confirmed any write or billed call. A layer that needs daily arguments must show as skipped when those arguments are absent, so a plain `mvn test` on a laptop stays green.

## Where a sample is allowed to come from

The agent walks this list and stops at the first hit. It does not call a write API in order to obtain a sample.

| Order | Source | Used when |
|---|---|---|
| 1 | Existing test | The JSON is already in the repo. It is copied. The original test is not edited. |
| 2 | Recording | A read-only daily IT ran, and `FixtureRecorder` wrote the return value under `target/api-test-samples/`. |
| 3 | Logs | `sampling.cls.enabled` is true and the code logs that return value. The plugin is selectable. See below. |
| 4 | Request table | The profile names a daily request-record table, queried read-only. |
| 5 | Vendor doc | The dependency is a public API with an official example. |
| 6 | You | None of the above worked. The agent names the call and the arguments. You paste the response. |
| 7 | Synthetic | Last resort, built from the return DTO. Failure branches (null, empty list, business error, thrown exception) start here on purpose, because those values are not worth a live call. |

Raw recordings are not redacted. They stay in `target/` until the agent replaces phone numbers, national IDs, emails, tokens, and the field names in your profile, shows you the result, and only then copies it into `src/test/resources/fixtures/`.

## Log plugin

Log sampling is off until you turn it on. The profile key is still `sampling.cls`. That name is the source id from the table above. It does not mean the only implementation is Tencent Cloud CLS.

```yaml
sampling:
  cls:
    enabled: false
    provider: cls                 # cls, or the name of another log plugin
    region: ap-guangzhou          # CLS only. Otherwise unused
    topicId: "<daily-topic-id>"   # CLS only. Never a production topic
    app: your-service-name        # service name, from spring.application.name. Not a business appId
    notes: ""                     # required when provider is not cls
```

`provider: cls` calls Tencent Cloud Log Service. The query is `app:"<service>"` plus the fixed text of the log line, through the CLS `SearchLog` tool, on a daily or test topic. The window starts at one hour and widens to seven days. A topic whose name or id contains `prod`, `pro`, or a production marker is not queried.

Set `provider` to any other name to leave CLS behind. `notes` must then say which plugin or tool to call, which arguments to pass, which field is the service name, and how the query stays on test logs. The agent does not call `SearchLog` in that mode. If the named tool is not connected, it tells you and moves to the next source. It does not invent a log line.

`enabled: false` skips the log source entirely. No log plugin is called.

## The format, and which products load it

A skill is a directory containing `SKILL.md`: YAML frontmatter (`name`, `description`) plus Markdown instructions. That layout is the [Agent Skills specification](https://agentskills.io/specification.md). Supporting files are read only when a step needs them. The specification keeps those links one level away from `SKILL.md`. These skills put the shared templates in the sibling directory `api-test-shared/`, and each `SKILL.md` points there with a relative path.

The specification defines the file format, not the install directory. Copy the four directories into the skills directory your product documents. These are the locations those products document today:

| Product | Personal (this machine) | Project (this repo) | How you invoke |
|---|---|---|---|
| [Cursor](https://cursor.com/docs/context/skills) | `~/.cursor/skills/` or `~/.agents/skills/` | `.cursor/skills/` or `.agents/skills/` | `/api-test-gen`, `/api-test-run`, `/api-test-audit` |
| [Claude Code](https://code.claude.com/docs/en/skills) | `~/.claude/skills/` | `.claude/skills/` | the same `/` names |
| [Codex](https://developers.openai.com/codex/skills) | `~/.agents/skills/` | `.agents/skills/` | `/skills`, or `$api-test-gen` |
| [Gemini CLI](https://geminicli.com/docs/cli/skills/) | `~/.gemini/skills/` or `~/.agents/skills/` | `.gemini/skills/` or `.agents/skills/` | selected from the description after you confirm; `/skills list` shows what was found |
| [VS Code](https://code.visualstudio.com/docs/copilot/customization/agent-skills) / [GitHub Copilot](https://docs.github.com/en/copilot/concepts/agents/about-agent-skills) | `~/.copilot/skills/` or `~/.agents/skills/` | `.github/skills/`, `.agents/skills/`, or `.claude/skills/` | the product's skill picker |

Windows personal paths use `%USERPROFILE%` in place of `~`. `.agents/skills/` is the cross-client directory from the specification, and Codex, Cursor, Gemini CLI, and Copilot all read it. Claude Code reads `.claude/skills/` and does not read `.agents/skills/`. Cursor also scans the Claude and Codex directories, so one copy is often enough for more than one product on the same machine. Junie, OpenCode, Goose, Amp, Roo Code, and the rest of the [client list](https://agentskills.io/clients.md) use the same `SKILL.md` format; use the directory in that product's own docs if it is not in the table. If a path has moved, the product docs win.

Two frontmatter details are worth knowing before you install:

- `name` is the command. The directory name matches it: `api-test-gen`, `api-test-run`, `api-test-audit`. Do not rename them. Relative links between the four directories would break, and the slash command would change.
- `disable-model-invocation: true` tells Cursor and Claude Code not to run the skill unless you type the command. Products that do not implement that field can still select the skill when your message matches the description. The instructions they follow are the same either way.

`api-test-shared` has no `SKILL.md`. Leave it next to the other three. Do not expect it to appear in the slash menu.

## Install

Pick a language, then copy four directories. From a clone of this repo:

```text
en/api-test-gen
en/api-test-run
en/api-test-audit
en/api-test-shared
```

Use the same four names under `zh/` if you want the agent to follow Chinese instructions. The destination is the personal or project directory from the table above. Example for Cursor on Windows, English instructions:

```text
%USERPROFILE%\.cursor\skills\api-test-gen
%USERPROFILE%\.cursor\skills\api-test-run
%USERPROFILE%\.cursor\skills\api-test-audit
%USERPROFILE%\.cursor\skills\api-test-shared
```

You can hand the install to the agent you already have open:

```text
Copy api-test-gen, api-test-run, api-test-audit, and api-test-shared
from the en directory of this repo into the personal skills directory
of the agent I am using. Keep the four directory names.
They must stay siblings, because the skills link to ../api-test-shared.
If I want Chinese instructions, copy those four directories from zh instead.
Do not install both languages.
```

Open a new session after copying. In Cursor and Claude Code, type `/` and the command name. In Codex, open `/skills` or mention `$api-test-gen`.

A project-level install (`.cursor/skills/`, `.claude/skills/`, or `.agents/skills/` inside the service repo) limits the skills to that repo and lets you commit them for the team. A personal install applies to every repo on that machine.

## What shows up in the service repo

```text
<project>/.api-test/profile.yaml                         module versions, boot mode, boundaries, sampling, redaction
<project>/.api-test/behaviors/<module>/<Controller>.md  only when something needed your confirmation
<module>/src/test/java/<basePackage>/apitest/           generated tests, BoundarySpring, TestFixtures, FixtureRecorder
<module>/src/test/resources/fixtures/                   large samples and one sources.yaml per Controller
<module>/target/api-test-samples/                       raw recordings, not redacted, not committed
```

`.api-test/` is independent of any `AGENTS.md`, `harness/`, or rules directory the service already has. The skills read those files as reference and do not write into them. Whether `.api-test/` itself is committed is your decision. `target/` is already ignored by a normal Java `.gitignore`. If it is not, the generate command asks before editing `.gitignore`.

The first run also learns how the contract slice has to start. Some services cannot boot a `@WebMvcTest` while Apollo and the registry are enabled, because the application class scans every Feign client. The profile then records offline flags for the contract layer (`apollo.bootstrap.enabled=false`, eager load off, Eureka and discovery off) and daily flags for the boundary layer. Those flags are passed on the Maven command line. They are not hardcoded into the test class.

On Windows PowerShell the run command quotes every `-D` argument. Without the quotes, PowerShell splits on `.` and Maven reports an unknown lifecycle phase.

## Rules that are not optional

- Daily or test environment only. A startup flag of `prod`, `pro`, or another production marker stops the run.
- A real test that would write or bill is listed on its own and waits for you. Methods named in `dailyRun.forbiddenCalls` are guarded: if the call path reaches them, the test fails with an assertion instead of performing the call.
- Job executors and queue consumers that would register with the shared test environment, or bind a local port, are disabled with an extra `-D` flag discovered from the project. The usual example is a scheduler executor.
- No new Maven dependency unless the behavior list names it and you accept it.
- Reports do not paste personal data. A hit is described by file, field path, and category.
- The generate command will not weaken an assertion to turn a red test green.

## Repository layout

```text
README.md                 this file; GitHub renders it by default
README.zh-CN.md           same document in Chinese
en/                       English SKILL.md set
zh/                       Chinese SKILL.md set
```

<p align="right"><a href="README.md"><strong>English</strong></a> · <a href="README.zh-CN.md">简体中文</a></p>
