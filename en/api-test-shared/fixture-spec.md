# Sample rules

A sample is a request body or a fake return value used by a test. Test code reads samples only through `TestFixtures` (see [support-classes.md](support-classes.md)). Do not embed a large JSON blob in the test.

## Layout and names

```
<module>/src/test/resources/fixtures/
├── <endpoint>.sources.yaml                              one source list for every saved sample of that Controller
├── request/<endpoint>/<method>-<scene>.json             large request bodies only
└── <dependency>/<method>-<scene>.json                   large fake return values only
```

- **Endpoint**: Controller class name without the `Controller` suffix, first letter lowercased. `AddressController` → `address`.
- **Dependency**: class name without a `Proxy`, `Client`, `Business`, `Service`, or `Mapper` suffix, first letter lowercased. `ZhideganProxy` → `zhidegan`.
- **Scene**: lower camel case. Common values: `ok`, `empty`, `bizError`, `missingTextContent`.
- **Inline, no file**: a request body with at most 5 fields, or a fake return value of at most 15 formatted lines. A missing field, an empty list, and a one-line error message belong here.
- **Reuse** a sample that already has `source: recorded` for the same dependency method. Do not record or store another copy for a new case.

## Source list (one per Controller)

Path: `fixtures/<endpoint>.sources.yaml`. Register saved samples only. Do not register JSON that lives inside a test method.

```yaml
- path: relay/relay-recognizeOk.json
  source: recorded                 # existing | recorded | cls | table | doc | user | synthetic
  type: com.shebao.ai.dto.response.RelayAiChatCompletionResponse
  from: "AddressControllerIT#recognize_realAddress_returnsProvinceCityDistrict"
  env: local                       # the environment name the user configured; local when they added no extra args. synthetic uses none
  capturedAt: 2026-10-08
  redacted: []                     # field paths that were replaced
  notes: ""                        # synthetic must say why the value was invented
```

If a saved sample has no matching `path`, or `source` or `type` is empty, `api-test-audit` treats the source as unknown.

## Source order

Search in this order. Stop at the first hit.

| Order | source | What to do | When |
|---|---|---|---|
| 1 | existing | Copy a JSON constant or resource from an existing test into a sample file. Do not edit the original test | A dependency that was tested before |
| 2 | recorded | `FixtureRecorder` writes to `target/api-test-samples/` while a real test runs. Redact, then copy | A read call, and daily is reachable |
| 3 | cls | When the code logs that return value, query test-environment logs using `sampling.cls`. `provider: cls` uses Tencent Cloud CLS. Any other `provider` uses the plugin described in `notes` | `sampling.cls.enabled: true` |
| 4 | table | Read-only query of a request-record table listed in `sampling.recordTables`, on daily | The project has such a table |
| 5 | doc | An official example from a public third-party API | A public model vendor or similar |
| 6 | user | Tell the user which call and which arguments to make, and paste the return value they provide | Nothing above worked |
| 7 | synthetic | Build a value from the return DTO | Last resort |

Use synthetic immediately, with no sampling, when:

- The return value is from a write (create, update, delete, send). Do not write daily data just to sample it.
- The branch needs a failure value: null, an empty list, a business error code, or a thrown exception. Invent it from the branch, and say which branch in `notes`.

## Recording (recorded)

1. In the real test, spy the dependency and pass the return value to `FixtureRecorder.record(dependency, method, value)`. If business code injects an `@FeignClient` interface directly and there is no wrapper, do not spy that interface (the bean is a JDK proxy and the spy may not apply). Spy the project class that calls it, or use the next source.
   When the dependency writes the result into an argument instead of returning it (for example `HttpServletResponse`), use `doAnswer`: call `callRealMethod()` first, then record what was written into the argument.
2. After the run, the raw sample is `<module>/target/api-test-samples/<dependency>/<method>-<seq>.json`. A sibling `.meta.yaml` stores the type and the time.
3. Keep 1 to 3 files per dependency method: one success, one empty result, one business failure is enough.
4. Raw samples are not redacted. Do not copy them into `src` and do not commit them.

## Log sampling (cls)

The profile field is named `sampling.cls`. It means "take the sample from logs". The default plugin is Tencent Cloud CLS. It can be replaced. When creating the profile, ask whether to turn this on and which plugin to use.

When `enabled` is not `true`, skip this source. Do not call any log plugin.

`provider` selects the plugin:

- `cls`: Tencent Cloud CLS. Query a daily or test topic only. Stop, and do not query, when the topic name or TopicId contains prod, pro, or a production marker.
  1. The query must include the project name: `app:"<project>"`. The project name is the service name (for example `platform-ai-service`), from `spring.application.name` or Apollo `app.id`. It is not the business `appId` on the request.
  2. Add the fixed text of that log line. Example: `log.info("getResumeByUserId response {}", ...)` → `getResumeByUserId response`.
  3. Follow the CLS tool: convert timestamps, build the query, then call `SearchLog`. Take `Region` and `TopicId` from the profile. Start with the last 1 hour. If nothing matches, widen up to 7 days.
  4. If the CLS tool is not available in this chat, tell the user to connect the CLS plugin. Do not switch environment and do not invent a sample.
- Any other value: use the plugin written in `notes`. Do not call CLS `SearchLog`. `notes` must name the plugin or tool, the query arguments, which field is the project name, and how to stay on test logs. If that tool is not in this chat, tell the user to connect it. Do not invent a sample and do not query production.

Both plugins follow these rules:

1. Take 1 to 3 hits. Cut the JSON out of the log body.
2. If JSON parsing fails (the log was usually truncated), drop that hit and try the next hit or the next source.
3. The log holds a serialized Java object. When it is a downstream return value, deserialize it into the fake return value. Do not treat it as a raw HTTP body.

## Redaction

Redact before writing `src/test/resources`, and show the result to the user.

1. **By field name** (case-insensitive): a hit is a name equal to an entry in `redact.exactFieldNames`, or a name ending with an entry in `redact.suffixFieldNames` (`userPhone` hits `phone`). Names in `keepFieldNames` are excluded.
2. **By content**: scan every string with these patterns and replace a hit:
   - Phone: `(?<![0-9A-Za-z])1[3-9]\d{9}(?![0-9A-Za-z])`
   - National id: `(?<![0-9A-Za-z])\d{17}[\dXx](?![0-9A-Za-z])`
   - Email: `[\w.+-]+@[\w-]+\.[\w.]+`

   The boundaries exist so a digit run inside an id, a traceId, or a hash is not treated as a phone number. If one of those fields still matches, treat it as a false positive, do not replace it, and tell the user when they review the sample.
3. **Replacement values** keep the type and the shape:

| Category | Replace with |
|---|---|
| Phone | `13800000000` |
| National id | `000000000000000000` |
| Name, nickname | `Test User` |
| Address | `Test Address` |
| Email | `test@example.com` |
| Bank card | `6222000000000000` |
| Token, password, secret | `REDACTED` |
| Any other hit string | `test-data` |

4. Free text such as a resume or a chat message can hide personal data that the regex misses. When showing the sample, ask the user to check those fields.
5. List every replaced field path in `redacted` on the matching entry of `<endpoint>.sources.yaml`. Inline samples in test methods are redacted too.
