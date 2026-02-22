# 自动配置 `RestClient` 或 `WebClient` Bean
本 starter 旨在通过应用配置属性自动配置 `RestClient` 和 `WebClient`：
- 请求授权：
  - 使用 Spring Security OAuth2 client registration 的 Bearer 认证
  - 复用 `oauth2ResourceServer` 请求 security context 中 access token 的 Bearer 认证
  - 静态请求头值（API KEY）
  - Basic 认证
- base path（可在每个部署环境中覆盖的配置项）
- 使用 `HTTP_PROXY` 和 `NO_PROXY` 环境变量自动配置代理（可通过配置属性覆盖或补充，例如为 HTTP 代理设置认证凭据）
- 连接超时和读取超时
- 按 client 粒度禁用 SSL 证书验证
- 选择 `RestClient` 底层使用的 `ClientHttpRequestFactory`：
  - `SimpleClientHttpRequestFactory` 不支持 `PATCH` 请求
  - 默认使用 `JdkClientHttpRequestFactory`，但它会设置某些 Microsoft 中间件不支持的请求头
  - `HttpComponentsClientHttpRequestFactory` 和 `JettyClientHttpRequestFactory` 需要在 classpath 中添加额外的 jar
- 对 `RestClient`/`WebClient` bean 配置的完全灵活性：可通过配置属性选择暴露预配置的 `Builder` 而非已构建好的实例，从而在 Java 代码中进一步精调配置（用配置属性处理 starter 支持的自动配置部分，手动定义 starter 不支持的部分）
- 支持同时自动配置多个 `RestClient`/`WebClient`（或 builder）bean

在 WebFlux 应用中实例化的 REST client 为 `WebClient`，在 servlet 应用中为 `RestClient`，但任意 client 都可以在 servlet 中切换为 `WebClient`。

暴露的 bean 名称默认是将配置属性 map 中 `kebab-case` 的 key 转换为 `camelCase` 的形式，若 `expose-builder` 为 `true` 则追加 `Builder` 后缀。也可以在配置属性中自定义为任意名称。

本 starter 与其他 `spring-addons` starter 没有依赖关系（`spring-addons-starter-rest` 可以不依赖 `spring-addons-starter-oidc` 单独使用）。

- [1. Bean 命名的重要注意事项](#warning)
- [2. 使用方法](#usage)
  - [2.1. 依赖配置](#dependency)
  - [2.2. 最简示例](#minimal-sample)
  - [2.3. 高级配置示例](#advanced-configuration)
  - [2.4. 在 Java 代码中定义 `RestClient.Builder`/`WebClient.Builder` bean](#builder-in-java)
  - [2.5. 将生成的 `@HttpExchange` 代理暴露为 `@Bean`](#http-exchange-proxies)
  - [2.6. 更换默认的 `ClientHttpRequestFactory`](#client-http-request-factory)
  - [2.7. 使用 SSL bundle](#ssl-bundles)
  - [2.8. 在非 Web 应用中使用 `spring-addons-starter-rest`](#non-web)

## <a name="warning">1. Bean 命名的重要注意事项

默认情况下，bean 名称是配置属性中 key 的 camel-case 转换形式，若 `expose-builder` 属性为 true 则追加 `Builder` 后缀。例如：
```yaml
com:
  c4-soft:
    springaddons:
      rest:
        client:
          foo-client:
            base-url: https://foo:8080
          bar-client:
            base-url: https://bar:8080
            expose-builder: true
```
会分别创建名为 `fooClient` 和 `barClientBuilder` 的 bean。

`restClientBuilder` 和 `webClientBuilder` 是"官方" starter 创建的 bean 名称，`spring-addons` 以它们为基础进行扩展，应视为保留名称。**强烈不建议在 `com.c4-soft.springaddons.rest.client` 配置属性中使用 `rest-client` 或 `web-client` 作为 key。**

不过，可以通过暴露 `@Bean RestClient.Builder restClientBuilder() { ... }` 或 `@Bean WebClient.Builder webClientBuilder() { ... }` 来覆盖 Spring Boot 的默认行为。这样 `spring-addons` 会使用这些自定义 bean 作为自动配置的基础，而非"官方" starter 暴露的 bean。

## <a name="usage">2. 使用方法

为了充分发挥 `RestClient`/`WebClient` 的价值，可以将其提供给 [`@HttpExchange` 代理工厂](https://github.com/fivetime/spring-addons/tree/master/spring-addons-starter-rest#exposing-a-generated-httpexchange-proxy-as-a-bean)。

提醒一下，`@HttpExchange` 接口从客户端视角描述一个 REST API，上述代理是用于消费该 API 的自动生成实现类。可以将其理解为 REST 领域等价于关系型数据库中 `@Repository` 的存在。

如果所消费的 REST API 暴露了 OpenAPI 规范（例如使用 Swagger，可能通过 [`springdoc-openapi`](https://springdoc.org/) 和 [Swagger annotations](https://github.com/swagger-api/swagger-core/wiki/Swagger-2.X---Annotations) 实现），则可以使用 [`openapi-generator-maven-plugin`](https://github.com/OpenAPITools/openapi-generator/tree/master/modules/openapi-generator-maven-plugin) 或 [`openapi-generator-gradle-plugin`](https://github.com/OpenAPITools/openapi-generator/tree/master/modules/openapi-generator-gradle-plugin) 自动生成 `@HttpExchange` 接口。

换言之，我们几乎可以零样板代码地消费 REST API：
1. 从 REST API 源码生成 OpenAPI 规范
2. 从 OpenAPI 规范生成描述客户端如何消费这些 API 的 `@HttpExchange` 接口
3. 生成 `@HttpExchange` 代理，为每个代理提供由 `spring-addons-starter-rest` 自动配置好的 `RestClient`/`WebClient` bean

以下内容描述第 3 步。关于如何从 `@RestController` 源码生成 OpenAPI 规范，或如何从规范生成 `@HttpExchange`，请参考上面链接的文档。

### <a name="dependency">2.1. 依赖配置
```xml
<dependency>
    <groupId>com.c4-soft.springaddons</groupId>
    <artifactId>spring-addons-starter-rest</artifactId>
    <version>${spring-addons.version}</version>
</dependency>
```

### <a name="minimal-sample" />2.2. 最简示例
```yaml
com:
  c4-soft:
    springaddons:
      rest:
        client:
          keycloak-admin-client:
            base-url: ${keycloak-base-uri}/admin/realms
            authorization:
              oauth2:
                forward-bearer: true
```
这会暴露一个名为 `keycloakAdminClient` 的预配置 bean。在 servlet 应用中该 bean 的默认类型为 `RestClient`，在 WebFlux 应用中为 `WebClient`。

### <a name="advanced-configuration" />2.3. 高级配置示例
```yaml
machin-api-base-url: http://localhost:8081
com:
  c4-soft:
    springaddons:
      rest:
        client:
          # 暴露名为 "machinClient" 的 bean
          machin-client:
            # 在每个部署环境中可以方便地覆盖（例如通过 MACHIN_API_BASE_URL 环境变量）
            base-url: ${machin-api-base-url}
            http:
              connect-timeout-millis: 1000
              read-timeout-millis: 1000
              # 需要 org.apache.httpcomponents.client5:httpclient5 在 classpath 中
              client-http-request-factory-impl: http-components
              # 禁用 SSL 证书验证
              # 当 "client-http-request-factory-impl" 为 "jdk" 时，仅禁用根证书颁发机构验证（不禁用主机名验证，因此证书的 CN 或 altnames 必须与 URL 主机名匹配）
              ssl-certificates-validation-enabled: false
              # 覆盖 HTTP_PROXY 和 NO_PROXY 环境变量中定义的值
              proxy:
                connect-timeout-millis: 500
                enabled: true
                host: proxy2.corporate.pf
                non-proxy-hosts-pattern: .+\.corporate\.pf
                username: spring-backend
                password: secret
                port: 8080
                protocol: http
            authorization:
              oauth2:
                # 使用 security context 中的 Bearer token 授权出站请求（仅在 resource server 应用中可用）
                forward-bearer: true
          # 暴露名为 "biduleClientBuilder" 的 bean（注意下方的 "expose-builder: true"）
          bidule-client:
            base-url: http://localhost:${bidule-api-port}
            # 暴露 RestClient.Builder 而非已构建好的 RestClient
            # 默认 bean 名称会追加 "Builder" 后缀（"bidule-client" -> "biduleClient" -> "biduleClientBuilder"）
            expose-builder: true
            authorization:
              oauth2:
                # 使用通过 OAuth2 client registration 获取的 Bearer token 授权出站请求
                oauth2-registration-id: bidule-registration
            http:
              proxy:
                # 使用 HTTP_PROXY 和 NO_PROXY 环境变量，并添加代理认证
                username: spring-backend
                password: secret
          chose-client:
            base-url: http://localhost:${chose-api-port}
            # 在 servlet 应用中暴露 WebClient 而非 RestClient
            type: WEB_CLIENT
            # 将 bean 名称改为 "chose"（默认为 "choseClient"）
            bean-name: chose
            authorization:
              # 使用 Basic 认证授权出站请求
              basic:
                username: spring-backend
                password: secret
            http:
              proxy:
                # 忽略 HTTP_PROXY 环境变量
                enabled: false
          chouette-client:
            base-url: https://something.pf/api
            # 添加静态值请求头
            headers:
              X-API-KEY: change-me
              X-MULTI-VALUED-HEADER: 
              - foo
              - bar
              - bam
```
`biduleClientBuilder` bean 可以如下方式定义 `biduleClient` bean：
```java
/** 
 * @param biduleClientBuilder 通过应用配置属性预配置好的 builder
 * @return 名为 "biduleClient" 的 {@link RestClient} bean
 */
@Bean
RestClient biduleClient(RestClient.Builder biduleClientBuilder) throws Exception {
  // 进一步精调 biduleClientBuilder 的配置
  return biduleClientBuilder.build();
}
```

### <a name="builder-in-java" />2.4. 在 Java 代码中定义 `RestClient.Builder`/`WebClient.Builder` bean

`spring-addons-starter-rest` 以"官方" Boot starter 定义的 `restClientBuilder` 和 `webClientBuilder` bean 为基础来构建它自动配置的 builder。

但由于这些 bean 都标注了 `@ConditionalOnMissingBean`，在 `@Configuration` 类中定义 `RestClient.Builder` 或 `WebClient.Builder` 会阻止这些基础 bean 的创建，应用代码必须自行提供替代实现。以下代码在大多数情况下已经足够：
```java
// 暴露一个供其他组件使用的基础 builder
// bean 名称很重要："restClientBuilder"（WebClient.Builder 对应 "webClientBuilder"）
@Bean
RestClient.Builder restClientBuilder() {
  // 大多数情况下这已足够，但也可以进一步增强，
  // 例如使用特定的 ObjectMapper 实例等
  return RestClient.builder();
}

// 将某些 builder 暴露为 bean
// 请慎重考虑，因为大多数情况下暴露 client 本身比暴露其 builder 更合适
@Bean
RestClient.Builder fooClientBuilder(RestClient.Builder autoConfiguredBuilder) {
  return autoConfiguredBuilder.clone();
}
```

### <a name="http-exchange-proxies" />2.5. 将生成的 `@HttpExchange` 代理暴露为 `@Bean`
REST client 配置完成后，可以用它们来生成 `@HttpExchange` 实现类：
```java
/** 
 * @param machinClient 由 spring-addons-starter-rest 根据应用配置属性预配置好的 client
 * @return {@link MachinApi} {@link HttpExchange &#64;HttpExchange} 的自动生成实现，以名为 "machinApi" 的 bean 暴露
 */
@Bean
MachinApi machinApi(RestClient machinClient) throws Exception {
  return new RestClientHttpExchangeProxyFactoryBean<>(MachinApi.class, machinClient).getObject();
}

/** 
 * @param biduleClient 上方暴露的 bean
 * @return {@link BiduleApi} {@link HttpExchange &#64;HttpExchange} 的自动生成实现，以名为 "biduleApi" 的 bean 暴露
 */
@Bean
BiduleApi biduleApi(RestClient biduleClient) throws Exception {
  return new RestClientHttpExchangeProxyFactoryBean<>(BiduleApi.class, biduleClient).getObject();
}
```

### <a name="client-http-request-factory" />2.6. 更换默认的 `ClientHttpRequestFactory`
如果应用中已配置了 `ClientHttpRequestFactory` bean，`spring-addons-starter-rest` 会将其用于所有自动配置的 `RestClient` bean。否则，会自动配置一个，包含：
- 若设置了配置属性或 `HTTP_PROXY` & `NO_PROXY` 环境变量，则配置 HTTP 代理
- 超时设置

默认实现为 `JdkClientHttpRequestFactory`，可通过配置属性切换为 `HttpComponentsClientHttpRequestFactory` 或 `JettyClientHttpRequestFactory`：
```yaml
com:
  c4-soft:
    springaddons:
      rest:
        client:
          machin-client:
            http:
              # 需要 org.apache.httpcomponents.client5:httpclient5 在 classpath 中
              client-http-request-factory-impl: http-components
          bidule-client:
            http:
              # 需要 org.eclipse.jetty:jetty-client 在 classpath 中
              client-http-request-factory-impl: jetty
```

### <a name="ssl-bundles" />2.7. 使用 SSL bundle
以下面的 Boot 配置为例，使用 `openssl req -x509 -newkey rsa:4096 -keyout tls.key -out tls.crt -sha256 -days 3650 -passout pass:change-me -subj "/C=PY/ST=Tahiti/L=Papeete/CN=localhost/emailAddress=ch4mp@c4-soft.com"` 命令生成自签名证书：
- 被调用的 REST API 端：
```yaml
server:
  port: 8081
  ssl:
    bundle: server
spring:
  ssl:
    bundle:
      pem:
        server:
          keystore:
            certificate: classpath:tls.crt
            private-key: classpath:tls.key
            private-key-password: change-me
```
- 使用 REST client 调用上述服务的调用方：
```yaml
spring:
  ssl:
    bundle:
      pem:
        client:
          truststore:
            certificate: classpath:tls.crt
com:
  c4-soft:
    springaddons:
      rest:
        client:
          machin-client:
            base-url: https://localhost:8081
            ssl-bundle: client
```
在上述配置中：
- 同一张（自签名）证书分别用于在被调用服务上配置名为 `server` 的 bundle，以及在调用方配置名为 `client` 的 bundle。
- 被调用服务的 `server.ssl.bundle` 属性引用了带有 `keystore` 的 bundle（`spring.ssl.bundle.server`）。
- 调用方的 `com.c4-soft.springaddons.rest.client.{client-id}.ssl-bundle` 属性引用了带有 `truststore` 的 bundle（`spring.ssl.bundle.client`）。


### <a name="non-web" />2.8. 在非 Web 应用中使用 `spring-addons-starter-rest`

由于 `spring-boot-starter-oauth2-client` 只自动配置 Web 应用，在非 Web 应用中必须导入 `OAuth2ClientProperties` 并声明一个 `OAuth2AuthorizedClientManager` bean：
```java
@Configuration
@Import(OAuth2ClientProperties.class)
public class SecurityConfiguration {

  @Bean
  ClientRegistrationRepository clientRegistrationRepository(OAuth2ClientProperties properties) {
    List<ClientRegistration> registrations = new ArrayList<>(
        new OAuth2ClientPropertiesMapper(properties).asClientRegistrations().values());
    return new InMemoryClientRegistrationRepository(registrations);
  }

  @Bean
  OAuth2AuthorizedClientService authorizedClientService(
      ClientRegistrationRepository clientRegistrationRepository) {
    return new InMemoryOAuth2AuthorizedClientService(clientRegistrationRepository);
  }

  @Bean
  OAuth2AuthorizedClientManager authorizedClientManager(
      ClientRegistrationRepository clientRegistrationRepository,
      OAuth2AuthorizedClientService authorizedClientService) {
    return new AuthorizedClientServiceOAuth2AuthorizedClientManager(clientRegistrationRepository,
        authorizedClientService);
  }
}
```

同样地，`spring-addons-starter-rest` 通过后处理 bean 定义注册表来为 Web 应用添加 `RestClient`/`WebClient`（或 builder）bean 的定义。因此，若要在 servlet REST 应用中自动配置 `RestClient` bean，需要添加以下配置：
```java
@Bean
SpringAddonsRestClientBeanDefinitionRegistryPostProcessor springAddonsRestClientBeanDefinitionRegistryPostProcessor(Environment environment) {
  return new SpringAddonsRestClientBeanDefinitionRegistryPostProcessor(environment);
}
```
若要获取 `WebClient` bean，则应根据应用是同步还是响应式，分别使用 `SpringAddonsServletWebClientBeanDefinitionRegistryPostProcessor` 或 `SpringAddonsServerWebClientBeanDefinitionRegistryPostProcessor`。

使用 `SpringAddonsServerWebClientBeanDefinitionRegistryPostProcessor` 时，必须暴露 `ReactiveOAuth2AuthorizedClientManager` 而非 `OAuth2AuthorizedClientManager`。