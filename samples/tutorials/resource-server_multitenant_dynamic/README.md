# 如何配置支持动态租户的 Spring REST API
spring-addons 自动配置高级定制示例。

本教程区分两种多租户模式：
- **静态多租户**：所有受信任的 issuer 在启动前已知。`spring-addons-starter-oidc` 通过应用配置属性支持此模式：`com.c4-soft.springaddons.oidc.ops` 是一个数组，其中 `ops` 代表 **O**penID **P**rovider**s**（即 issuer）。在此场景下，需要为每个 OP 定义一组属性，properties resolver 将使用 access token 的 `iss` claim 来匹配构建请求 security context 时应使用的属性组。
- **动态多租户**：部分需要信任的 OpenID Provider 可能在 Spring Relying Party 启动后才创建。典型用例是 B2B 系统，为每家订阅服务的公司创建一个专属 issuer。在这种情况下，无法在应用配置属性中提前配置 OP 信息。`spring-addons-starter-oidc` 通过扫描应用上下文中的 `OpenidProviderPropertiesResolver` 来解决这个问题——该 resolver 负责根据 access token 的 claim 解析对应的 OP 配置属性。

本教程关注的是**动态多租户**。

我们将暴露的 properties resolver 设计为接受某个指定 Keycloak 服务器上任意 realm 签发的 access token。

Issuer URI 的格式应为 `${keycloak-host}/realms/{realm-id}`。

我们将对所有 realm 使用同一组配置属性，同时确保只接受来自我们信任的唯一 Keycloak 服务器上某个 realm 签发的 token。

## 0. 免责声明
本仓库示例数量较多，所有示例均纳入 CI 以确保代码可编译且测试全部通过。遗憾的是，此 README 不会随源码变更自动更新。请将其作为理解源码的参考指引。**如需复制代码，请务必从源码中复制，而非从此 README 中复制。**

## 1. 前置条件
假设已完成[教程主 README 的前置条件章节](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials#prerequisites)，并已准备好至少 1 个 OIDC Provider（2 个更好），且已配置带有 authorization-code 流程的 client ID 和 secret。

## 2. 项目初始化
本教程在 [`resource-server_with_oauthentication` 教程](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials/resource-server_with_oauthentication)的基础上继续。请确保该项目已正常运行后再开始本教程。

## 3. Web Security 配置
如 [`spring-addons-starter-oidc` README]() 中所述，我们只需要：
- 定义应用于所有 realm 的配置：
```yaml
scheme: http
keycloak-port: 8080
keycloak-host: ${scheme}://localhost:${keycloak-port}

com:
  c4-soft:
    springaddons:
      oidc:
        ops:
        - iss: ${keycloak-host}
          authorities:
          - path: $.realm_access.roles
          - path: $.resource_access.*.roles
```
- 暴露一个 `OpenidProviderPropertiesResolver`，根据 token claim 解析上述配置属性：
```java
@Configuration
@EnableMethodSecurity
public class WebSecurityConfig {

  @Component
  public class IssuerStartsWithOpenidProviderPropertiesResolver implements OpenidProviderPropertiesResolver {
    private final SpringAddonsOidcProperties properties;

    public IssuerStartsWithOpenidProviderPropertiesResolver(SpringAddonsOidcProperties properties) {
      this.properties = properties;
    }

    @Override
    public Optional<OpenidProviderProperties> resolve(Map<String, Object> claimSet) {
      final var tokenIss =
          Optional.ofNullable(claimSet.get(JwtClaimNames.ISS)).map(Object::toString)
              .orElseThrow(() -> new RuntimeException("Invalid token: missing issuer"));
      return properties.getOps().stream().filter(opProps -> {
        final var opBaseHref =
            Optional.ofNullable(opProps.getIss()).map(URI::toString).orElse(null);
        if (!StringUtils.hasText(opBaseHref)) {
          return false;
        }
        return tokenIss.startsWith(opBaseHref);
      }).findAny();
    }
  }
}
```
`OpenidProviderPropertiesResolver` 仅在 token 的 `iss` claim 以配置中 `iss` 属性的值为前缀时，才返回对应的 `OpenidProviderProperties`。

注意，我们也可以通过模式匹配实现更严格的校验，但这个实现对于演示目的已经足够。

## 5. 示例 `@RestController`
无需改动，直接沿用 [`resource-server_with_oauthentication`](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials/resource-server_with_oauthentication) 中的 controller 即可。

## 5. 单元测试
无需改动，直接沿用 [`resource-server_with_oauthentication`](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials/resource-server_with_oauthentication) 中的测试即可。

## 6. 总结
Et voilà！现在我们的 API 可以接受 Keycloak 实例上任意 realm 签发的 access token 了（同时也领略了 `spring-addons-starter-oidc` 的灵活性）。