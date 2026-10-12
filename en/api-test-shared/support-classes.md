# Test support classes

One copy per module, under `<module>/src/test/java/<basePackage path>/apitest/`, in the same package as the generated tests. Do not put them in the package used by handwritten tests. If a class of the same name already exists, do not overwrite it and do not edit it.

Both classes use only Java 8 APIs, so they work on Java 8 and Java 21 modules. Pick the JSON library from `modules[].json`: start from the fastjson2 version below, then replace the marked parts using "Other JSON libraries".

| Class | Role |
|---|---|
| `TestFixtures` | Reads samples under `src/test/resources/fixtures/` |
| `FixtureRecorder` | While a real test runs, records downstream return values into `target/api-test-samples/` |

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
 * Reads a sample under src/test/resources/fixtures/. path is relative to fixtures/.
 */
public final class TestFixtures {

    private static final String ROOT = "fixtures/";

    private TestFixtures() {
    }

    public static String read(String path) {
        String resource = ROOT + path;
        try (InputStream in = TestFixtures.class.getClassLoader().getResourceAsStream(resource)) {
            if (in == null) {
                throw new IllegalStateException("sample not found: " + resource);
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

A generic return value (for example `List<XxxDTO>`) uses `TypeReference`:

```java
List<XxxDTO> items = TestFixtures.readJson("xxx/list-ok.json", new TypeReference<List<XxxDTO>>() {});
```

## FixtureRecorder (fastjson2)

`{{envName}}` is the environment name from the profile. When the user added no extra startup arguments, write `local`, not `daily`.

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
 * Records a downstream return value from a real test. The output is not redacted. Write it only under target/api-test-samples/. Do not copy it into src and do not commit it.
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
                + "env: {{envName}}\n"
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

`target/` is relative to the Surefire working directory, which is the module root. When the return value is a generic collection, `type` is recorded only as the runtime class such as `ArrayList`. When copying the sample into `src`, fill `type` in `.meta.yaml` from the method signature.

## Other JSON libraries

Replace only the imports and the methods called out below. Leave the rest of the code unchanged.

### fastjson（1.x）

```java
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.TypeReference;
import com.alibaba.fastjson.serializer.SerializerFeature;

// TestFixtures: leave both readJson method bodies unchanged

// Serialization inside FixtureRecorder.record:
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

// Serialization inside FixtureRecorder.record (same MAPPER setup, inside the try block):
String json = MAPPER.writerWithDefaultPrettyPrinter().writeValueAsString(value);
```

Spring Boot 4 defaults to Jackson 3, package `tools.jackson`, and the exceptions are unchecked. Adjust the imports to the classes that actually exist in the dependencies, and drop the `try/catch`.
