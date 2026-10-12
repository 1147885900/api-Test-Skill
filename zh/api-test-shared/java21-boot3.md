# 写法：Java 17 / 21 + Spring Boot 3.x

适用于画像 `styleGuide: java21-boot3` 的模块：编译目标 17 或 21、Spring Boot 3.x、`jakarta.*`、JUnit 5、Mockito 5。

## 版本差异

| Spring Boot | 替身注解 | 包名 |
|---|---|---|
| 3.4 及以上 | `@MockitoBean`、`@MockitoSpyBean` | `org.springframework.test.context.bean.override.mockito` |
| 3.0 到 3.3 | `@MockBean`、`@SpyBean` | `org.springframework.boot.test.mock.mockito` |

Spring Boot 4.x：`@WebMvcTest`、`@AutoConfigureMockMvc` 已移到独立模块，包名以依赖中实际存在的类为准（在依赖 jar 中查找 `WebMvcTest.class`）。缺少对应的 test starter 时，列入行为清单的“需要新增的测试依赖”，不自行添加。

## 命名与位置

| 用例 | 类名 | 位置 |
|---|---|---|
| 契约层替身用例 | `<Controller>ApiTest` | `<basePackage>.apitest` |
| 边界层替身用例 | `<Controller>FlowTest` | `<basePackage>.apitest` |
| 真实用例 | `<Controller>IT` | `<basePackage>.apitest` |
| 支持类 | `TestFixtures`、`FixtureRecorder`、`BoundarySpring` | `<basePackage>.apitest` |

- 方法名：`<Controller 方法>_<条件>_<结果>`，如 `recognize_missingTextContent_rejected`。模块现有测试有固定风格时，以现有风格为准。
- 类和方法使用包级可见性，不加 `public`。
- 断言用 AssertJ 和 MockMvc 的 `jsonPath`。
- 可以用 `var`、`List.of`、文本块；与模块现有测试风格不一致时不用。

## 规则

- 测试代码里不写启动参数（`properties`、`@TestPropertySource` 中不放 Apollo 等开关）。启动方式由 `api-test-run` 按画像 `boot` 在命令行传入。
- `@WebMvcTest` 会加载项目里的 `Filter`、`WebMvcConfigurer`、`@ControllerAdvice`；它们依赖的 Bean 也要替换，否则上下文起不来。缺哪个按启动报错补。
- 全局异常处理在框架 jar 中且 `@WebMvcTest` 未加载时，加 `@Import({{exceptionHandler}}.class)`；类名取画像 `exceptionHandler`。
- 画像 `sliceConfig: explicit` 时，启动类上额外的扫描注解不受 `@WebMvcTest` 过滤，会把 Service、Mapper、Feign 全部加载进来。契约层改用 `@ContextConfiguration` 显式指定要加载的类，见模板一后的说明；此时不再需要 `@Import`。
- 画像 `boot` 中为 `daily` 的层，测试类加 daily 启动条件，见“daily 启动条件”。
- 拒绝类用例的断言取画像 `rejectResponse`；为 `unknown` 时，第一条拒绝用例先加 `.andDo(print())` 运行，把实际返回给用户确认后写入画像，再去掉 `print()`。
- 同一模块的 `FlowTest` 共用上下文，见“边界层共用上下文”。
- 静态方法（如 `RedisUtil`）用 `mockStatic`。它只对当前线程生效：MockMvc 在测试线程内执行请求，可以生效；业务里异步线程中的静态调用不受影响，这类分支写进“不生成”。
- 直接使用 `OkHttpClient` 等 HTTP 客户端的代码：优先在已有分层处替换（如把响应字节直接交给解析方法）。必须在 HTTP 层替换时用本地假服务器（OkHttp `MockWebServer`），需要新增测试依赖并经确认。不要替换 `OkHttpClient` 的调用链。

## 模板一：契约层替身用例

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
                        .content("{{只缺目标字段的请求体，如 {\"appId\":\"100\"}}}"))
                .andExpect({{rejectResponse}});

        verify({{dependencyField}}, never()).{{dependencyMethod}}(any());
    }
}
```

Controller 的每个注入字段都要有对应的 `@MockitoBean`。模板中的 `post` 按接口实际方法换成 `get`、`put` 等；GET 请求用 `.param("{{name}}", "{{value}}")` 传参，不设请求体。

画像 `sliceConfig: explicit` 时，类上改为：

```java
@WebMvcTest(controllers = {{Controller}}.class)
@ContextConfiguration(classes = { {{Controller}}.class, {{exceptionHandler}}.class })
class {{Controller}}ApiTest {
```

需要的 import：`org.springframework.test.context.ContextConfiguration`，以及 `{{exceptionHandler}}` 的全名。接口行为依赖项目自己的 `Filter`、拦截器（如鉴权）时，把它们也加进 `classes`。

## 边界层共用上下文

`FlowTest` 的耗时在 Spring 启动。一个模块第一次起来大约几十秒，后面的用例若能复用这次上下文，只花断言本身的时间。

Spring 按「启动方式 + 替换了哪些 Bean 类型」缓存上下文，同一 Maven 进程里键相同就复用。每个 `mvn test` 都是新进程，分开跑两个 `FlowTest` 类会各启动一次。

同一模块做到这三件：

1. 所有 `FlowTest` 继承同包的 `BoundarySpring`（`<basePackage>.apitest`）。`@SpringBootTest`、`@AutoConfigureMockMvc`、daily 启动条件、全部 `@MockitoBean` 只写在这个基类上。
2. 子类不再声明 `@MockitoBean` / `@MockitoSpyBean`，不用 `@DirtiesContext`，不写 `properties` 或 `@TestPropertySource`。新用例要替换的类型还没有时，加到基类上，让本模块已有的 `FlowTest` 一起改用新键，避免一个类一把钥匙。
3. `api-test-run` 把本模块要跑的 `FlowTest` 放在同一次 `mvn` 里。`mockStatic` 仍在每个用例的 `@BeforeEach` / `@AfterEach` 里开关，不影响上下文缓存。

```java
package {{basePackage}}.apitest;

@SpringBootTest
@AutoConfigureMockMvc
@EnabledIfSystemProperty(named = "{{参数名}}", matches = "{{参数值}}", disabledReason = "需要画像里的本地启动参数，默认 mvn test 跳过")
public class BoundarySpring {

    @Autowired
    protected MockMvc mockMvc;

    @MockitoBean
    protected {{ProxyType}} {{proxyField}};

    @MockitoBean
    protected {{MapperType}} {{mapperField}};
}
```

子类只保留用例方法和本类的 `@BeforeEach`。基类已存在时追加缺少的 `@MockitoBean`，不另建第二个基类。

## 模板二：边界层替身用例

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

替换静态方法：

```java
try (MockedStatic<{{StaticUtil}}> mocked = mockStatic({{StaticUtil}}.class)) {
    mocked.when(() -> {{StaticUtil}}.{{method}}({{args}})).thenReturn({{value}});

    mockMvc.perform(...)
            .andExpect(...);

    mocked.verify(() -> {{StaticUtil}}.{{writeMethod}}({{args}}));
}
```

需要的 import：`org.mockito.MockedStatic`，静态导入 `org.mockito.Mockito.mockStatic`。

多个用例都要替换同一个静态类时，在 `@BeforeEach` 中 `mockStatic`，在 `@AfterEach` 中 `close()`，不要放在字段初始化里。

## daily 启动条件

画像 `boot` 中为 `daily` 的层，默认 `mvn test` 不带 daily 参数时整个类显示为跳过，带参数时正常运行。

这个注解放在 `BoundarySpring` 上，不放在每个 `FlowTest` 子类上。只有 `commands.dailyArgs` 里确有一项用来区分环境时才加。参数名和值取用户配置的那一项，例如用户写了 `-Denv=test`，才是 `named = "env", matches = "test"`。`dailyArgs` 为空时不要加这个注解，也不要写成 `env=daily`。

- import：`org.junit.jupiter.api.condition.EnabledIfSystemProperty`。
- 真实用例（`*IT`）不进入默认 `mvn test`，不需要加。

## 模板三：真实用例

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
 * 连 daily 运行，默认 mvn test 不执行。由 api-test-run 的 daily 模式运行。
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
        doThrow(new AssertionError("真实用例禁止调用 {{ForbiddenType}}#{{forbiddenMethod}}"))
                .when({{forbiddenField}}).{{forbiddenMethod}}(any());
    }

    @Test
    void {{method}}_{{condition}}_{{result}}() throws Exception {
        mockMvc.perform(post("{{path}}")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(TestFixtures.read("request/{{controllerKey}}/{{method}}-{{scene}}.json")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("{{关键字段}}").exists());
    }
}
```

- 需要录制的依赖和画像 `dailyRun.forbiddenCalls` 中调用链会经过的方法，都用 `@MockitoSpyBean` 包住；没有禁止调用时删掉对应字段和 `doThrow`。
- 录制点选项目自己的包装类（如 `*Proxy`），不直接包 Feign 接口：Feign 接口的 Bean 是 JDK 动态代理，spy 可能不生效。业务代码直接注入 Feign 接口时，spy 调用它的项目内类。
- 返回值为 `void`、结果写进入参的方法（如写 `HttpServletResponse`），`doAnswer` 中先 `invocation.callRealMethod()`，再从入参取内容录制，最后 `return null`。
- 参数匹配器按参数类型选：基本类型用 `anyLong()`、`anyBoolean()` 等，`any()` 只用于对象参数，否则打桩时会空指针。
- 断言只写结构和关键不变量（如返回成功、关键字段存在、取值属于输入范围）。daily 数据会变、大模型输出不稳定，不断言完整内容。
