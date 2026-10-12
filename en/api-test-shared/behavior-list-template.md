# Behavior list template

api-test-gen step 4 uses this template only when something needs confirmation. After confirmation, append the list to
`<project root>/.api-test/behaviors/<module>/<Controller>.md`. Append by date in the same file. Do not overwrite older sections.
Do not create the file when nothing needs confirmation.

## How to fill it

**Kind**

- `mock`: every boundary is replaced. Included in the default `mvn test`.
- `real`: connects to daily, does not replace dependencies, only records return values. The class name ends with `IT`. Not in the default `mvn test`.

**Layer** (mock cases only)

- `contract`: `<Controller>ApiTest`, `@WebMvcTest`. Replace beans the Controller depends on directly. Check the route, validation, and response shape.
- `boundary`: `<Controller>FlowTest`, `@SpringBootTest` + MockMvc. Business code runs. Replace only types listed in `boundaries`.

**Expectation source**. Use only these values:

| Value | Meaning |
|---|---|
| requirement | Wording the user gave |
| comment | A comment on the method, class, or field |
| annotation | A validation annotation, such as `@NotBlank` |
| doc | An API document |
| existing test | The same kind of rule already expressed by an existing test |
| user confirmed | Given by the user during this confirmation |
| snapshot | Readable only from the implementation. It must also appear under "Needs confirmation" |

**Data source**: where the request body and the fake return value come from, in the order in [fixture-spec.md](fixture-spec.md): `existing sample`, `recorded`, `logs`, `record table`, `doc`, `user`, `synthetic`. The log plugin is `sampling.cls.provider`. The default is Tencent Cloud CLS, and it can be replaced. When no sample is needed, write `none`.

Behaviors already covered by an existing test do not go in the case table. Put them under "Already covered".

## Template

```markdown
## {{yyyy-MM-dd}} {{HTTP method}} {{path}}

- Module: {{module}} (style: {{styleGuide}})
- Scope: {{git diff <base>...HEAD / user named / user selected}}
- Controller: {{class}}#{{method}}
- Boundaries: {{class#method (read / write / billed)}}, ...
- Existing tests: {{test class, or none}}

### Cases

| # | Kind | Layer | Behavior | Request source | Fake and return source | Expectation source |
|---|---|---|---|---|---|---|
| 1 | {{mock / real}} | {{contract / boundary / -}} | {{one sentence: under what condition, what result}} | {{source}} | {{class#method → source}} | {{source}} |

### Already covered

- {{behavior}}: {{test class#method}}

### Needs confirmation

- #{{n}} Expectation source is a snapshot: {{what the code does now}}. Use it as the official expectation?
- #{{n}} Data the real test needs: {{for example a userId that has a resume on daily}}
- #{{n}} This triggers a write or a billed call: {{class#method}}

### Test dependencies to add

{{none / coordinates + reason}}

### Not generated

| Behavior | Reason |
|---|---|
| {{behavior}} | {{for example: a static call cannot be replaced without changing business code}} |
```

## Example

```markdown
## 2026-10-08 POST /address/recognize

- Module: platform-ai-service (style: java21-boot3)
- Scope: git diff master...HEAD
- Controller: AddressController#recognize
- Boundaries: ResourceIdValidUtils#validate (read), AddressBusiness#recognize (billed: calls a model)
- Existing tests: none

### Cases

| # | Kind | Layer | Behavior | Request source | Fake and return source | Expectation source |
|---|---|---|---|---|---|---|
| 1 | mock | contract | Missing textContent is rejected by validation and AddressBusiness is not called | synthetic (from the DTO) | AddressBusiness → none | annotation @NotBlank |
| 2 | mock | contract | A non-blank resourceId with a bad format returns "invalid resource id" and does not call AddressBusiness | synthetic (from the DTO) | ResourceIdValidUtils#validate → synthetic (returns false) | snapshot |
| 3 | mock | contract | On a valid request, data returned by AddressBusiness is placed in the response data as-is | synthetic (from the DTO) | AddressBusiness#recognize → recorded | snapshot |
| 4 | real | - | The address "a public street sample" returns success and province is non-empty | user | not replaced; record AddressBusiness#recognize | user confirmed |

### Already covered

- none

### Needs confirmation

- #2 Expectation source is a snapshot: the failure message for a bad resourceId. Use it as the official expectation?
- #3 Expectation source is a snapshot: the Controller does not transform data. Use it as the official expectation?
- #4 This triggers a billed call: AddressBusiness#recognize calls a model, and each run has a cost.

### Test dependencies to add

none

### Not generated

| Behavior | Reason |
|---|---|
| Many addresses with different model wording | Output is unstable. The real test asserts key fields only |
```
