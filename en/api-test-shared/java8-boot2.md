# Style: Java 8 + Spring Boot 2.x

For modules with `styleGuide: java8-boot2`: compile target 8, Spring Boot 2.x, `javax.*`. JUnit follows existing tests in the module. When there are no tests, follow the dependencies: `junit-jupiter` means JUnit 5, and only `junit:junit` means JUnit 4.

Layers, names, locations, and usage rules match [java21-boot3.md](java21-boot3.md). This file lists only the differences.

## Banned syntax and APIs

After generating, check every row. Rewrite a hit.

| Banned | Rewrite as |
|---|---|
| `var` | The concrete type |
| `List.of`, `Set.of`, `Map.of` | `Arrays.asList`, `Collections.singletonList`, `new HashMap<>()` then `put` |
| Text block `"""` | A sample file, or string concatenation |
| `record`, switch expressions, `instanceof` pattern matching | A normal class, a `switch` statement, a cast |
| Diamond on an anonymous class `new TypeReference<>() {}` | Full type arguments `new TypeReference<List<XxxDTO>>() {}` |
| `String.isBlank`, `strip`, `repeat`, `lines`, `formatted` | `trim().isEmpty()`, a loop, `String.format` |
| `Optional.isEmpty`, no-arg `orElseThrow()`, `ifPresentOrElse` | `!isPresent()`, `orElseThrow(IllegalStateException::new)`, an `if` |
| `Stream.toList()`, `Collectors.toUnmodifiableList()` | `collect(Collectors.toList())` |
| `InputStream.readAllBytes`, `Files.readString`, `Files.writeString` | A read loop, `Files.write(path, bytes)`. See [support-classes.md](support-classes.md) |
| An existing variable directly in try-with-resources `try (in)` | `try (InputStream in2 = in)` |

## Fake annotations and version differences

- Fake annotations: `@MockBean`, `@SpyBean`, package `org.springframework.boot.test.mock.mockito`.
- `mockStatic` needs Mockito 3.4 or newer, and `mockito-inline` on the classpath (Mockito 4 and below do not include it by default). If it is missing, put the static-call branch under "Not generated", or list it under "Test dependencies to add". Do not add the dependency yourself.
- From Spring Boot 2.4, `spring-boot-starter-test` no longer brings the JUnit 4 engine. If existing JUnit 4 tests in the module already run, the engine is already on the classpath. Keep using it.

## JUnit 5

Apply these replacements to the three templates in [java21-boot3.md](java21-boot3.md). Leave the rest unchanged:

| Where | Replace with |
|---|---|
| `@MockitoBean` | `@MockBean` |
| `@MockitoSpyBean` | `@SpyBean` |
| `import org.springframework.test.context.bean.override.mockito.MockitoBean;` | `import org.springframework.boot.test.mock.mockito.MockBean;` |
| `import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;` | `import org.springframework.boot.test.mock.mockito.SpyBean;` |

After replacing, check the banned table again.

The daily startup condition matches [java21-boot3.md](java21-boot3.md). `@EnabledIfSystemProperty` needs JUnit 5.1 or newer. The `disabledReason` attribute needs 5.7 or newer. Below 5.7, drop that attribute.

## JUnit 4

Differences from JUnit 5:

- Add `@RunWith(SpringRunner.class)` on the class.
- Test classes and test methods must be `public`.
- `org.junit.Test` replaces `org.junit.jupiter.api.Test`. `@Before` replaces `@BeforeEach`.
- A static-method fake is wrapped in try-with-resources inside the method. It cannot sit on a field.

### Contract mock (JUnit 4)

```java
package {{basePackage}}.apitest;

import {{controllerPackage}}.{{Controller}};
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@RunWith(SpringRunner.class)
@WebMvcTest(controllers = {{Controller}}.class)
public class {{Controller}}ApiTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private {{DependencyType}} {{dependencyField}};

    @Test
    public void {{method}}_ok_{{result}}() throws Exception {
        when({{dependencyField}}.{{dependencyMethod}}(any()))
                .thenReturn(TestFixtures.readJson("{{dependencyKey}}/{{dependencyMethod}}-ok.json", {{ReturnType}}.class));

        mockMvc.perform(post("{{path}}")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(TestFixtures.read("request/{{controllerKey}}/{{method}}-ok.json")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("{{jsonPath}}").value({{expected}}));
    }

    @Test
    public void {{method}}_missing{{Field}}_rejected() throws Exception {
        mockMvc.perform(post("{{path}}")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{{request body missing only the target field}}"))
                .andExpect({{rejectResponse}});

        verify({{dependencyField}}, never()).{{dependencyMethod}}(any());
    }
}
```

### Boundary mock and real test (JUnit 4)

Start from templates 2 and 3 in [java21-boot3.md](java21-boot3.md):

1. Add `@RunWith(SpringRunner.class)` on the class. Keep `@SpringBootTest` and `@AutoConfigureMockMvc`.
2. Make the class and the methods `public`.
3. `@MockitoBean` → `@MockBean`, `@MockitoSpyBean` → `@SpyBean`, `@BeforeEach` → `@Before`, and update the imports.
4. The daily startup condition becomes `@IfProfileValue(name = "{{daily property name}}", value = "{{daily property value}}")` on the class (`org.springframework.test.annotation.IfProfileValue`). `SpringRunner` reads system properties by default. When the value does not match, the whole class is ignored before the context starts.
5. Check the banned table.

## Other Spring Boot 2.x differences

- `@WebMvcTest` still loads `Filter`, `WebMvcConfigurer`, and `@ControllerAdvice`. Handle them the same way as [java21-boot3.md](java21-boot3.md).
- When `sliceConfig` is `explicit`, use `@ContextConfiguration(classes = {...})` the same way as the note after template 1 in [java21-boot3.md](java21-boot3.md). On JUnit 4, keep `@RunWith(SpringRunner.class)`.
- If a bean wrapped by `@SpyBean` is also an AOP proxy (for example `@Transactional`), stubbing can fail on Boot 2.x. Record the next dependency it calls instead.
- Shared boundary context matches [java21-boot3.md](java21-boot3.md). Replace annotations on the base class as this file says: `@MockBean`, plus `@RunWith(SpringRunner.class)` on JUnit 4, and `@IfProfileValue` for the daily startup condition. Subclasses still do not declare their own fakes.
