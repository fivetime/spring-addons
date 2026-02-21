# 使用 `spring-addons-starter-oidc` 的应用的单元测试与集成测试

`spring-addons-starter-oidc-test` 构建在 [`spring-addons-oauth2-test`](https://github.com/ch4mpy/spring-addons/tree/master/spring-addons-oauth2-test) 之上，新增了以下功能：
- 将 `spring-addons-starter-oidc` 自动配置应用到测试 security context 的注解
- 对 `MockMvc` 和 `WebTestClient` 的轻量封装，使测试请求的编写更加简便

## 准备测试上下文
`@AutoConfigureAddons*` 系列注解可将 `spring-addons-starter-oidc` 自动配置的各个部分导入测试上下文，应与导入自定义 `SecurityConfig`（如有）配合使用。

```java
@WebMvcTest(controllers = GreetingController.class)
@AutoConfigureAddonsWebmvcResourceServerSecurity
@Import({ SecurityConfig.class })
class GreetingControllerTest {

	@Autowired
	MockMvcSupport mockMvc;

	@Test
	@WithAnonymousUser
	void givenRequestIsAnonymous_whenGreet_thenUnauthorized() throws Exception {
		mockMvc.get("/greet")
			.andExpect(status().isUnauthorized());
	}

	@Test
	@WithJwt("ch4mp.json")
	void givenUserIsGrantedWithNice_whenGreet_thenOk() throws Exception {
		mockMvc.get("/greet")
			.andExpect(status().isOk())
			.andExpect(content().string("Hi ch4mp! You are granted with: [NICE, AUTHOR] and can proxy: [chose, machin]."));
	}
}
```

## `MockMvcSupport` 和 `WebTestClientSupport`
这两者分别是 `MockMvc` 和 `WebTestClient` 的轻量封装，新增了以下特性：
- 简单请求（不含请求参数、特殊请求头或 cookie）的快捷方法
- 根据 `com.c4-soft.springaddons.test.web.default-media-type` 测试配置属性设置默认的 `Accept` 和 `Content-Type` 请求头
- 使用**已注册的 message converter** 按 Content-Type 序列化请求体：自动选取与声明的 content-type 匹配、且实际已由（自动）配置注册的 message converter（content-type 可在请求构建时显式指定，也可通过测试配置默认值或 `application/json` 兜底）

例如，以下代码：
```java
@Autowired
private MockMvc mvc;

@Test
public void createEmployeeAPI() throws Exception 
{
  mvc.perform( MockMvcRequestBuilders
	      .post("/employees")
	      .content(asJsonString(new EmployeeDto(null, "firstName4", "lastName4", "email4@mail.com")))
	      .contentType(MediaType.APPLICATION_JSON)
	      .accept(MediaType.APPLICATION_JSON))
      .andExpect(status().isCreated())
      .andExpect(MockMvcResultMatchers.jsonPath("$.employeeId").exists());
}
 
public static String asJsonString(final Object obj) {
    try {
        return new ObjectMapper().writeValueAsString(obj);
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```
可以简化为：
```java
@Autowired
private MockMvcSupport mvc;

@Test
public void createEmployeeAPI() throws Exception
{
    mvc.post(new EmployeeDto(null, "firstName4", "lastName4", "email4@mail.com"), "/employees")
            .andExpect(status().isCreated())
            .andExpect(MockMvcResultMatchers.jsonPath("$.employeeId").exists());
}
```