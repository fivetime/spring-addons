# spring-addons-oauth2 示例

建议先从 [tutorials](https://github.com/fivetime/spring-addons/tree/master/samples/tutorials) 入手，再克隆本仓库运行或修改示例。

以下场景的示例均包含**配置、单元测试和集成测试**：
- servlet（webmvc）/ 响应式（webflux）应用
- JWT decoder / access token introspection
- Spring 内置的 `JwtAuthenticationToken`（JWT decoder）或 `BearerTokenAuthentication`（introspection）/ 本仓库的 `OAuthentication<OpenidClaimSet>`
- 从 token 中获取 granted authorities，或从外部数据源获取（示例中使用 JPA repository，也可以是 web service）
- 使用测试注解或"流式 API"（MockMvc 请求后处理器和 WebTestClient mutator）

所有使用本仓库 starter `@AutoConfiguration` 的示例，其配置来源有三处：
- `application.properties` 文件
- servlet 或响应式应用的自动配置 bean
- 主类中通过 `@Bean` 覆盖的 bean

## `Authentication` 实现的可用性对比
各示例使用三种不同的 `Authentication`，但结构相同：一个简单的 `@RestController` 从 `@Service` 获取消息，`@Service` 再从 `@Repository` 获取数据。

以下是访问 granted authorities 和 `preferred_username` OpenID claim 的 `greet()` 方法的实现对比：

### `JwtAuthenticationToken`
由 Spring Security 配合 JWT decoder 提供。简单，但不提供 OpenID claim 访问器。
``` java
public String greet(JwtAuthenticationToken who) {
    return String.format(
        "Hello %s! You are granted with %s.",
        who.getToken().getClaimAsString(StandardClaimNames.PREFERRED_USERNAME),
        who.getAuthorities().stream().map(GrantedAuthority::getAuthority).toList());
}
```

### `BearerTokenAuthentication`
与上述类似，用于 access token introspection。
``` java
public String greet(BearerTokenAuthentication who) {
    return String.format(
            "Hello %s! You are granted with %s.",
            who.getTokenAttributes().get(StandardClaimNames.PREFERRED_USERNAME),
            who.getAuthorities().stream().map(GrantedAuthority::getAuthority).toList());
}
```

### `OAuthentication<OpenidClaimSet>`
由 `spring-addons-starter-oidc` 配合 `Converter<Jwt, ? extends AbstractAuthenticationToken>` 提供。三者中可用性/灵活性/可扩展性最强。
``` java
public String greet(OAuthentication<OpenidClaimSet> who) {
    return String.format(
        "Hello %s! You are granted with %s.",
        who.getToken().getPreferredUsername(),
        who.getAuthorities().stream().map(GrantedAuthority::getAuthority).toList());
}
```