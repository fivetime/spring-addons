# 从 `7.x` 迁移到 `8.x`

## `spring-addons-starter-oidc`

唯一的破坏性变更围绕 `OAuthentication` 展开——它现在继承自 `AbstractOAuth2TokenAuthenticationToken`，以便与 Spring Security 生态系统的其余部分更好地集成。

如果直接使用 `OpenidClaimSet`，请将其包装在 `OpenidToken` 中；如果继承了它，请改为继承 `OpenidToken`。

将 token 字符串参数从 `OAuthentication` 的构造函数移动到 `principal` 参数（通常是一个 `OpenidToken`）中。

```java
new OAuthentication<>(new OpenidClaimSet(claims), authorities, tokenString);
```
变为：
```java
new OAuthentication<>(new OpenidToken(new OpenidClaimSet(claims), tokenString), authorities);
```

## `spring-addons-starter-rest`

`SpringAddonsRestClientSupport`、`SpringAddonsWebClientSupport` 和 `ReactiveSpringAddonsWebClientSupport` 已被 `ProxyFactoryBean` 取代：
- `RestClient` 和 `WebClient` bean 定义（或其 builder 的定义）在 bean 注册表后处理阶段注册 => 请删除应用配置中所有显式的 bean 定义，Boot starter 已经自动完成这项工作。
- 将 `@HttpExchange` 服务代理的 bean 定义改为使用 `RestClientHttpExchangeProxyFactoryBean` 或 `WebClientHttpExchangeProxyFactoryBean`

代理属性现在可以为每个 client 单独配置 => 在 YAML 中，将其下移一级（复制到每个需要代理配置的 client 下）。

新增了更多配置选项：
- 一个标志，用于暴露 client builder 而非已构建完成的 client
- 强制指定 bean 名称（默认为配置属性中 kebab-case client ID 的 camelCase 转换形式，当 `expose-builder` 为 `true` 时追加 `Builder` 后缀）
- 设置连接超时和读取超时
- 在 servlet 应用中暴露 `WebClient` 而非默认的 `RestClient`
- 设置 chunk-size（仅适用于 `RestClient`）