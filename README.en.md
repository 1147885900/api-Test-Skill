# API test skills

Three Cursor skills for Java / Spring Boot Maven projects: generate API tests, run them, and audit gaps. New tests are appended. Existing tests and business code stay untouched.

Chinese files are in `zh/`. English files are in `en/`. Install one set. Both use the same commands: `/api-test-gen`, `/api-test-run`, `/api-test-audit`.

## Install

Copy these four directories together into the Cursor skills folder. Do not rename them:

- `api-test-gen`
- `api-test-run`
- `api-test-audit`
- `api-test-shared`

Windows: `C:\Users\<you>\.cursor\skills\`

macOS / Linux: `~/.cursor/skills/`

Manual install: copy the four directories from `zh` or from `en`. Use `zh` for Chinese, `en` for English.

To let an AI install them, send:

```text
Copy api-test-gen, api-test-run, api-test-audit, and api-test-shared
from the en directory of this repo into my Cursor skills folder
(Windows: %USERPROFILE%\.cursor\skills, macOS/Linux: ~/.cursor/skills).
Do not rename the four directories.
For Chinese, copy the same four directories from zh instead.
```

Open a new chat and invoke them with `/`. They are not auto-triggered.

## Commands

| Command | What it does |
|---|---|
| `/api-test-gen` | Generates tests for the current change or a Controller you name. One daily IT per endpoint; the rest are mocks. If something needs confirmation, it lists behaviors first and writes code only after you confirm. |
| `/api-test-run` | Runs tests and classifies failures. The default runs mocks only. Say daily to run the real IT cases. Report only; it does not edit code. |
| `/api-test-audit` | Checks missing tests, sample sources, redaction, and whether an IT would write data. Report only. |

Mock tests have two layers. Both run in a default `mvn test`:

- `*ApiTest`: contract slice. Loads the Controller, checks validation and the response shape, no environment.
- `*FlowTest`: boundary. Business code runs; Feign, database, Redis, HTTP, and MQ are fakes.

`*IT` is the real daily case. It is not part of the default `mvn test`.

## Log plugin

Log sampling defaults to Tencent Cloud CLS. That plugin is replaceable.

In the project profile `.api-test/profile.yaml`:

```yaml
sampling:
  cls:
    enabled: false          # true turns log sampling on
    provider: cls           # default: Tencent CLS. Any other name disables CLS
    region: ap-guangzhou    # used only when provider is cls
    topicId: "{{daily topic; never a production topic}}"
    app: your-service-name  # service name, not a business appId
    notes: ""               # required when provider is not cls: plugin, query tool, how to stay on test logs
```

When `enabled` is false, no log plugin is called. When `provider` is not `cls`, the skill follows `notes` and does not call Tencent `SearchLog`. It stops if the topic or query scope looks like production.

## Notes

- The first generate writes `.api-test/profile.yaml` at the project root. Committing it is the project's choice.
- Raw recordings land in `<module>/target/api-test-samples/`. They are not redacted. Do not commit them. Most projects already ignore `target/`.
- A real IT may bill a model call or write the test database. The run skill lists those cases and waits for confirmation.
- Daily or test environments only. Never production.
