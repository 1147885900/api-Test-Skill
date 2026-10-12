---
name: api-test-audit
description: Audit Java/Spring Boot Maven API tests and report gaps by severity: missing endpoints, missing mock or real layers, synthetic or stale samples, missing redaction, IT that triggers writes, and style that does not match the module Java version. Full, scoped, or current-change. Report only. Use when the user asks to review tests, check coverage, or validate samples.
disable-model-invocation: true
---

# API test audit

Compare the profile, behavior lists, and code. Report gaps and problems by severity. Terms (profile, mock test, real test, sample, behavior list, snapshot) match [api-test-gen](../api-test-gen/SKILL.md).

## Hard rules

1. Report only. Do not edit test code, business code, samples, or the profile. Fixes go to `/api-test-gen` or the user.
2. Do not paste personal data. For a phone number or an id number, write the file, the field path, and the category.
3. Every finding needs evidence (file path and line, or a command-output summary). Do not report a finding without evidence.

## Scope

| User says | Scope |
|---|---|
| Nothing | Current change: `git diff --name-only <baseBranch>...HEAD` plus `git status --porcelain`. Map files to endpoints the same way as step 2 of [api-test-gen](../api-test-gen/SKILL.md) |
| A module, Controller, or path | Only that part |
| Everything | Every module in the profile. Skip paths in `exclude` |

When there is no profile, skip checks in B and D that need it, and say so. Run the other checks.

## Workflow

```
- [ ] 1. Read the profile and behavior lists
- [ ] 2. List endpoints and tests
- [ ] 3. Compile test code
- [ ] 4. Check each item
- [ ] 5. Report
```

### 1. Read the profile and behavior lists

Read `.api-test/profile.yaml` and `.api-test/behaviors/**`. Sample rules are in [fixture-spec.md](../api-test-shared/fixture-spec.md). Style rules are in [java21-boot3.md](../api-test-shared/java21-boot3.md) and [java8-boot2.md](../api-test-shared/java8-boot2.md).

### 2. List endpoints and tests

1. Scan Controllers in scope. Join class-level and method-level `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, and `@PatchMapping` into the full path.
2. Search the test tree for each path and Controller class name. Build the map "endpoint → test class and method".
3. Classify by suffix: `*ApiTest` and `*FlowTest` are mocks, `*IT` is real, anything else is an existing test.

### 3. Compile test code

Run `commands.testCompile` with the JDK and quoting rules from [api-test-run](../api-test-run/SKILL.md). Do not run the tests. A renamed or re-signed method fails here. Report a compile failure as D1 and continue the other checks.

### 4. Checks

**A Coverage**

| Id | Check | Severity |
|---|---|---|
| A1 | An endpoint in scope has no test. Under the current-change scope, a changed endpoint has no new or updated test | Medium |
| A2 | An endpoint has only mocks and no real test, or only a real test and no mocks | Low |
| A3 | A confirmed case in the behavior list has no matching test method (match the `// #n` comment and the method name) | Medium |
| A4 | A validated request-DTO field has no rejection case | Low |

**B Samples**

| Id | Check | Severity |
|---|---|---|
| B1 | A saved sample has no matching `path` in `fixtures/<endpoint>.sources.yaml`, or that entry's `source` or `type` is empty | Medium |
| B2 | The class named by `type` no longer exists, or a field in the sample is missing from the class, or a new field on the class is missing from the sample | Medium |
| B3 | `source: synthetic` but the dependency is a read that could be recorded | Low |
| B4 | `capturedAt` is more than 180 days ago | Low |
| B5 | No test references the sample file | Low |

When the B2 class is inside a dependency jar and the source is unreadable, skip the field comparison and only check that the class exists.

**C Safety**

| Id | Check | Severity |
|---|---|---|
| C1 | A file under `src/test/resources/fixtures/` matches a personal-data regex (phone, national id, email), or a field hit by the profile `redact` rules is not the required replacement value | High |
| C2 | A file under `target/api-test-samples/` is tracked by git, or that directory is not ignored by `.gitignore` | High |
| C3 | A real test's call path goes through a method in `dailyRun.forbiddenCalls` and has no matching guard | High |
| C4 | A real test's call path has a write or a billed call, and the behavior list has no separate confirmation | High |
| C5 | Test code hardcodes an environment URL, account, password, secret, or token | High |

Use `git ls-files` and `git check-ignore` for C2.

**D Style**

| Id | Check | Severity |
|---|---|---|
| D1 | Test code does not compile | High |
| D2 | Style does not match the module: a Java 8 module uses a banned syntax or API; `javax` and `jakarta` are mixed; the JUnit version disagrees with the profile | Medium |
| D3 | Fake annotations disagree with Spring Boot: 3.4+ still uses `@MockBean` / `@SpyBean`, or below 3.4 uses `@MockitoBean` | Low |
| D4 | A mock test calls a real dependency: a `*FlowTest` call path goes through a type in `boundaries` and that type is not replaced | Medium |
| D5 | Startup args are hardcoded (`@SpringBootTest(properties = ...)`, or Apollo / Eureka switches inside `@TestPropertySource`) | Low |
| D6 | Class name or location disagrees with the style file | Low |
| D7 | A layer whose `boot` value is `daily` is missing the daily startup condition, so the default `mvn test` fails on config or a port conflict | High |

**E Usefulness**

| Id | Check | Severity |
|---|---|---|
| E1 | A test method has no assertion: no `andExpect`, `assertThat`, `assertEquals`, `verify`, or similar | Medium |
| E2 | A case whose expectation source is a snapshot has no user-confirmation record | Medium |
| E3 | A test is skipped with `@Disabled` / `@Ignore` and no reason is written | Low |

Severity:

- **High**: leaks data, calls a daily write or a billed endpoint by mistake, or the test cannot run.
- **Medium**: the result can mislead, or coverage has a gap.
- **Low**: style or maintenance.

### 5. Report

```markdown
## API test audit

- Scope: {{current change / named / all}}; endpoints {{n}}, test classes {{n}}, samples {{n}}
- Profile: {{read / missing, skipped checks: ...}}
- Test compile: {{pass / fail}}

### Summary

| Severity | Count |
|---|---|
| High | {{n}} |
| Medium | {{n}} |
| Low | {{n}} |

### Findings

| Severity | Id | Location | Detail | Suggestion |
|---|---|---|---|---|
| {{High / Medium / Low}} | {{A1}} | {{file:line or path}} | {{problem}} | {{for example: run /api-test-gen}} |

### Coverage

| Endpoint | Mock tests | Real tests | Behavior list |
|---|---|---|---|
| {{HTTP method path}} | {{n}} | {{n}} | {{yes / no}} |
```

Sort the findings table from high to low. When one id has more than 10 rows, list the first 10 and write the total.
