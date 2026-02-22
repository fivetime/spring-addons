# spring-addons-starter-openapi

`springdoc-openapi` 的维护者拒绝在其库中考虑 Spring（反）序列化的特殊性。他甚至直接删除（而非仅仅关闭）了 issue #2494——该 issue 详细解释了为什么 Swagger 无法可靠地将枚举的可选值生成为 Spring REST API 的输入/输出规范，以及 `springdoc-openapi` 应该用什么来替代 `swagger-core` 的默认行为。

## 枚举可选值
`springdoc-openapi` 依赖 `swagger-core` 的枚举序列化来将枚举的可选值写入规范。这种做法并不合理，因为 Spring 从不使用 `swagger-core` 来（反）序列化枚举值。Spring 实际使用的是：
- `FormattingConversionService` 提供的 converter，用于处理 `@RequestParam`、`@RequestHeader`、`@PathVariable`、`@MatrixVariable` 和 `@CookieValue`
- servlet 应用中用 `HttpMessageConverter` 反序列化 `@RequestBody` 并序列化 `@ResponseBody`
- 响应式应用中用 `HttpMessageReader`（处理 `@RequestBody`）和 `HttpMessageWriter`（处理 `@ResponseBody`）

`spring-addons-starter-openapi` 的工作原理是：
- 扫描 classpath 以判断应用是 servlet 还是响应式应用
- 从 classpath 中挑选相关的 Spring bean 来（反）序列化给定的枚举
- 注册一个使用 Spring bean 的自定义 `ModelConverter`，使 Swagger 在 OpenAPI 规范中写入正确的枚举序列化值

## 警告：本 starter 目前仅支持 servlet 应用 :/
响应式应用的大部分工作已经完成，但本仓库的维护者尚未找到同步 `HttpMessageWriter` 的方法。

如果你找到了让 `SpringReactiveEnumModelConverter#toWrittingExtractor` 正常工作的方法（[修复其中的 `FIXME`](https://github.com/fivetime/spring-addons/blob/dfbe20387ae7c423c103b6e5b6d853087473a537/spring-addons-starter-openapi/src/main/java/com/c4_soft/springaddons/openapi/SpringReactiveEnumModelConverter.java#L158)），欢迎提交 PR ;)

要测试你的实现，可以重新启用：
- `src/main/java/com/c4_soft/springaddons/openapi/SpringAddonsOpenapiAutoConfiguration.java` 中的 `springReactiveEnumModelConverter` bean
- `samples/enum-bug-reproducer-reactive/src/test/java/com/c4soft/EnumBugReproducerApplicationTests.java` 中的测试用例