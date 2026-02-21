# [https://github.com/springdoc/springdoc-openapi/issues/2494](https://github.com/springdoc/springdoc-openapi/issues/2494) 的复现案例

考虑以下枚举定义：
```java
public static enum EnumSerializedByName {
    A("name a"),
    B("name b");

    String label;

    EnumSerializedByName(String label) {
        this.label = label;
    }

    @Override
    public String toString() {
        return label;
    }
}
```
```java
public static enum EnumSerializedByToString {
    A("str a"),
    B("str b");

    String label;

    EnumSerializedByToString(String label) {
        this.label = label;
    }

    @Override
    @JsonValue // 强制使用 toString() 进行序列化
    public String toString() {
        return label;
    }
}
```
```java
public static enum BijectiveEnumSerializedByToString {
    A("bij a"),
    B("bij b");

    String label;

    BijectiveEnumSerializedByToString(String label) {
        this.label = label;
    }

    @Override
    @JsonValue // 强制使用 toString() 进行序列化
    public String toString() {
        return label;
    }

    public static BijectiveEnumSerializedByToString fromString(String str) {
        for (final var e : BijectiveEnumSerializedByToString.values()) {
            if (Objects.equals(e.toString(), str)) {
                return e;
            }
        }
        return null;
    }

    @Component
    static class StringEnumSerializedByToStringConverter implements Converter<String, BijectiveEnumSerializedByToString> {
        @Override
        public BijectiveEnumSerializedByToString convert(String source) {
            return BijectiveEnumSerializedByToString.fromString(source);
        }
    }
}
```

在 Spring 默认配置下，包含以下 bean：
- `HttpMessageConverter` beans
- `Converter<String, EnumSerializedByName>`
- `Converter<String, EnumSerializedByToString>`

生成的规范如下：
```json
  "components": {
    "schemas": {
      "Dto": {
        "required": ["bij", "name", "str"],
        "type": "object",
        "properties": {
          "name": { "type": "string", "enum": ["name a", "name b"] },
          "str": { "type": "string", "enum": ["str a", "str b"] },
          "bij": { "type": "string", "enum": ["bij a", "bij b"] }
        }
      }
    }
  }
```

结果分析：
- `EnumSerializedByName` 的结果**有误**——它始终使用 `name()` 进行序列化，并使用 `valueOf()` 进行反序列化
- `BijectiveEnumSerializedByToString` 的结果**正确**
- `EnumSerializedByToString` 的结果：使用 `HttpMessageConverter` beans 时（`@RequestBody` 和 `@ResponseBody`）**正确**，但使用默认的 `Converter<String, EnumSerializedByToString>` 时（`@RequestParam`）**有误**