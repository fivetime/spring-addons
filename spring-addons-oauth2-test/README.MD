# 带安全上下文的单元测试与集成测试
测试方法级安全（`@PreAuthorize`、`@PostFilter` 等）需要配置 security context。`spring-security-test` 提供了 `MockMvc` 请求后处理器和 `WebTestClient` mutator 来实现这一点，但它们只能在请求上下文中工作，因此仅适用于测试受保护的 controller。

要测试任意类型的 `@Component`（`@Controller` 当然可以，`@Service` 和 `@Repository` 也不例外）上的方法级安全，只有两种选择：自行构建测试 security context 并用 stub / mock 的 authentication 填充它，或者使用注解来代劳。**本库提供了注解，可直接在测试 security context 中配置 OAuth2 认证。**

## 应该使用哪个注解

`@WithMockAuthentication` 足以测试任何使用 RBAC（基于角色的访问控制）的 OAuth2 应用：它可以方便地定义 `name` 和 `authorities`，以及在应用代码需要特定类型时要 mock 的 Authentication principal 类型。

如果你的访问控制不只依赖 name 和 authorities，还需要定义 claim-set 的详细内容，则可以选择：
- 在 `oauth2ResourceServer` 上，使用 `@WithJwt` 和 `@WithOpaqueToken`，它们使用 **classpath 中的 JSON payload** 和上下文中的 authentication converter 来构建 authentication 实例（见下方的[重要警告](#important)）
- 在使用 `oauth2Login` 的 client 上，使用 `@WithOAuth2Login` 和 `@WithOidcLogin`

## 快速上手

Baeldung 上发布了一篇[介绍本库中 OAuth2 测试注解用法的文章](https://www.baeldung.com/spring-oauth-testing-access-control)。

结合所有 [samples](https://github.com/ch4mpy/spring-addons/tree/master/samples) 和 [tutorials](https://github.com/ch4mpy/spring-addons/tree/master/samples/tutorials) 源码（包含大量单元测试和集成测试），应该足以帮你快速上手。

也可以参考下方的[示例](#sample)章节。

## <a name="important"/>重要警告
**`@WithJwt` 和 `@WithMockJwtAuth` 要求将自定义 authentication converter 暴露为 `@Bean`**（而非在 `SecurityFilterChain` 定义中以 lambda 内联的方式提供）。认证工厂需要通过这个 bean 来构建与运行时完全相同的 Authentication 实例。

`spring-addons-starter-oidc` 会将 authentication converter 暴露为 bean，但网上大多数仅使用 `spring-boot-starter-oauth2-resource-server` 的示例并不会这样做。

实践上，不要将 authentication converter 内联在 `SecurityFilterChain` 定义中，而应使用如下方式：
```java
@Bean
// 实现 Converter<Jwt, AbstractAuthenticationToken> 的类必须暴露为 @Bean，这一点很重要
JwtAuthenticationConverter authenticationConverter() {
    final var authenticationConverter = new JwtAuthenticationConverter();
    authenticationConverter.setPrincipalClaimName(StandardClaimNames.PREFERRED_USERNAME);
    authenticationConverter
            .setJwtGrantedAuthoritiesConverter(
                    (jwt) -> Optional
                            .ofNullable(jwt.getClaimAsStringList("roles"))
                            .orElse(List.of())
                            .stream()
                            .map(SimpleGrantedAuthority::new)
                            .map(GrantedAuthority.class::cast)
                            .toList());
    return authenticationConverter;
}

@Bean
// 该 bean 随后可以按如下方式注入 SecurityFilterChain
SecurityFilterChain securityFilterCHain(HttpSecurity http, Converter<Jwt, AbstractAuthenticationToken> authenticationConverter) throws Exception {
    http.oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(authenticationConverter)));
    ...
    return http.build();
}
```
这样做是为了让测试注解背后的工厂能从测试上下文中获取此 authentication converter（并用它构建放入测试 security context 的 `Authentication` 实例）。

## <a name="sample"/>示例

假设有如下受保护的 `@Service`：
```java
@Service
public class SecuredService {
    @PreAuthorize("hasAuthority('NICE')")
    String nice() {
        return "Dear %s, glad to see you!".formatted(SecurityContextHolder.getContext().getAuthentication().getName());
    }
    
    @PreAuthorize("isAuthenticated()")
    String hello() {
        return "Hello %s.".formatted(SecurityContextHolder.getContext().getAuthentication().getName());
    }
}
```
现在，假设你有一个预发布环境，其中有几个有代表性的用户（如果你熟悉 UX，可以理解为"用户画像"），可以获取这些用户的 access token 样本，并将 claim 转储为 JSON 文件放在测试资源目录中（通过 https://jwt.io 等工具解码 JWT，或通过 introspection 解析 opaque token）。以下示例假设有一个名为 `brice`、拥有 `NICE` authority 的用户，以及另一个名为 `igor`、没有 `NICE` authority 的用户，并且已将 claim-set 样本分别保存至 `src/test/resources/brice.json` 和 `src/test/resources/igor.json`。

### 使用 `@WithMockAuthentication`

在测试 RBAC（基于角色的访问控制）时，仅定义 authorities 通常就已足够。有时需要定义 `Authentication#name`，少数情况下应用代码还需要特定的 `Authentication` 实现。`@WithMockAuthentication` 正是为满足这些需求而设计的：
```java
@SpringBootTest(classes = { SecurityConfig.class, MessageService.class })
class MessageServiceTests {

	@Autowired
	private SecuredService securedService;
	
	@Test
	@WithMockAuthentication("BAD_BOY")
	void givenUserIsNotGrantedWithNice_whenCallNice_thenThrows() {
		assertThrows(Exception.class, () -> securedService.nice());
	}

	@Test
	@WithMockAuthentication(name = "brice", authorities = "NICE")
	void givenUserIsNice_whenCallNice_thenReturnsGreeting() {
		assertThat(securedService.nice()).isEqualTo("Dear brice, glad to see you!");
	}

	@ParameterizedTest
	@AuthenticationSource(
	    @WithMockAuthentication(name = "brice", authorities = "NICE"),
	    @WithMockAuthentication(name = "ch4mp", authorities = { "VERY_NICE", "AUTHOR" }))
	void givenUserIsAuthenticated_whenCallHello_thenReturnsGreeting(@ParameterizedAuthentication Authentication auth) {
		assertThat(securedService.hello()).isEqualTo("Hello %s.".formatted(auth.getName()));
	}
}
```

### 使用 `@WithJwt` 或 `@WithOpaqueToken` 配合 JSON claim-set

`@WithJwt` 和 `@WithOpaqueToken` 可以加载上述 claim-set 文件并将其转换为 `Authentication` 实例。

为了让 Authentication 的构建方式与运行时一致（类型、authorities、name、claims 等），`@WithJwt` 使用你的 `Converter<Jwt, ? extends AbstractAuthenticationToken>`，`@WithOpaqueToken` 使用 `OpaqueTokenAuthenticationConverter`（响应式应用中使用对应的响应式版本）。

为使自动装配正常工作，需要导入 `AuthenticationFactoriesTestConf`（使用 `@AddonsWebmvcComponentTest`、`AutoConfigureAddonsWebmvcClientSecurity`、`AutoConfigureAddonsWebmvcResourceServerSecurity` 或其响应式对应版本时已自动导入）：
```java
@Import(AuthenticationFactoriesTestConf.class) // 使用 spring-addons-oauth2-test 但未使用 spring-addons-starter-oidc 时需要
@SpringBootTest(classes = { SecurityConfig.class, MessageService.class })
class MessageServiceTests {

	@Autowired
	private SecuredService securedService;

	@Autowired
	WithJwt.AuthenticationFactory authFactory;
	
	@Test
	@WithJwt("igor.json")
	void givenUserIsIgor_whenCallNice_thenThrows() {
		assertThrows(Exception.class, () -> securedService.nice());
	}

	@Test
	@WithJwt("brice.json")
	void givenUserIsBrice_whenCallNice_thenReturnsGreeting() {
		assertThat(securedService.nice()).isEqualTo("Dear brice, glad to see you!");
	}

	@ParameterizedTest
	@MethodSource("identities")
	void givenUserIsAuthenticated_whenCallHello_thenReturnsGreeting(@ParameterizedAuthentication Authentication auth) {
		assertThat(securedService.hello()).isEqualTo("Hello %s.".formatted(auth.getName()));
	}

	Stream<AbstractAuthenticationToken> identities() {
		return authFactory.authenticationsFrom("brice.json", "igor.json");
	}
}
```
上述代码有几点值得关注：
- 我们在测试一个方法上带有 `@PreAuthorize` 的 `@Service`，完全不依赖 `MockMvc` 或 `WebTestClient`（及其请求后处理器或 mutator）
- 测试过程中 authorities 和 username 与 claims 保持一致（而使用 MockMvc 请求后处理器和 WebTestClient mutator 分别声明这三者时不一定能保证一致）。`WithJwt.AuthenticationFactory` 会使用 security 配置中的 JWT authorities converter，因此 `username` 和 `authorities` 与运行时一样从 claims 中解析得出
- claims 从测试 classpath 中的 JSON 文件加载
- 使用了 JUnit 5 的 `@ParameterizedTest`：测试会对 `identities` 方法返回的 stream 中的每个 authentication 各执行一次
- 注解与 BDD（given-when-then）风格完美契合：测试的前置条件（given）通过注解装饰测试方法，而非像 MockMvc 请求后处理器和 WebTestClient mutator 那样混在测试内容中
- 注解可以非常简洁且富有表达力

**使用 `@WithJwt`（以及自 `7.3.0` 起的 `@WithMockJwtAuth`）但未使用 `spring-addons-starter-oidc` 的重要提醒**：应将 JWT converter 暴露为 bean。实践上，不要将 authentication converter 内联在 `SecurityFilterChain` 定义中，而应使用如下方式：
```java
@Bean
// 实现 Converter<Jwt, AbstractAuthenticationToken> 的类必须暴露为 @Bean，这一点很重要
JwtAuthenticationConverter authenticationConverter() {
    final var authenticationConverter = new JwtAuthenticationConverter();
    authenticationConverter.setPrincipalClaimName(StandardClaimNames.PREFERRED_USERNAME);
    authenticationConverter
            .setJwtGrantedAuthoritiesConverter(
                    (jwt) -> Optional
                            .ofNullable(jwt.getClaimAsStringList("roles"))
                            .orElse(List.of())
                            .stream()
                            .map(SimpleGrantedAuthority::new)
                            .map(GrantedAuthority.class::cast)
                            .toList());
    return authenticationConverter;
}

@Bean
// 该 bean 随后可以按如下方式注入 SecurityFilterChain
SecurityFilterChain securityFilterCHain(HttpSecurity http, Converter<Jwt, AbstractAuthenticationToken> authenticationConverter) throws Exception {
    http.oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(authenticationConverter)));
    ...
    return http.build();
}
```
这样做是为了让测试注解能从测试上下文中获取此 authentication converter（并用它构建放入测试 security context 的 `Authentication` 实例）。