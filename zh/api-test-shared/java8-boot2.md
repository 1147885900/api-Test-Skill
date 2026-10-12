# 写法：Java 8 + Spring Boot 2.x

适用于画像 `styleGuide: java8-boot2` 的模块：编译目标 8、Spring Boot 2.x、`javax.*`。JUnit 版本以模块现有测试为准；没有测试时看依赖：有 `junit-jupiter` 用 JUnit 5，只有 `junit:junit` 用 JUnit 4。

用例分层、命名、位置和使用规则与 [java21-boot3.md](java21-boot3.md) 相同，本文只列不同之处。

## 禁用的语法和 API

生成后逐条自查，命中即改写：

| 禁用 | 改写为 |
|---|---|
| `var` | 写出具体类型 |
| `List.of`、`Set.of`、`Map.of` | `Arrays.asList`、`Collections.singletonList`、`new HashMap<>()` 后 `put` |
| 文本块 `"""` | 样本文件，或字符串拼接 |
| `record`、switch 表达式、`instanceof` 模式匹配 | 普通类、`switch` 语句、强制类型转换 |
| 匿名类使用菱形 `new TypeReference<>() {}` | 写全类型参数 `new TypeReference<List<XxxDTO>>() {}` |
| `String.isBlank`、`strip`、`repeat`、`lines`、`formatted` | `trim().isEmpty()`、循环拼接、`String.format` |
| `Optional.isEmpty`、无参 `orElseThrow()`、`ifPresentOrElse` | `!isPresent()`、`orElseThrow(IllegalStateException::new)`、`if` 判断 |
| `Stream.toList()`、`Collectors.toUnmodifiableList()` | `collect(Collectors.toList())` |
| `InputStream.readAllBytes`、`Files.readString`、`Files.writeString` | 循环读取、`Files.write(path, bytes)`；见 [support-classes.md](support-classes.md) |
| try-with-resources 中直接写已有变量 `try (in)` | `try (InputStream in2 = in)` |

## 替身注解与版本差异

- 替身注解：`@MockBean`、`@SpyBean`，包名 `org.springframework.boot.test.mock.mockito`。
- `mockStatic` 需要 Mockito 3.4 及以上，并且依赖中有 `mockito-inline`（Mockito 4 及以下默认不带）。缺少时把静态调用分支写进行为清单的“不生成”，或列入“需要新增的测试依赖”，不自行添加。
- Spring Boot 2.4 起 `spring-boot-starter-test` 不再带 JUnit 4 的运行引擎；模块现有 JUnit 4 测试能运行，说明已自行引入，照用即可。

## JUnit 5

在 [java21-boot3.md](java21-boot3.md) 的三个模板上做以下替换，其余不变：

| 位置 | 替换 |
|---|---|
| `@MockitoBean` | `@MockBean` |
| `@MockitoSpyBean` | `@SpyBean` |
| `import org.springframework.test.context.bean.override.mockito.MockitoBean;` | `import org.springframework.boot.test.mock.mockito.MockBean;` |
| `import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;` | `import org.springframework.boot.test.mock.mockito.SpyBean;` |

替换后再按“禁用的语法和 API”自查一遍。

daily 启动条件同 [java21-boot3.md](java21-boot3.md)：`@EnabledIfSystemProperty` 需要 JUnit 5.1 及以上，`disabledReason` 属性需要 5.7 及以上，低于 5.7 时去掉该属性。

## JUnit 4

与 JUnit 5 的区别：

- 类上加 `@RunWith(SpringRunner.class)`。
- 测试类和测试方法必须是 `public`。
- `org.junit.Test` 替代 `org.junit.jupiter.api.Test`，`@Before` 替代 `@BeforeEach`。
- 静态方法替身在方法内用 try-with-resources 包住，不能放在字段上。

### 契约层替身用例（JUnit 4）

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
                        .content("{{只缺目标字段的请求体}}"))
                .andExpect({{rejectResponse}});

        verify({{dependencyField}}, never()).{{dependencyMethod}}(any());
    }
}
```

### 边界层替身用例与真实用例（JUnit 4）

以 [java21-boot3.md](java21-boot3.md) 的模板二、模板三为基础：

1. 类上加 `@RunWith(SpringRunner.class)`，保留 `@SpringBootTest`、`@AutoConfigureMockMvc`。
2. 类和方法改为 `public`。
3. `@MockitoBean` → `@MockBean`，`@MockitoSpyBean` → `@SpyBean`，`@BeforeEach` → `@Before`，并修改对应 import。
4. daily 启动条件改用类上的 `@IfProfileValue(name = "{{daily 参数名}}", value = "{{daily 参数值}}")`（`org.springframework.test.annotation.IfProfileValue`）。`SpringRunner` 默认从系统属性取值，不满足时整个类在启动上下文之前就被忽略。
5. 按“禁用的语法和 API”自查。

## Spring Boot 2.x 的其他差异

- `@WebMvcTest` 同样加载 `Filter`、`WebMvcConfigurer`、`@ControllerAdvice`，处理方式同 [java21-boot3.md](java21-boot3.md)。
- 画像 `sliceConfig: explicit` 时同样用 `@ContextConfiguration(classes = {...})` 显式指定加载的类，写法同 [java21-boot3.md](java21-boot3.md) 模板一后的说明；JUnit 4 下保留 `@RunWith(SpringRunner.class)`。
- `@SpyBean` 包住的 Bean 若被 AOP 代理（如带 `@Transactional`），Boot 2.x 可能出现打桩不生效；遇到时改为录制它调用的下一层依赖。
- 边界层共用上下文同 [java21-boot3.md](java21-boot3.md)。基类上的注解按本文替换：`@MockBean`、JUnit 4 再加 `@RunWith(SpringRunner.class)`，daily 启动条件用 `@IfProfileValue`。子类同样不再单独声明替身。
