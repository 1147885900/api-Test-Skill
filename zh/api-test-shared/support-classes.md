# 测试支持类

每个模块一份，放在 `<模块>/src/test/java/<basePackage 路径>/apitest/`，与生成的测试类同包，不放进手写单测的业务包。模块里已存在同名类时不覆盖、不修改。

两个类都只用 Java 8 API，Java 8 和 Java 21 模块通用。JSON 库按画像 `modules[].json` 选择：先用下面的 fastjson2 完整版本，再按“其他 JSON 库”替换对应部分。

| 类 | 用途 |
|---|---|
| `TestFixtures` | 读取 `src/test/resources/fixtures/` 下的样本 |
| `FixtureRecorder` | 真实用例运行时录制下游返回值到 `target/api-test-samples/` |

## TestFixtures（fastjson2）

```java
package {{basePackage}}.apitest;

import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.TypeReference;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.UncheckedIOException;
import java.nio.charset.StandardCharsets;

/**
 * 读取 src/test/resources/fixtures/ 下的样本文件，path 为相对 fixtures/ 的路径。
 */
public final class TestFixtures {

    private static final String ROOT = "fixtures/";

    private TestFixtures() {
    }

    public static String read(String path) {
        String resource = ROOT + path;
        try (InputStream in = TestFixtures.class.getClassLoader().getResourceAsStream(resource)) {
            if (in == null) {
                throw new IllegalStateException("样本不存在: " + resource);
            }
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            byte[] buffer = new byte[4096];
            int n;
            while ((n = in.read(buffer)) != -1) {
                out.write(buffer, 0, n);
            }
            return new String(out.toByteArray(), StandardCharsets.UTF_8);
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    public static <T> T readJson(String path, Class<T> type) {
        return JSON.parseObject(read(path), type);
    }

    public static <T> T readJson(String path, TypeReference<T> type) {
        return JSON.parseObject(read(path), type);
    }
}
```

泛型返回值（如 `List<XxxDTO>`）用 `TypeReference`：

```java
List<XxxDTO> items = TestFixtures.readJson("xxx/list-ok.json", new TypeReference<List<XxxDTO>>() {});
```

## FixtureRecorder（fastjson2）

```java
package {{basePackage}}.apitest;

import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.JSONWriter;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.time.LocalDate;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 真实用例录制下游返回值。输出未脱敏，只写到 target/api-test-samples/，禁止复制到 src 或提交。
 */
public final class FixtureRecorder {

    private static final Path ROOT = Paths.get("target", "api-test-samples");

    private static final AtomicInteger SEQUENCE = new AtomicInteger();

    private FixtureRecorder() {
    }

    public static <T> T record(String dependency, String method, T value) {
        int seq = SEQUENCE.incrementAndGet();
        Path dir = ROOT.resolve(dependency);
        String type = value == null ? "null" : value.getClass().getName();
        String json = JSON.toJSONString(value, JSONWriter.Feature.PrettyFormat, JSONWriter.Feature.WriteNulls);
        String meta = "source: recorded\n"
                + "type: " + type + "\n"
                + "env: daily\n"
                + "capturedAt: " + LocalDate.now() + "\n";
        try {
            Files.createDirectories(dir);
            Files.write(dir.resolve(method + "-" + seq + ".json"), json.getBytes(StandardCharsets.UTF_8));
            Files.write(dir.resolve(method + "-" + seq + ".meta.yaml"), meta.getBytes(StandardCharsets.UTF_8));
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
        return value;
    }
}
```

`target/` 相对 surefire 的工作目录，即模块根目录。返回值是泛型集合时，`type` 只记录到 `ArrayList` 这类实现类，转存时按方法签名补全 `.meta.yaml` 的 `type`。

## 其他 JSON 库

只替换 import 和标出的方法，其余代码不变。

### fastjson（1.x）

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.TypeReference;
import com.alibaba.fastjson.serializer.SerializerFeature;

// TestFixtures：readJson 两个方法体不变

// FixtureRecorder.record 中的序列化：
String json = JSON.toJSONString(value, SerializerFeature.PrettyFormat, SerializerFeature.WriteMapNullValue);
```

### Jackson 2.x

```java
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;

private static final ObjectMapper MAPPER = new ObjectMapper()
        .findAndRegisterModules()
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

// TestFixtures
public static <T> T readJson(String path, Class<T> type) {
    try {
        return MAPPER.readValue(read(path), type);
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}

public static <T> T readJson(String path, TypeReference<T> type) {
    try {
        return MAPPER.readValue(read(path), type);
    } catch (IOException e) {
        throw new UncheckedIOException(e);
    }
}

// FixtureRecorder.record 中的序列化（同一个 MAPPER 配置，放进 try 块内）：
String json = MAPPER.writerWithDefaultPrettyPrinter().writeValueAsString(value);
```

Spring Boot 4 默认使用 Jackson 3，包名为 `tools.jackson`，异常为非受检异常；按依赖中实际存在的类调整 import 并去掉 `try/catch`。
