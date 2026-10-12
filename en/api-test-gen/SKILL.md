---
name: api-test-gen
description: Generate incremental API tests for Java/Spring Boot Maven projects. Confirms a behavior list when needed, then writes mock tests (MockMvc + Mockito, included in mvn test) and one daily IT per endpoint. Records and redacts samples. Append-only. Use when the user asks to generate API tests, add JUnit for a change, or generate mock or real daily cases.
disable-model-invocation: true
---

# API test generation

Generate tests for Controller endpoints. Each endpoint gets one real daily IT. The rest are mocks. While the IT runs, record downstream return values, redact them, and reuse them as mock samples.

## Terms

| Term | Meaning |
|---|---|
| Profile | `<project root>/.api-test/profile.yaml`. Module versions, commands, boundary types, sampling, and redaction |
| Boundary | A call outside this process: Feign / Proxy, Mapper, Redis, HTTP client, MQ, current time. Types are listed in `boundaries` |
| Mock test | Every boundary is a Mockito fake. Included in the default `mvn test`. Contract layer `*ApiTest` and boundary layer `*FlowTest` |
| Real test | Class name ends with `IT`. Runs with daily args, calls downstream for real, records but does not replace. Not in the default `mvn test` |
| Sample | Large request bodies and fake return values under `src/test/resources/fixtures/`. One `<endpoint>.sources.yaml` per Controller. Small JSON stays inline in the test method |
| Behavior list | The cases to generate, with where the expectation and the data come from. Write code only after the user confirms items that need confirmation |
| Snapshot | An expectation that can only be read from the implementation. It is an assertion only after the user confirms it |

## Output locations

```
<project root>/.api-test/
├── profile.yaml                                      profile
└── behaviors/<module>/<Controller>.md                confirmed behavior list, appended by date
<module>/src/test/
├── java/<basePackage path>/apitest/                  tests and support classes from this skill, not in the handwritten test package
└── resources/fixtures/                               large samples and one sources.yaml per Controller
<module>/target/api-test-samples/                     raw recordings, do not commit
```

`.api-test/` is independent of project-doc folders (`harness/`, `AGENTS.md`, `.cursor/rules`, and similar). Run normally when those folders are absent. Do not write into them when they exist. Whether to commit `.api-test/` is the project's choice. This skill does not change `.gitignore` rules for that folder.

## Shared files

Read from the sibling directory `api-test-shared/` when the step needs it:

| File | When |
|---|---|
| [profile-template.yaml](../api-test-shared/profile-template.yaml) | Step 1, when the project has no profile |
| [behavior-list-template.md](../api-test-shared/behavior-list-template.md) | Step 4 |
| [fixture-spec.md](../api-test-shared/fixture-spec.md) | Step 5 |
| [support-classes.md](../api-test-shared/support-classes.md) | Step 6, when `TestFixtures` or `FixtureRecorder` is missing |
| [java21-boot3.md](../api-test-shared/java21-boot3.md) | Step 6, `styleGuide: java21-boot3` |
| [java8-boot2.md](../api-test-shared/java8-boot2.md) | Step 6, `styleGuide: java8-boot2` |
| [api-test-run](../api-test-run/SKILL.md) | Steps 5 and 7, when tests must run |

## Hard rules

1. If anything needs confirmation, do not write test code or sample files until the user confirms the behavior list. If nothing needs confirmation, generate directly in step 4.
2. Append only. Do not edit, delete, or reorder existing test classes or methods. Do not change business code or `pom.xml`. Put needed changes in the behavior list and let the user decide.
3. Every expectation has a source. A snapshot must be confirmed by the user. When a self-check fails, do not change the assertion to make it pass. Report a suspected defect.
4. A real test must not trigger a write or a billed call unless that case is listed and confirmed. Methods on the call path that appear in `dailyRun.forbiddenCalls` need a guard that fails the test if they are invoked.
5. Daily or test environments only, read-only. Do not read or use prod data.
6. Raw recordings stay in `target/api-test-samples/`. Do not commit them and do not copy them into `src`. Redact before writing `src/test/resources`, and show the result to the user.
7. Generate test code from the style file for that module. For Java 8, check every banned syntax and API.
8. Do not add dependencies unless they are in the behavior list and confirmed. Do not paste personal data into the chat or the report.
9. Write only the paths listed under Output locations. Do not edit `harness/`, `AGENTS.md`, `.cursor/rules`, or other project-doc files. Read them only as reference.
10. Versions follow the project. Take JUnit, Mockito, and Spring Boot from dependencies and existing tests. When a doc disagrees with the dependencies, follow the dependencies, say so in the report, and do not ask about it.

## Workflow

```
- [ ] 1. Read or create the profile
- [ ] 2. Choose the scope
- [ ] 3. Analyze endpoints
- [ ] 4. Confirm the behavior list
- [ ] 5. Prepare samples
- [ ] 6. Write test code
- [ ] 7. Self-check
- [ ] 8. Report
```

### 1. Read or create the profile

Read `.api-test/profile.yaml`. Use it if it exists. Probe or ask when a field is `unknown` and you need it.

If it does not exist, scan the project using [profile-template.yaml](../api-test-shared/profile-template.yaml):

| Field | How to fill it |
|---|---|
| `modules` | `<modules>` in the root `pom.xml`. Keep modules that have Controllers or that the user asked to test |
| `java` | `maven.compiler.release` / `source` / `target` and `maven-compiler-plugin` in the module and parent poms |
| `springBoot`, `mockito`, `junit` | `mvn -pl <module> dependency:tree "-Dincludes=<groupId>:<artifactId>"`. Do not add `-q` (it swallows the output) |
| `servlet` | Whether main code imports `jakarta.servlet` or `javax.servlet` |
| `junit` (when tests exist) | Existing tests import `org.junit.jupiter` or `org.junit.Test`. Follow the existing tests |
| `json` | Annotations on boundary DTOs (`@JSONField` and similar) and the library the business code uses to parse them |
| `styleGuide` | `java21-boot3` when the compile target is 17+ and Spring Boot is 3.x. `java8-boot2` when the target is 8 and Spring Boot is 2.x. Ask the user for other combinations |
| `exceptionHandler` | `@RestControllerAdvice` / `@ControllerAdvice` in source. If absent, search dependency jars (company frameworks often ship one). If still absent, `unknown` |
| `sliceConfig` | `explicit` when the `@SpringBootApplication` class also has `@ComponentScan`, `@MapperScan`, or `@EnableFeignClients` with `basePackages`. Otherwise `auto` |
| `boundaries` | Actual class names of Feign wrappers (`*Proxy`), `@FeignClient` interfaces injected by business code (including jars, such as `DictClient`), `*Mapper`, Redis helpers, HTTP clients, MQ templates |
| `sampling`, `dailyRun` | Ask the user: whether to sample logs, which log plugin to use (default Tencent Cloud CLS, replaceable), the test-environment topic or query, request-record tables, and which methods real tests must not call. When the plugin is not CLS, write the usage into `sampling.cls.notes` and set `provider` to the plugin name |
| `dailyRun.extraArgs` | When the dependencies include a job executor such as xxl-job, or an MQ consumer, find the setting that disables it and write a `-D` arg. A trial run counts as success when the executor no longer starts in the log |

Write `unknown` when there is no evidence. Do not guess. A project testing doc may be read, but version fields follow dependencies and existing tests. Show the whole profile to the user, then write `.api-test/profile.yaml` after they confirm.

Also run `git check-ignore <module>/target/api-test-samples/x.json`. If it is not ignored, ask the user to add `target/` to `.gitignore`, and edit it only after they agree.

### 2. Choose the scope

| User says | Scope |
|---|---|
| Nothing | Current change: `git diff --name-only <baseBranch>...HEAD` plus untracked files from `git status --porcelain` |
| A Controller, path, or file | Only that part |
| Everything | List endpoints that have no tests and let the user pick. Do not generate all of them at once |

Map changed files to endpoints. A Controller change maps to its endpoints. A Service, Proxy, or DTO change maps upward to Controllers that reference it, at most 3 levels. Skip paths in `exclude`. When there are more than 10 endpoints, let the user pick which ones to do now.

### 3. Analyze endpoints

For each endpoint:

1. Read the Controller method: HTTP method, full path (class mapping plus method mapping), parameter binding, response wrapper.
2. Read validation annotations on the request DTO (`@NotBlank`, `@NotNull`, `@Size`, and so on) and where `@Validated` / `@Valid` sits.
3. Follow the call chain until a boundary. Record branch conditions, whether each boundary call is a read, a write, or a billed call, and the return type.
4. Find existing tests: search the test tree for the path and the Controller class name, and note which behaviors are already covered.
5. Read `.api-test/behaviors/<module>/<Controller>.md` when it exists.

Case mix for each endpoint:

- Contract mocks: one rejection per validation annotation, plus the success response shape.
- Boundary mocks: only when there is a real business branch. A branch means an empty downstream, a business failure, a parse failure, a discarded result, or a write whose arguments must be checked. Do not generate `FlowTest` when the endpoint is only validation and pass-through.
- Real test: one happy path per endpoint. Assert success and that key fields exist. Do not assert the full model or downstream body. Generate it only for read-only endpoints. A write or a billed endpoint needs a separate confirmation. Add a second path only when the user asks for one.

### 4. Confirm the behavior list

When every expectation comes from an annotation, a comment, a doc, or an existing test, and there is no write, no billed call, and no data the user must supply: list the cases in the chat and generate. Do not write a behavior file, and do not ask about each case.

When something needs confirmation, output the list from [behavior-list-template.md](../api-test-shared/behavior-list-template.md), then:

1. Ask only about items under "Needs confirmation". For a snapshot, the choices are "use as the expectation" or "do not use it (suspected defect)".
2. The user supplies data a real test needs (for example a userId that exists on daily). If they do not, put that case under "Not generated". After the data is known, check it against daily allowlists and switches (such as an appId allowlist). Read the local Apollo cache at `C:\opt\data\<app.id>\config-cache\` (Linux / macOS: `/opt/data/<app.id>/config-cache/`).
3. After confirmation, append the list to `.api-test/behaviors/<module>/<Controller>.md`. Do not create that file when nothing needed confirmation.

### 5. Prepare samples

Find data for each sample in the source order in [fixture-spec.md](../api-test-shared/fixture-spec.md). When recording is required:

1. Write the real test first (the real-test part of step 6) and wrap the recording point with `FixtureRecorder`.
2. Run those real tests in daily mode, following [api-test-run](../api-test-run/SKILL.md).
3. Take the recordings from `target/api-test-samples/` and redact them.
4. Follow [fixture-spec.md](../api-test-shared/fixture-spec.md): small JSON goes into the test method; large JSON is redacted into `fixtures/` and registered in that Controller's `sources.yaml`. Show the sample and its source to the user before saving. Reuse an existing recorded sample for the same dependency method. Do not record or store another copy.

If daily is unreachable, tell the user why, use the next source, and say so in the report.

### 6. Write test code

1. If the module has no `TestFixtures` or `FixtureRecorder`, create them from [support-classes.md](../api-test-shared/support-classes.md). If they exist, use them.
2. Read the style file for `styleGuide` and fill the templates.
3. When the test class already exists (a previously generated class of the same name), append methods at the end and add missing fields and imports. Do not touch the rest. Do not touch handwritten test classes. Classes from this skill live only in `<basePackage>.apitest`.
4. On a method-name clash, pick a more specific name. Do not overwrite.
5. One comment line above each method: the behavior number and the expectation source, for example `// #3 expectation: user confirmed`.
6. For a Java 8 module, check every row of the banned table in [java8-boot2.md](../api-test-shared/java8-boot2.md).
7. A layer whose `boot` value is `daily` gets the daily startup condition, so it runs only when daily args are present and is skipped by the default `mvn test`. See "Daily startup condition" in the style file.
8. `FlowTest` classes in the same module extend one boundary base class, and the fakes are declared on that base. See "Shared boundary context" in the style file. Keep three classes: do not merge `ApiTest`, `FlowTest`, and `IT`.

### 7. Self-check

Follow [api-test-run](../api-test-run/SKILL.md):

1. `testCompile` passes. A compile failure is fixed only in code added this time.
2. New mock tests pass 3 runs in a row. A mismatch is "flaky". Fix the style (freeze time, remove order dependence) and rerun.
3. New real tests run once in daily mode.
4. A layer whose startup mode is `daily` is run once more with no extra args. Every test in that layer must be skipped. An error means the daily startup condition is missing, and the default `mvn test` would fail.
5. On an assertion failure: if the test is wrong (bad JSON path, wrong sample), fix the test. If the behavior disagrees with the confirmed expectation, keep the assertion and report a suspected defect.

### 8. Report

```markdown
## API test generation

- Scope: {{current change / named / selected}}, {{n}} endpoints
- Profile: {{already existed / created and confirmed this time}}
- Behavior list: {{.api-test/behaviors/... path}}

### Added

| File | New methods | Kind |
|---|---|---|
| {{path}} | {{n}} | {{mock-contract / mock-boundary / real}} |

### Samples

| Sample | Source | Redacted fields |
|---|---|---|
| {{fixtures/...}} | {{recorded / cls / synthetic ...}} | {{field paths}} |

### Self-check

- Compile: {{pass / failure reason}}
- Mock tests: {{x}}/{{y}} passed (3 runs each); run without args: {{daily layers skipped / n/a}}
- Real tests: {{x}}/{{y}} passed; {{reason not run}}

### Suspected defects

- #{{n}} {{endpoint}}: expected {{expected}}, actual {{actual}}, source {{source}}

### Not generated

- {{behavior}}: {{reason}}

### Needs you

- {{for example: profile rejectResponse unconfirmed, .gitignore missing target/, a test dependency to add}}
```
