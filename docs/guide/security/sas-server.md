# Ballcat Spring Authorization Server

目前文档内容对标 ballcat v1.1.0 以上版本

## 简介

Spring Authorization Server 是 Spring 推出的一款 OAuth 2.1 协议授权服务器，以下简称为 **SAS**, 旨在简化 OAuth 2.1 协议在 Spring 应用中的使用。OAuth 2.1 协议是 OAuth 2.0 协议的升级版，主要解决了一些安全性和可用性方面的问题，并提供了一些新的特性和扩展。

ballcat 的  `ballcat-spring-security-oauth2-authorization-server` 模块对 SAS 的配置做了进一步的封装，并添加了在 OAuth 2.1 协议中被删除的 password 授权模式支持。



> 注意：此文档仅介绍 Ballcat 对于 SAS 的扩展部分，更多 SAS 自身使用介绍请移步官方文档： [spring-authorization-server doc](https://docs.spring.io/spring-authorization-server/docs/0.4.1/reference/html)



## 使用

### 依赖引入

```xml
	<dependency>
		<groupId>com.hccake</groupId>
		<artifactId>ballcat-spring-security-oauth2-authorization-server</artifactId>
        <version>${lastVersion}</version>
	</dependency>
```



### 数据库表导入

ballcat 默认使用 jdbc 作为 OAuth2RegisteredClient、OAuth2Authorization、OAuth2AuthorizationConsent 的存储方式，在使用之前需要创建对应的表结构。

```sql
/*
IMPORTANT:
    If using PostgreSQL, update ALL columns defined with 'blob' to 'text',
    as PostgreSQL does not support the 'blob' data type.
*/
CREATE TABLE oauth2_registered_client (
    id varchar(100) NOT NULL,
    client_id varchar(100) NOT NULL,
    client_id_issued_at timestamp DEFAULT CURRENT_TIMESTAMP NOT NULL,
    client_secret varchar(200) DEFAULT NULL,
    client_secret_expires_at timestamp DEFAULT NULL,
    client_name varchar(200) NOT NULL,
    client_authentication_methods varchar(1000) NOT NULL,
    authorization_grant_types varchar(1000) NOT NULL,
    redirect_uris varchar(1000) DEFAULT NULL,
    scopes varchar(1000) NOT NULL,
    client_settings varchar(2000) NOT NULL,
    token_settings varchar(2000) NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE oauth2_authorization (
    id varchar(100) NOT NULL,
    registered_client_id varchar(100) NOT NULL,
    principal_name varchar(200) NOT NULL,
    authorization_grant_type varchar(100) NOT NULL,
    authorized_scopes varchar(1000) DEFAULT NULL,
    attributes blob DEFAULT NULL,
    state varchar(500) DEFAULT NULL,
    authorization_code_value blob DEFAULT NULL,
    authorization_code_issued_at timestamp DEFAULT NULL,
    authorization_code_expires_at timestamp DEFAULT NULL,
    authorization_code_metadata blob DEFAULT NULL,
    access_token_value blob DEFAULT NULL,
    access_token_issued_at timestamp DEFAULT NULL,
    access_token_expires_at timestamp DEFAULT NULL,
    access_token_metadata blob DEFAULT NULL,
    access_token_type varchar(100) DEFAULT NULL,
    access_token_scopes varchar(1000) DEFAULT NULL,
    oidc_id_token_value blob DEFAULT NULL,
    oidc_id_token_issued_at timestamp DEFAULT NULL,
    oidc_id_token_expires_at timestamp DEFAULT NULL,
    oidc_id_token_metadata blob DEFAULT NULL,
    refresh_token_value blob DEFAULT NULL,
    refresh_token_issued_at timestamp DEFAULT NULL,
    refresh_token_expires_at timestamp DEFAULT NULL,
    refresh_token_metadata blob DEFAULT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE oauth2_authorization_consent (
    registered_client_id varchar(100) NOT NULL,
    principal_name varchar(200) NOT NULL,
    authorities varchar(1000) NOT NULL,
    PRIMARY KEY (registered_client_id, principal_name)
);
```



### Oauth2 Client 创建

建议使用 junit test 进行 client 创建：

```java
@JdbcTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OAuth2RegisteredClientTest {

	@Autowired
	private JdbcTemplate jdbcTemplate;

	@Test
	void createTestClient() {
		JdbcRegisteredClientRepository jdbcRegisteredClientRepository = new JdbcRegisteredClientRepository(
				jdbcTemplate);

		RegisteredClient test = jdbcRegisteredClientRepository.findByClientId("test");
		if (test == null) {
			RegisteredClient registeredClient = RegisteredClient.withId(UUID.randomUUID().toString())
				.clientId("test")
				.clientSecret("{noop}test")
	     		.clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
				.authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
				.authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
				.authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
				.authorizationGrantType(AuthorizationGrantType.PASSWORD)
				.redirectUri("http://127.0.0.1:8080/authorized")
				.scope("skip_captcha") // 跳过验证码
				.scope("skip_password_decode") // 跳过 AES 密码解密
				.tokenSettings(TokenSettings.builder().accessTokenFormat(OAuth2TokenFormat.REFERENCE).build())  
                .clientSettings(ClientSettings.builder().requireAuthorizationConsent(true).build())
				.build();
			jdbcRegisteredClientRepository.save(registeredClient);

			test = jdbcRegisteredClientRepository.findByClientId("test");
			Assertions.assertNotNull(test);
		}
	}
}

```

也可以使用 sql 直接插入：

```sql
INSERT INTO `oauth2_registered_client` (`id`, `client_id`, `client_id_issued_at`, `client_secret`, `client_secret_expires_at`, `client_name`, `client_authentication_methods`, `authorization_grant_types`, `redirect_uris`, `scopes`, `client_settings`, `token_settings`) VALUES ('4a2ab399-77e1-4d3a-818e-a48410706a3b', 'test', '2023-03-11 16:42:02', '{noop}test', NULL, '4a2ab399-77e1-4d3a-818e-a48410706a3b', 'client_secret_basic', 'refresh_token,client_credentials,password,authorization_code', 'http://127.0.0.1:8080/authorized', 'skip_captcha,skip_password_decode', '{\"@class\":\"java.util.Collections$UnmodifiableMap\",\"settings.client.require-proof-key\":false,\"settings.client.require-authorization-consent\":true}', '{\"@class\":\"java.util.Collections$UnmodifiableMap\",\"settings.token.reuse-refresh-tokens\":true,\"settings.token.id-token-signature-algorithm\":[\"org.springframework.security.oauth2.jose.jws.SignatureAlgorithm\",\"RS256\"],\"settings.token.access-token-time-to-live\":[\"java.time.Duration\",300.000000000],\"settings.token.access-token-format\":{\"@class\":\"org.springframework.security.oauth2.server.authorization.settings.OAuth2TokenFormat\",\"value\":\"reference\"},\"settings.token.refresh-token-time-to-live\":[\"java.time.Duration\",3600.000000000],\"settings.token.authorization-code-time-to-live\":[\"java.time.Duration\",300.000000000]}');
```



## 端点

Spring Security Authorization Server 的默认端点在 `AuthorizationServerSettings` 类下，该类位于 `org.springframework.security.authorization.server` 包中。默认端点包括：

- `/oauth2/authorize`：授权端点，用于向用户展示授权页面并处理用户的授权决策。

- `/oauth2/token`：令牌端点，用于颁发访问令牌和刷新令牌。

- `/oauth2/jwks`：用于获取 JSON Web Key Set（JWKS），以支持对 JWT 签名的验证。

- `/oauth2/revoke`：令牌撤销端点，用于撤销访问令牌和刷新令牌。

- `/oauth2/introspect`：令牌内省端点，用于验证令牌是否有效。

- `/connect/register`:  用于注册 OIDC 的客户端

- `/userinfo`:  用于获取  OIDC 用户信息



ballcat sas 使用了默认的端点，如需自定义端点地址，可以注册 `AuthorizationServerSettings` 覆盖默认行为, 例如：
```java
@Configuration
public class MyConfig {
 	@Bean
	public AuthorizationServerSettings authorizationServerSettings() {
		return AuthorizationServerSettings.builder()
            .authorizationEndpoint("/oauth/toekn") // 修改令牌端点为 /oauth/token
            .build();
	}
}
```



## 配置

配置介绍：

| 配置key                                                      | 描述                                                         | 类型    | 默认值 |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------- | ------ |
| ballcat.security.password-secret-key                         | 密码 AES 加密钥(必须 16 位)，配置后使用 password 授权类型登陆时，传输的 password 值为 AES 加密后的密码 | string  | -      |
| ballcat.security.oauth2.authorizationserver.login-captcha-enabled | 登陆验证码开关                                               | boolean | false  |
| ballcat.security.oauth2.authorizationserver.login-page-enabled | 开启内置的表单登录，授权码模式需要                           | boolean | false  |
| ballcat.security.oauth2.authorizationserver.login-page       | 表单登录页地址，默认使用 security 提供的登录页，地址为：/login | string  | -      |
| ballcat.security.oauth2.authorizationserver.consent-page     | 用户同意授权页面，不配置则使用 SAS 默认提供的                | string  | -      |
| ballcat.security.oauth2.authorizationserver.stateless        | 无状态，默认的表单登陆是有状态的，服务端存储 session，若开启无状态则徐配合对应的 SecurityContextRepository 使用 | boolean | false  |




## 组件

### 密码编码器

密码编码器 `PasswordEncoder` 用于加密密码，以及登陆时的密码验证, ballcat sas 默认提供了一个 `DelegatingPasswordEncoder` 类型的密码编码器,  编码时默认使用 `bcrypt` 算法。

`DelegatingPasswordEncoder` 将密码编码分为两部分，前缀和实际加密后的字符串，前缀用于标识密码的编码方式。在进行密码验证时，DelegatingPasswordEncoder 会自动根据前缀选择对应的加密算法进行验证，无需手动指定。

例如，密码 **a123456** 可存储为：

- `{noop}a123456`: 明文密码
- `{bcrypt}$2a$10$IRAHstZa7wgcrrifF6tpNeSlpvCBe3Tl3GEDQEUxtI/Gxc30OUxlW`： bcrypt 算法
- `{MD5}dc483e80a7a0bd9ef71d8cf973673924`: Md5 算法
- `$2a$10$IRAHstZa7wgcrrifF6tpNeSlpvCBe3Tl3GEDQEUxtI/Gxc30OUxlW`：无前缀，默认会使用 bcrypt 算法进行处理

用户可以通过注册自己的 `PasswordEncoder` 来覆盖默认编码器。



### 令牌响应增强器 

令牌响应增强器 `OAuth2TokenResponseEnhancer` 位于 `org.ballcat.springsecurity.oauth2.server.authorization.web.authentication` 包下，用户可以通过注册自己的增强器来扩展 `oauth2/token ` 令牌端点的响应数据。

> 在 ballcat-admin-core 模块中添加了适用于 ballcat admin 的增强类 `BallcatOAuth2TokenResponseEnhancer`, 可以参考



### 令牌撤销响应处理器

令牌撤销响应处理器 `OAuth2TokenRevocationResponseHandler`, ballcat sas 默认提供的处理器在令牌撤销时会默认发布一个 `LogoutSuccessEvent` 事件，用户可以监听此事件做令牌撤销时的相应处理。

也可以编写自己的处理器并继承 `OAuth2TokenRevocationResponseHandler` ，同时注册到 spring 容器中，即可覆盖默认行为。

> 在 ballcat-admin-core 模块中开启了对 `LogoutSuccessEvent` 事件的监听，并进行了登出日志记录




## 扩展

### 授权服务器 SAS 配置定制器

`Auth2AuthorizationServerConfigurer` 是 SAS 的核心配置类，ballcat 默认对其做了部分扩展，同时又希望用户可以进行对其进行定制化处理，所以提供了

`OAuth2AuthorizationServerConfigurerCustomizer` 类。

```java
/**
 * 对 OAuth2授权服务器配置({@link OAuth2AuthorizationServerConfigurer}) 进行个性化配置的的定制器
 *
 * @author hccake
 */
@FunctionalInterface
public interface OAuth2AuthorizationServerConfigurerCustomizer {
	/**
	 * 对授权服务器配置进行自定义
	 * @param oAuth2AuthorizationServerConfigurer OAuth2AuthorizationServerConfigurer
	 * @param httpSecurity security configuration
	 */
	void customize(OAuth2AuthorizationServerConfigurer oAuth2AuthorizationServerConfigurer, HttpSecurity httpSecurity)
			throws Exception;
}
```



ballcat 默认注册了以下几个定制器：

- `FormLoginConfigurerCustomizer`：用于根据配置文件进行表单登陆相关设置的定制化器
- `OAuth2ResourceOwnerPasswordConfigurerCustomizer`：用于支持 OAuth2.1 密码模式的定制化器
- `OAuth2TokenResponseEnhanceConfigurerCustomizer`：用于支持 OAuth2 令牌端点响应增强配置
- `OAuth2TokenRevocationEndpointConfigurerCustomizer`：用于支持 OAuth2 撤销令牌端点响应增强的配置



用户可以注册自己的定制器到 spring 容器中，即可完成 sas 配置的定制，也可以通过编写自定义类并继承 ballcat 提供的默认定制器，以达到覆盖默认配置的效果。



### 授权服务器的 HttpSecurity 的扩展配置器

`OAuth2AuthorizationServerExtensionConfigurer` 是对授权服务器更为深入的定制，可以脱离 `OAuth2AuthorizationServerConfigurer` 对 `HttpSecurity` 进行个性化拓展，其优先级在 SAS 配置之后，所以可以覆盖 `OAuth2AuthorizationServerConfigurer` 的一些配置

```java
/**
 * 对 OAuth2 授权服务器的 SecurityConfigurer 进行扩展的配置类
 *
 * @author hccake
 */
public abstract class OAuth2AuthorizationServerExtensionConfigurer<C extends OAuth2AuthorizationServerExtensionConfigurer<C, H>, H extends HttpSecurityBuilder<H>>
		extends AbstractHttpConfigurer<C, H> {
}

```

ballcat 默认提供了以下几个扩展配置器：

- `OAuth2LoginCaptchaConfigurer`：

  password 模式下登录验证码校验扩展，当开启验证码时注册，拥有 `skip_captcha` scope 的客户端可以跳过登录验证码验证

- `OAuth2LoginPasswordDecoderConfigurer`：

  password 模式下登陆时的密码解密配置，当配置了 AES 加密密钥时注册，拥有 `skip_password_decode` scope 的客户端可以跳过密码 AES 解密步骤（即登陆时可以传递明文）

用户同样可以定制自己的扩展配置器。