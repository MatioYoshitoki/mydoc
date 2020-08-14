### 场景:
在使用spring gateway 作为网关， jwt作为权限管理， 使用webflux进行微服务开发的项目中，在网关中添加swagger来对整个项目进行接口的展示与测试。
### 环境:
1. JDK8
2. idea
3. spring cloud H版
4. spring cloud alibaba
5. spring cloud gateway 2.2.2.RELEASE
6. spring boot 2.2.2.RELEASE
7. swagger3.0.0

### 步骤：
1. 在gateway中添加相关依赖。
```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>${swagger.version}</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-spring-webflux</artifactId>
    <version>${swagger.version}</version>
</dependency>
```
2. 由于gateway本身不带任何api，所以需要重写SwaggerProvider来获取微服务的资源，凭借出/v2/api-docs的url。
```java
@Component
@Primary
public class SwaggerProvider implements SwaggerResourcesProvider {
    public static final String API_URI = "/v2/api-docs";
    private final GatewayProperties gatewayProperties;

    public SwaggerProvider(GatewayProperties gatewayProperties) {
        this.gatewayProperties = gatewayProperties;
    }

    @Override
    public List<SwaggerResource> get() {
        List<SwaggerResource> resources = new ArrayList<>();
        // 结合路由配置，只获取有效的route节点
        gatewayProperties.getRoutes()
                .forEach(
                        routeDefinition ->
                                routeDefinition.getPredicates().stream()
                                        .filter(predicateDefinition -> ("Path").equalsIgnoreCase(predicateDefinition.getName()))
                                        .forEach(predicateDefinition -> resources.add(swaggerResource(routeDefinition.getId(),
                                                predicateDefinition.getArgs().get(NameUtils.GENERATED_NAME_PREFIX + "0")
                                                        .replace("/**", API_URI)))));
        return resources;
    }

    private SwaggerResource swaggerResource(String name, String location) {
        SwaggerResource swaggerResource = new SwaggerResource();
        swaggerResource.setName(name);
        swaggerResource.setLocation(location);
        swaggerResource.setSwaggerVersion("2.0");
        return swaggerResource;
    }
}
```
3. 虽然现在会在网关中会返回各个微服务的swagger描述json的获取地址。但是实际上这个地址并不存在，所以我们需要在各个微服务中添加swagger的相关依赖。
在使用生成swagger接口清单的模块中添加以下依赖
```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>${swagger.version}</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-spring-webflux</artifactId>
    <version>${swagger.version}</version>
</dependency>
```
4. 在每个微服务中需要配置相关的SpringFoxConfig
```java
@Configuration
public class SpringFoxConfig {
    @Bean
    public Docket apiDocket() {
        return new Docket(DocumentationType.OAS_30)
                .apiInfo(apiInfo())
                .securitySchemes(Collections.singletonList(securitySchemes()))
                .securityContexts(Collections.singletonList(securityContexts()))
                .select()
                .apis(RequestHandlerSelectors.any())
                .paths(PathSelectors.any())
                .build();
    }

    /**
     * 认证方式使用密码模式
     */
    private SecurityScheme securitySchemes() throws NullPointerException{
        GrantType grantType = new ResourceOwnerPasswordCredentialsGrant("/xxx/xxx");
        return new OAuthBuilder()
                .name("Authorization")
                .grantTypes(Collections.singletonList(grantType))
                .scopes(Arrays.asList(scopes()))
                .build();
    }

    /**
     * 允许认证的scope
     */
    private AuthorizationScope[] scopes() {
        AuthorizationScope authorizationScope = new AuthorizationScope("all", "所有接口");
        AuthorizationScope[] authorizationScopes = new AuthorizationScope[1];
        authorizationScopes[0] = authorizationScope;
        return authorizationScopes;
    }

    /**
     * 设置 swagger2 认证的安全上下文
     */
    private SecurityContext securityContexts() {
        return SecurityContext.builder()
                .securityReferences(Collections.singletonList(new SecurityReference("Authorization", scopes())))
                .forPaths(PathSelectors.any())
                .build();
    }

    public ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("xxxx的API文档")
                .description("用restful风格写接口")
                .termsOfServiceUrl("")
                .version("3.0")
                .build();
    }
```
在上面的配置中，securitySchemes与securityContexts的配置是开启权限认证必备的。swagger原生的权限主要支持2种形式：
1. 直接在页面中配置token，之后作用域中的请求的请求头（也可以设置token添加的位置为request body）中都会带上全局配置的token。
2. 在页面上使用账户密码进行登录，返回的token会作为全局参数，自动添加到作用域的请求的指定位置中。但是swagger的账号密码登录我目前只发现了content-type: application/x-www-form-urlencoded 形式的，暂未发现修改方法，使用如果原先的登录接口不是application/x-www-form-urlencoded的需要添加一个此类型的登录接口专门用于swagger的权限校验。
3. 在生成SecurityScheme方法中的GrantType 用于指定权限校验的地址。

### 完成
