# 如何扩展 `OAuthentication<OpenidClaimSet>`

## 0. 免责声明
本仓库示例数量较多，所有示例均纳入 CI 以确保代码可编译且测试全部通过。遗憾的是，此 README 不会随源码变更自动更新。请将其作为理解源码的参考指引。**如需复制代码，请务必从源码中复制，而非从此 README 中复制。**

## 1. 概述
假设我们有一个业务需求，其安全控制不仅限于基于角色的方式。

假设授权服务器还提供了一个 `proxies` claim，其中包含一个以用户 `preferredUsername` 为键、权限集合为值的映射（即当前用户被授权代表其他用户执行的操作）。

本教程将演示：
- 如何扩展 `OAuthentication<OpenidClaimSet>`，在 authorities 之外额外持有这些代理权限（proxies）
- 如何扩展 security SpEL，以便轻松判断已认证用户被授予的代理权限、OpenID claim 或其他与 security context 相关的信息

## 2. 项目初始化
借助 https://start.spring.io/ 创建一个 Spring Boot 3 项目，需要以下依赖：
- lombok

然后添加 spring-addons 相关依赖：
- [`spring-addons-starter-oidc`](https://central.sonatype.com/artifact/com.c4-soft.springaddons/spring-addons-starter-oidc)
- [`spring-addons-starter-oidc-test`](https://central.sonatype.com/artifact/com.c4-soft.springaddons/spring-addons-starter-oidc-test)（`test` scope）
```xml
<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-starter-oidc</artifactId>
    <version>${spring-addons.version}</version>
</dependency>
<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-starter-oidc-test</artifactId>
    <version>${spring-addons.version}</version>
    <scope>test</scope>
</dependency>
```

## 3. Web Security 配置

### 3.1. `ProxiesClaimSet` 和 `ProxiesAuthentication`
首先定义 `Proxy` 是什么：
```java
@Data
public class Proxy implements Serializable {
  private static final long serialVersionUID = 8853377414305913148L;

  private final String proxiedUsername;
  private final String tenantUsername;
  private final Set<String> permissions;

  public Proxy(String proxiedUsername, String tenantUsername, Collection<String> permissions) {
    this.proxiedUsername = proxiedUsername;
    this.tenantUsername = tenantUsername;
    this.permissions = Collections.unmodifiableSet(new HashSet<>(permissions));
  }

  public boolean can(String permission) {
    return permissions.contains(permission);
  }
}
```

然后，扩展 `OpenidToken` 以添加 `proxies` 私有 claim 的解析：
```java
@Data
@EqualsAndHashCode(callSuper = true)
public class ProxiesToken extends OpenidToken {
  private static final long serialVersionUID = 2859979941152449048L;

  private final Map<String, Proxy> proxies;

  public ProxiesToken(Map<String, Object> claims, String tokenValue) {
    super(claims, StandardClaimNames.PREFERRED_USERNAME, tokenValue);
    this.proxies = Collections
        .unmodifiableMap(Optional.ofNullable(proxiesConverter.convert(this)).orElse(Map.of()));
  }

  public Proxy getProxyFor(String username) {
    return proxies.getOrDefault(username, new Proxy(username, getName(), List.of()));
  }

  private static final Converter<OpenidClaimSet, Map<String, Proxy>> proxiesConverter = claims -> {
    @SuppressWarnings("unchecked")
    final var proxiesClaim = (Map<String, List<String>>) claims.get("proxies");
    if (proxiesClaim == null) {
      return Map.of();
    }
    return proxiesClaim.entrySet().stream()
        .map(e -> new Proxy(e.getKey(), claims.getPreferredUsername(), e.getValue()))
        .collect(Collectors.toMap(Proxy::getProxiedUsername, p -> p));
  };
}
```
最后，扩展 `OAuthentication` 以：
- 覆盖 `getName()`（本教程中用户以 preferred_username 作为唯一标识）
- 提供对指定用户代理权限的直接访问器（来自上面的 `ProxiesClaimSet`）
```java
@Data
@EqualsAndHashCode(callSuper = true)
public class ProxiesAuthentication extends OAuthentication<ProxiesToken> {
  private static final long serialVersionUID = 447991554788295331L;

  public ProxiesAuthentication(ProxiesToken token,
      Collection<? extends GrantedAuthority> authorities) {
    super(token, authorities);
  }

  public boolean hasName(String username) {
    return Objects.equals(getName(), username);
  }

  public Proxy getProxyFor(String username) {
    return getAttributes().getProxyFor(username);
  }
}
```

### 3.2. Security `@Bean` 配置
我们依赖 `spring-addons-starter-oidc` 的 `@AutoConfiguration`，只需强制指定 authentication converter。

同时，我们还扩展了 security SpEL，添加以下几个方法：
- 将当前用户的用户名与给定值进行比较
- 访问当前用户代表他人（按用户名指定）的代理权限
- 判断当前用户是否拥有某个"nice" authority

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

  @Bean
  JwtAbstractAuthenticationTokenConverter authenticationConverter(
      Converter<Map<String, Object>, Collection<? extends GrantedAuthority>> authoritiesConverter) {
    return jwt -> {
      final var token = new ProxiesToken(jwt.getClaims(), jwt.getTokenValue());
      return new ProxiesAuthentication(token, authoritiesConverter.convert(token));
    };
  }

  @Bean
  static MethodSecurityExpressionHandler methodSecurityExpressionHandler() {
    return new SpringAddonsMethodSecurityExpressionHandler(
        ProxiesMethodSecurityExpressionRoot::new);
  }

  static final class ProxiesMethodSecurityExpressionRoot
      extends SpringAddonsMethodSecurityExpressionRoot {

    public boolean is(String preferredUsername) {
      return Objects.equals(preferredUsername, getAuthentication().getName());
    }

    public Proxy onBehalfOf(String proxiedUsername) {
      return get(ProxiesAuthentication.class).map(a -> a.getProxyFor(proxiedUsername))
          .orElse(new Proxy(proxiedUsername, getAuthentication().getName(), List.of()));
    }

    public boolean isNice() {
      return hasAnyAuthority("NICE", "SUPER_COOL");
    }
  }
}
```

### 3.3. 配置属性
`application.yml`：
```yaml
com:
  c4-soft:
    springaddons:
      oidc:
        ops:
        - iss: ${keycloak-issuer}
          username-claim: preferred_username
          authorities:
          - path: $.realm_access.roles
          - path: $.resource_access.*.roles
        - iss: ${cognito-issuer}
          username-claim: username
          authorities:
          - path: cognito:groups
        - iss: ${auth0-issuer}
          username-claim: $['https://c4-soft.com/user']['name']
          authorities:
          - path: $['https://c4-soft.com/user']['roles']
          - path: $.permissions
        resourceserver:
          cors:
          - path: /**
            allowed-origin-patterns: ${origins}
          permit-all:
          - "/greet/public"
```

## 4. 示例 `@RestController`
注意第二个方法上的 `@PreAuthorize("is(#username) or isNice() or onBehalfOf(#username).can('greet')")`，它断言用户满足以下条件之一：
- 正在向自己打招呼
- 拥有某个"nice" authority
- 拥有代表 `preferred_username` 等于路径变量 `username` 的用户执行 `greet` 操作的权限（路由为 `/greet/{username}`）

``` java
@RestController
@RequestMapping("/greet")
public class GreetingController {

    @GetMapping()
    @PreAuthorize("hasAuthority('NICE')")
    public String getGreeting(ProxiesAuthentication auth) {
        return "Hi %s! You are granted with: %s and can proxy: %s.".formatted(
                auth.getName(),
                auth.getAuthorities().stream().map(GrantedAuthority::getAuthority).collect(Collectors.joining(", ", "[", "]")),
                auth.getClaims().getProxies().keySet().stream().collect(Collectors.joining(", ", "[", "]")));
    }

    @GetMapping("/public")
    public String getPublicGreeting() {
        return "Hello world";
    }

    @GetMapping("/on-behalf-of/{username}")
    @PreAuthorize("is(#username) or isNice() or onBehalfOf(#username).can('greet')")
    public String getGreetingFor(@PathVariable(name = "username") String username, Authentication auth) {
        return "Hi %s from %s!".formatted(username, auth.getName());
    }
}
```

## 5. 单元测试

`@WithJwt` 背后的 authentication factory 如果在 security context 中找到了 authentication converter bean，就会使用它。

由于我们将自己的 converter 暴露为 bean，`@WithJwt` 将用 `ProxiesAuthentication` 实例填充测试 security context。但请注意，`spring-security-tests` 中的 mutator（如 `.jwt()`）不会这样做。
```java
@WebMvcTest(controllers = GreetingController.class)
@AutoConfigureAddonsWebmvcResourceServerSecurity
@Import({ SecurityConfig.class })
class GreetingControllerTest {

    @Autowired
    MockMvcSupport mockMvc;

    // @formatter:off
    @Test
    @WithAnonymousUser
    void givenRequestIsAnonymous_whenGreet_thenUnauthorized() throws Exception {
        mockMvc.get("/greet")
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithAnonymousUser
    void givenRequestIsAnonymous_whenGreetPublic_thenOk() throws Exception {
        mockMvc.get("/greet/public")
            .andExpect(status().isOk())
            .andExpect(content().string("Hello world"));
    }

    @Test
    @WithJwt("ch4mp.json")
    void givenUserIsGrantedWithNice_whenGreet_thenOk() throws Exception {
        mockMvc.get("/greet")
            .andExpect(status().isOk())
            .andExpect(content().string("Hi ch4mp! You are granted with: [NICE, AUTHOR] and can proxy: [chose, machin]."));
    }

    @Test
    @WithJwt("tonton_proxy_ch4mp.json")
    void givenUserIsNotGrantedWithNice_whenGreet_thenForbidden() throws Exception {
        mockMvc.get("/greet")
            .andExpect(status().isForbidden());
    }

    @Test
    @WithJwt("tonton_proxy_ch4mp.json")
    void givenUserIsNotGrantedWithNiceButHasProxyForGreetedUser_whenGreetOnBehalfOf_thenOk() throws Exception {
        mockMvc.get("/greet/on-behalf-of/ch4mp")
            .andExpect(status().isOk())
            .andExpect(content().string("Hi ch4mp from Tonton Pirate!"));
    }

    @Test
    @WithJwt("ch4mp.json")
    void givenUserIsGrantedWithNice_whenGreetOnBehalfOf_thenOk() throws Exception {
        mockMvc.get("/greet/on-behalf-of/Tonton Pirate")
            .andExpect(status().isOk())
            .andExpect(content().string("Hi Tonton Pirate from ch4mp!"));
    }

    @Test
    @WithJwt("tonton_proxy_ch4mp.json")
    void givenUserIsNotGrantedWithNiceAndHasNoProxyForGreetedUser_whenGreetOnBehalfOf_thenForbidden() throws Exception {
        mockMvc.get("/greet/on-behalf-of/greeted")
            .andExpect(status().isForbidden());
    }

    @Test
    @WithJwt("tonton_proxy_ch4mp.json")
    void givenUserIsGreetingHimself_whenGreetOnBehalfOf_thenOk() throws Exception {
        mockMvc.get("/greet/on-behalf-of/Tonton Pirate")
            .andExpect(status().isOk())
            .andExpect(content().string("Hi Tonton Pirate from Tonton Pirate!"));
    }
    // @formatter:on
}
```

## 6. 总结
本示例引导你构建了一个带有 JWT decoder 和自定义 `Authentication` 的 servlet（webmvc）应用。如需配置 webflux（响应式）resource server 或 access token introspection 的帮助，请参阅 [samples](https://github.com/fivetime/spring-addons/tree/master/samples)。