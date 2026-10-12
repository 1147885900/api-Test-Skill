---
name: api-test-run
description: Run Java/Spring Boot Maven API tests and classify failures. Mock mode runs ApiTest and FlowTest. Real mode runs IT with the local arguments from the profile. The user configures those arguments, and the list may be empty. Switches JDK per module. Report only; does not edit code. Use when the user asks to run API tests, run IT, or verify a change.
disable-model-invocation: true
---

# API test run

Build the Maven command from the profile, run the tests, and classify failures. Terms (profile, mock test, real test, sample) match [api-test-gen](../api-test-gen/SKILL.md).

## Hard rules

1. Report only. Do not edit test code, business code, or `pom.xml`. Write a profile change only after the user confirms the suggestion.
2. On Windows PowerShell, every `-D` argument must be wrapped in double quotes, for example `"-Dsome.flag=value"`. Without quotes PowerShell splits on `.` and Maven reports `Unknown lifecycle phase`. The comma in `-Dtest=A,B` needs quotes too.
3. Mock mode does not pass real-environment args unless that layer's `boot` value is `daily`. Real tests (`*IT`) use only `commands.dailyArgs` and `dailyRun.extraArgs`. When both are empty, add no `-D` arguments. Do not invent `-Denv=daily` or any argument that is not in the profile. The profile name daily means "run against the real environment". It does not mean the environment is named daily.
4. Use only the test environment the user named. If a startup arg is `-Denv=prod`, `-Denv=pro`, or another production marker, stop and tell the user.
5. Recordings are not redacted. Do not paste their contents into the chat, and do not copy them elsewhere.

## Mode and scope

| Mode | What runs | Startup args |
|---|---|---|
| Mock (default) | `*ApiTest`, `*FlowTest`, and other existing tests in scope | From `boot.contract` / `boot.boundary`: `offline` uses `boot.offlineArgs`; `daily` uses `commands.dailyArgs` plus `dailyRun.extraArgs` |
| daily | `*IT` | `commands.dailyArgs` plus `dailyRun.extraArgs` |
| all | Mock first, then daily | Same as each mode above |

| Scope | How to pick classes |
|---|---|
| Current change (default) | Test files in `git diff --name-only <baseBranch>...HEAD` and `git status --porcelain`. For a changed main class, find tests by name (`<Class>ApiTest` / `FlowTest` / `IT` / `Test`) and by references to the class name under the test tree |
| Named | The class or method the user gave. A method is `ClassName#methodName` |
| Module | Every test in that module |
| All | Every module in the profile |

## Workflow

```
- [ ] 1. Read the profile; choose mode, scope, and classes
- [ ] 2. Choose the JDK
- [ ] 3. Probe startup mode (when boot is unknown)
- [ ] 4. Build the command and run
- [ ] 5. Read results and classify failures
- [ ] 6. Rerun failed cases
- [ ] 7. Collect recordings (daily mode)
- [ ] 8. Report
```

### 1. Read the profile

Read `<project root>/.api-test/profile.yaml`. If it is missing, tell the user to run `/api-test-gen` first. If they only want existing tests, run `mvn -pl <module> test` and skip step 3.

Before real mode, list the real tests that will run. Cases marked as a write or a billed call in `.api-test/behaviors/` need a reminder and a confirmation before they run.

When `commands.dailyArgs` and `dailyRun.extraArgs` are both empty, and the user has not already said that no extra arguments are needed, ask which `-D` arguments the local real run should add. If they need none, leave both lists empty and remember that. Do not ask again, and do not fill in example arguments.

### 2. Choose the JDK

Maven compiles and runs tests with the JDK at `JAVA_HOME`. It must be at least the module compile target (`modules[].java`).

Local JDK paths live in `~/.cursor/api-test-local.yaml` (a machine file, not part of the project):

```yaml
# Local JDK paths. api-test-run switches by module.
jdks:
  "8": "{{JDK 8 home, for example C:/Program Files/Java/jdk1.8.0_202}}"
  "17": "{{JDK 17 home; delete this line if absent}}"
  "21": "{{JDK 21 home}}"
```

When the file is missing or lacks the needed version:

1. Collect candidate directories, in order:
   - The current `JAVA_HOME`, and directories from `where java` (bash: `which -a java`).
   - Windows registry: `JavaHome` under `HKLM\SOFTWARE\JavaSoft\JDK\*` and `HKLM\SOFTWARE\JavaSoft\Java Development Kit\*`.
   - `jdkHome` in the Maven toolchain file `~/.m2/toolchains.xml`.
   - Common install locations: `C:/Program Files/Java/`, `C:/Program Files/Eclipse Adoptium/`, `C:/Program Files/Zulu/`, `C:/Program Files/Amazon Corretto/`, `C:/Program Files/Microsoft/`, `~/.jdks/`, `/Library/Java/JavaVirtualMachines/`, `/usr/lib/jvm/`.
   - Custom locations: the parent of `JAVA_HOME` (for example `D:/software/jdk/`), and `software/jdk`, `tools/jdk`, or `java` directories on each drive.
2. Run `<dir>/bin/java -version` on each candidate.
3. Tell the user the result and ask whether to write the file. If they decline, use the path only in this chat. If nothing is found, ask for a path.

Set the JDK on each command. Do not rely on the environment left by the previous command:

```powershell
$env:JAVA_HOME = '{{JDK home}}'; $env:Path = "$env:JAVA_HOME\bin;$env:Path"; {{mvn command}}
```

On bash: `JAVA_HOME='{{JDK home}}' PATH="$JAVA_HOME/bin:$PATH" {{mvn command}}`.

### 3. Probe startup mode

When `boot.contract` or `boot.boundary` is `unknown` and that layer has tests in scope:

1. Run one test class of that layer with `boot.offlineArgs`. If the context starts (whether or not assertions pass), record `offline`.
2. If startup fails because config cannot be loaded or a connection fails, run again with `commands.dailyArgs` plus `dailyRun.extraArgs`. If it starts, record `daily`. A missing switch in the config center counts. A common sign is `BeanDefinitionOverrideException` (missing `spring.main.allow-bean-definition-overriding`) or an unresolved placeholder.
3. If both fail, report the last `Caused by` and leave the value `unknown`.
4. Offer the probe result as a profile change. Write it after the user confirms.

If `-D` args never reach the test JVM (offline args have no effect, and the log still connects to Apollo), say so in the report and suggest `"-DargLine=..."`. `argLine` replaces an existing `argLine` in the pom (for example a coverage agent), so the user must confirm.

### 4. Build the command and run

Start from `commands.testClasses` and append startup args. Quote each `-D` argument:

```powershell
mvn -pl {{module}} "-Dtest={{class1}},{{class2}}" "-Dsurefire.failIfNoSpecifiedTests=false" {{startup args}} test
```

Example (real mode, no extra arguments in the profile):

```powershell
mvn -pl <module> -am "-Dtest=<Controller>IT" "-Dsurefire.failIfNoSpecifiedTests=false" test
```

When `commands.dailyArgs` or `dailyRun.extraArgs` has entries, insert each one as its own quoted argument before `test`. When both are empty, the command ends at `test`.

- When the contract layer and the boundary layer use different startup modes, run them in separate groups. Put `FlowTest` classes of one group in the same `mvn` so the Spring context is reused in that process. A separate `mvn` per class is a new process and pays startup again.
- `-Dtest` overrides Surefire's default includes, so an `*IT` named there will run. Mock mode must not put `*IT` in `-Dtest`.
- If the output says `Tests are skipped`, add `"-DskipTests=false"`. If test code was not compiled, add `"-Dmaven.test.skip=false"`.
- If the error is `Could not resolve dependencies` and the missing artifacts are other modules of this project, add `-am`.
- Do not add `-q`. For a long run, start it in the background and check the output.

### 5. Read results and classify failures

Read the summary line `Tests run: x, Failures: y, Errors: z, Skipped: w`, then the failed class `.txt` and `TEST-*.xml` under `<module>/target/surefire-reports/`. Use the last `Caused by` as the root cause.

| Category | How to recognize it | Next step |
|---|---|---|
| Wrong JDK | `invalid target release`, `release version 21 not supported`, `UnsupportedClassVersionError` | Recheck the JDK from step 2 |
| Compile failure | `COMPILATION ERROR`, `cannot find symbol` | List file and line. A changed method signature shows up here |
| Context failed to start | `Failed to load ApplicationContext`, `NoSuchBeanDefinitionException`, `UnsatisfiedDependencyException`, `BeanDefinitionOverrideException` | A missing bean means a fake is missing. An unresolved property or a duplicate bean means the startup mode is wrong. If `@WebMvcTest` loads many unrelated beans, check `sliceConfig` |
| Port in use | `port[...] is used`, `Address already in use`, `BindException` | A local process holds the port, often a job executor. Check `dailyRun.extraArgs`, or whether the layer is missing the daily startup condition |
| Environment down | `Connection refused`, `UnknownHostException`, `connect timed out`, `ApolloConfigException`, `RedisConnectionFailureException`, `CannotGetJdbcConnectionException` | Daily mode: check VPN and the daily service. Mock mode: a boundary was not replaced |
| Guard tripped | `AssertionError: real test must not call` | The call path hit a forbidden method. Check the request data, or change the case to a mock |
| Assertion failed | `AssertionError`, `Status expected`, `JSON path`, `expected:` | Suspected defect or an unconfirmed expectation. Do not change the assertion |
| Not executed | `Tests run: 0`, `No tests matching pattern`, `Tests are skipped` | Check `-Dtest` and skip flags |
| All skipped | `Skipped` equals `Tests run`, and the class has the daily startup condition | Daily args were not passed. If the profile says this layer is `daily`, rerun with those args |

### 6. Rerun failed cases

For assertion failures and environment failures, rerun with `"-Dtest=ClassName#methodName"` at most 2 more times:

- All reruns pass, or the result flips: mark "flaky". The case may depend on time, order, or shared state.
- Every rerun fails: keep the original category.

Do not rerun compile failures, wrong JDK, or context startup failures.

### 7. Collect recordings

After daily mode, list files under `<module>/target/api-test-samples/` and count them by dependency name. List names and counts only. Do not show contents. Tell the user the samples are not redacted. `/api-test-gen` step 5 must redact them and show them to the user before they can be written to `src/test/resources/fixtures/`.

### 8. Report

```markdown
## API test run

- Mode: {{mock / daily / all}}; scope: {{current change / named / module / all}}
- JDK: {{version and directory}}
- Command: `{{the command that ran, one line per group}}`

### Summary

| Module | Run | Passed | Failed | Skipped | Flaky |
|---|---|---|---|---|---|
| {{module}} | {{n}} | {{n}} | {{n}} | {{n}} | {{n}} |

### Failures

| Case | Category | Root cause | Next step |
|---|---|---|---|
| {{Class#method}} | {{category}} | {{one-line summary of the last Caused by}} | {{suggestion}} |

### Recordings

- {{dependency}}: {{n}} files at `{{target/api-test-samples/...}}`

### Profile suggestions

- {{for example: boot.contract: unknown → offline (AddressControllerApiTest started with offline args)}}
```
