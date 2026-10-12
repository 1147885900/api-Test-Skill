# Style: Java 17 / 21 + Spring Boot 3.x

For modules with `styleGuide: java21-boot3`: compile target 17 or 21, Spring Boot 3.x, `jakarta.*`, JUnit 5, Mockito 5.

## Version differences

| Spring Boot | Fake annotations | Package |
|---|---|---|
| 3.4 and above | `@MockitoBean`, `@MockitoSpyBean` | `org.springframework.test.context.bean.override.mockito` |
| 3.0 through 3.3 | `@MockBean`, `@SpyBean` | `org.springframework.boot.test.mock.mockito` |

Spring Boot 4.x moved `@WebMvcTest` and `@AutoConfigureMockMvc` into their own modules. Use the package of the class that actually exists in the dependencies (search the jars for `WebMvcTest.class`). If the matching test starter is missing, list it under "Test dependencies to add" in the behavior list. Do not add it yourself.

## Names and location

| Case | Class name | Location |
|---|---|---|
| Contract mock | `<Controller>ApiTest` | `<basePackage>.apitest` |
| Boundary mock | `<Controller>FlowTest` | `<basePackage>.apitest` |
| Real test | `<Controller>IT` | `<basePackage>.apitest` |
| Support | `TestFixtures`, `FixtureRecorder`, `BoundarySpring` | `<basePackage>.apitest` |

- Method name: `<controller method>_<condition>_<result>`, for example `recognize_missingTextContent_rejected`. If existing tests in the module have a fixed style, follow that style.
- Classes and methods are package-private. Do not add `public`.
- Assert with AssertJ and MockMvc `jsonPath`.
- `var`, `List.of`, and text blocks are allowed. Do not use them when they disagree with existing tests in the module.

## Rules

- Do not put startup args in test code (no Apollo switches in `properties` or `@TestPropertySource`). `api-test-run` passes the startup mode from `boot` on the command line.
- `@WebMvcTest` loads project `Filter`, `WebMvcConfigurer`, and `@ControllerAdvice` beans. Replace the beans they depend on, or the context will not start. Add a missing fake from the startup error.
- When the global exception handler lives in a framework jar and `@WebMvcTest` did not load it, add `@Import({{exceptionHandler}}.class)`. The class name comes from `exceptionHandler` in the profile.
- When `sliceConfig` is `explicit`, extra scan annotations on the application class are not filtered by `@WebMvcTest`, so Service, Mapper, and Feign beans all load. The contract layer uses `@ContextConfiguration` to name the classes to load. See the note after template 1. `@Import` is not needed in that case.
- A layer whose `boot` value is `daily` gets the daily startup condition. See "Daily startup condition".
- Rejection assertions come from `rejectResponse`. When it is `unknown`, add `.andDo(print())` on the first rejection case, show the actual response to the user, write the confirmed assertion into the profile, then remove `print()`.
- `FlowTest` classes in one module share a context. See "Shared boundary context".
- Static methods (for example `RedisUtil`) use `mockStatic`. It applies to the current thread only. MockMvc runs the request on the test thread, so it works there. A static call on an async business thread is not affected. Put that branch under "Not generated".
- Code that uses `OkHttpClient` or another HTTP client directly: prefer to replace at an existing seam (for example hand the response bytes to the parse method). If the HTTP layer itself must be replaced, use a local fake server (OkHttp `MockWebServer`). That needs a new test dependency and confirmation. Do not stub the `OkHttpClient` call chain.

## Template 1: contract mock

```java
package {{basePackage}}.apitest;

import {{controllerPackage}}.{{Controller}};
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(controllers = {{Controller}}.class)
class {{Controller}}ApiTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private {{DependencyType}} {{dependencyField}};

    @Test
    void {{method}}_ok_{{result}}() throws Exception {
        when({{dependencyField}}.{{dependencyMethod}}(any()))
                .thenReturn(TestFixtures.readJson("{{dependencyKey}}/{{dependencyMethod}}-ok.json", {{ReturnType}}.class));

        mockMvc.perform(post("{{path}}")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(TestFixtures.read("request/{{controllerKey}}/{{method}}-ok.json")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("{{jsonPath}}").value({{expected}}));
    }

    @Test
    void {{method}}_missing{{Field}}_rejected() throws Exception {
        mockMvc.perform(post("{{path}}")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{{request body missing only the target field, for example {\"appId\":\"100\"}}}"))
                .andExpect({{rejectResponse}});

        verify({{dependencyField}}, never()).{{dependencyMethod}}(any());
    }
}
```

Every field the Controller injects needs a matching `@MockitoBean`. Change `post` in the template to `get`, `put`, or the real HTTP method. A GET request uses `.param("{{name}}", "{{value}}")` and has no body.

When `sliceConfig` is `explicit`, change the class annotations to:

```java
@WebMvcTest(controllers = {{Controller}}.class)
@ContextConfiguration(classes = { {{Controller}}.class, {{exceptionHandler}}.class })
class {{Controller}}ApiTest {
```

Imports: `org.springframework.test.context.ContextConfiguration`, and the fully qualified name of `{{exceptionHandler}}`. If endpoint behavior depends on a project `Filter` or interceptor (for example auth), add those classes to `classes` too.

## Shared boundary context

Most of a `FlowTest`'s time is Spring startup. The first start in a module takes tens of seconds. Later cases that reuse that context spend only the assertion time.

Spring caches a context by "startup mode + which bean types were replaced". The same key is reused inside one Maven process. Each `mvn test` is a new process, so two `FlowTest` classes launched separately each pay startup once.

Do these three things in one module:

1. Every `FlowTest` extends `BoundarySpring` in the same package (`<basePackage>.apitest`). `@SpringBootTest`, `@AutoConfigureMockMvc`, the daily startup condition, and every `@MockitoBean` are written only on that base class.
2. Subclasses do not declare `@MockitoBean` / `@MockitoSpyBean`, do not use `@DirtiesContext`, and do not set `properties` or `@TestPropertySource`. When a new case needs a type that is not replaced yet, add it on the base class so every existing `FlowTest` in the module moves to the new key together. One class must not keep its own key.
3. `api-test-run` puts the module's `FlowTest` classes in one `mvn` invocation. `mockStatic` still opens and closes in each case's `@BeforeEach` / `@AfterEach`. That does not affect the context cache.

```java
package {{basePackage}}.apitest;

@SpringBootTest
@AutoConfigureMockMvc
@EnabledIfSystemProperty(named = "env", matches = "daily", disabledReason = "context needs daily args; skipped by the default mvn test")
public class BoundarySpring {

    @Autowired
    protected MockMvc mockMvc;

    @MockitoBean
    protected {{ProxyType}} {{proxyField}};

    @MockitoBean
    protected {{MapperType}} {{mapperField}};
}
```

A subclass keeps only its test methods and its own `@BeforeEach`. When the base class already exists, append the missing `@MockitoBean`. Do not create a second base class.

## Template 2: boundary mock

```java
package {{basePackage}}.apitest;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import org.springframework.http.MediaType;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

class {{Controller}}FlowTest extends BoundarySpring {

    @Test
    void {{method}}_{{condition}}_{{result}}() throws Exception {
        when({{proxyField}}.{{proxyMethod}}({{argumentMatchers}}))
                .thenReturn(TestFixtures.readJson("{{dependencyKey}}/{{proxyMethod}}-{{scene}}.json", {{ReturnType}}.class));

        mockMvc.perform(post("{{path}}")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(TestFixtures.read("request/{{controllerKey}}/{{method}}-{{scene}}.json")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("{{jsonPath}}").value({{expected}}));

        ArgumentCaptor<{{EntityType}}> captor = ArgumentCaptor.forClass({{EntityType}}.class);
        verify({{mapperField}}).{{writeMethod}}(captor.capture());
        assertThat(captor.getValue().{{getter}}()).isEqualTo({{expected}});
    }
}
```

Replacing a static method:

```java
try (MockedStatic<{{StaticUtil}}> mocked = mockStatic({{StaticUtil}}.class)) {
    mocked.when(() -> {{StaticUtil}}.{{method}}({{args}})).thenReturn({{value}});

    mockMvc.perform(...)
            .andExpect(...);

    mocked.verify(() -> {{StaticUtil}}.{{writeMethod}}({{args}}));
}
```

Imports: `org.mockito.MockedStatic`, and a static import of `org.mockito.Mockito.mockStatic`.

When several cases replace the same static class, call `mockStatic` in `@BeforeEach` and `close()` in `@AfterEach`. Do not open it in a field initializer.

## Daily startup condition

A layer whose `boot` value is `daily` is skipped as a whole class when the default `mvn test` has no daily args, and it runs when those args are present.

`@EnabledIfSystemProperty` is not `@Inherited`. Put it on `BoundarySpring` and on each `FlowTest` subclass. Take the property name and value from the env marker in `commands.dailyArgs`. For example `-Denv=daily` becomes `named = "env", matches = "daily"`.

- Import: `org.junit.jupiter.api.condition.EnabledIfSystemProperty`.
- Real tests (`*IT`) are already outside the default `mvn test`. They do not need this annotation.

## Template 3: real test

```java
package {{basePackage}}.apitest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.doAnswer;
import static org.mockito.Mockito.doThrow;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

/**
 * Runs against daily. The default mvn test does not run this class. api-test-run daily mode does.
 */
@SpringBootTest
@AutoConfigureMockMvc
class {{Controller}}IT {

    @Autowired
    private MockMvc mockMvc;

    @MockitoSpyBean
    private {{RecordedType}} {{recordedField}};

    @MockitoSpyBean
    private {{ForbiddenType}} {{forbiddenField}};

    @BeforeEach
    void recordAndGuard() {
        doAnswer(invocation -> FixtureRecorder.record("{{dependencyKey}}", "{{recordedMethod}}", invocation.callRealMethod()))
                .when({{recordedField}}).{{recordedMethod}}({{argumentMatchers}});
        doThrow(new AssertionError("real test must not call {{ForbiddenType}}#{{forbiddenMethod}}"))
                .when({{forbiddenField}}).{{forbiddenMethod}}(any());
    }

    @Test
    void {{method}}_{{condition}}_{{result}}() throws Exception {
        mockMvc.perform(post("{{path}}")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(TestFixtures.read("request/{{controllerKey}}/{{method}}-{{scene}}.json")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("{{key field}}").exists());
    }
}
```

- Spy, with `@MockitoSpyBean`, every dependency that must be recorded and every method on the call path that appears in `dailyRun.forbiddenCalls`. When there is no forbidden call, delete that field and the `doThrow`.
- Record on a project wrapper (for example `*Proxy`), not on a Feign interface. A Feign interface bean is a JDK proxy and the spy may not apply. When business code injects the Feign interface directly, spy the project class that calls it.
- For a `void` method that writes the result into an argument (for example `HttpServletResponse`), `doAnswer` calls `invocation.callRealMethod()` first, records the written argument, then `return null`.
- Pick matchers by parameter type. Primitives use `anyLong()`, `anyBoolean()`, and similar. `any()` is only for object parameters. Using it on a primitive throws a null-pointer while stubbing.
- Assert structure and key invariants only (success, a key field exists, a value is inside the input set). Daily data changes and model output is unstable. Do not assert the full body.
