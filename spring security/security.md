# Safe Methods Must be Idempotent

In order for either protection against CSRF to work, the application must ensure that "safe" HTTP methods are idempotent. This means that requests with the HTTP method GET, HEAD, OPTIONS, and TRACE should not change the state of the application.

# Synchronized Token Pattern

The predominant and most comprehensive way to protect against CSRF attacks is to use the Synchronizer Token Pattern. This solution is to ensure that each HTTP request requires, in addition to our session cookie, a secure random generated value called a CSRF token must be present in the HTTP request.

> 抵御CSRF攻击的最主要，最全面的方法是使用同步器令牌模式。 该解决方案是为了确保每个HTTP请求除了我们的会话cookie外，还必须在HTTP请求中包含一个安全的，随机生成的值，称为CSRF令牌。

When an HTTP request is submitted, the server must look up the expected CSRF token and compare it against the actual CSRF token in the HTTP request. If the values do not match, the HTTP request should be rejected.

> 提交HTTP请求时，服务器必须查找预期的CSRF令牌，并将其与HTTP请求中的实际CSRF令牌进行比较。 如果值不匹配，则应拒绝HTTP请求。

The key to this working is that the actual CSRF token should be in a part of the HTTP request that is not automatically included by the browser. For example, requiring the actual CSRF token in an HTTP parameter or an HTTP header will protect against CSRF attacks. Requiring the actual CSRF token in a cookie does not work because cookies are automatically included in the HTTP request by the browser.

> 这项工作的关键在于，实际的CSRF令牌应该位于浏览器不会自动包含的HTTP请求的一部分中。 例如，在HTTP参数或HTTP标头中要求实际的CSRF令牌将防止CSRF攻击。 在Cookie中要求实际CSRF令牌无效，因为浏览器会自动将Cookie包含在HTTP请求中

We can relax the expectations to only require the actual CSRF token for each HTTP request that updates state of the application. For that to work, our application must ensure that safe HTTP methods are idempotent. This improves usability since we want to allow linking to our website using links from external sites. Additionally, we do not want to include the random token in HTTP GET as this can cause the tokens to be leaked.

> 我们可以放宽对每个更新应用程序状态的HTTP请求仅要求实际CSRF令牌的期望。 为此，我们的应用程序必须确保安全的HTTP方法是幂等的。 由于我们希望允许使用外部站点的链接来链接到我们的网站，因此可以提高可用性。 此外，我们不想在HTTP GET中包含随机令牌，因为这可能导致令牌泄漏。

Let’s take a look at how our example would change when using the Synchronizer Token Pattern. Assume the actual CSRF token is required to be in an HTTP parameter named _csrf. Our application’s transfer form would look like:

> 让我们看一下使用“同步器令牌模式”时示例如何变化。 假设实际的CSRF令牌必须位于名为_csrf的HTTP参数中。 我们应用程序的转帐表格如下：

```html
<form method="post"
    action="/transfer">
<input type="hidden"
    name="_csrf"
    value="4bfd1575-3ad1-4d21-96c7-4ef2d9f86721"/>
<input type="text"
    name="amount"/>
<input type="text"
    name="routingNumber"/>
<input type="hidden"
    name="account"/>
<input type="submit"
    value="Transfer"/>
</form>

```

The form now contains a hidden input with the value of the CSRF token. External sites cannot read the CSRF token since the same origin policy ensures the evil site cannot read the response.

> 现在，该表单包含具有CSRF令牌值的隐藏输入。 外部站点无法读取CSRF令牌，因为相同的来源策略可确保恶意站点无法读取响应。

The corresponding HTTP request to transfer money would look like this:

```html
POST /transfer HTTP/1.1
Host: bank.example.com
Cookie: JSESSIONID=randomid
Content-Type: application/x-www-form-urlencoded
amount=100.00&routingNumber=1234&account=9876&_csrf=4bfd1575-3ad1-4d21-96c7-4ef2d9f86721
```

You will notice that the HTTP request now contains the _csrf parameter with a secure random value. The evil website will not be able to provide the correct value for the _csrf parameter (which must be explicitly provided on the evil website) and the transfer will fail when the server compares the actual CSRF token to the expected CSRF token.

> 您会注意到，HTTP请求现在包含带有安全随机值的_csrf参数。 恶意网站将无法为_csrf参数提供正确的值（必须在邪恶网站上明确提供），并且当服务器将实际CSRF令牌与预期CSRF令牌进行比较时，传输将失败。

# SameSite Attribute

An emerging way to protect against CSRF Attacks is to specify the SameSite Attribute on cookies. A server can specify the SameSite attribute when setting a cookie to indicate that the cookie should not be sent when coming from external sites.

一种防止CSRF攻击的新兴方法是在cookie上指定SameSite属性。 设置cookie时，服务器可以指定SameSite属性，以指示从外部站点发出时不应发送该cookie。

> Spring Security does not directly control the creation of the session cookie, so it does not provide support for the SameSite attribute. Spring Session provides support for the SameSite attribute in servlet based applications. Spring Framework’s CookieWebSessionIdResolver provides out of the box support for the SameSite attribute in WebFlux based applications.
>
> Spring Security不直接控制会话cookie的创建，因此不提供对SameSite属性的支持。 Spring Session在基于servlet的应用程序中为SameSite属性提供支持。 Spring Framework的CookieWebSessionIdResolver为基于WebFlux的应用程序中的SameSite属性提供了开箱即用的支持。

An example, HTTP response header with the SameSite attribute might look like:

```html
Set-Cookie: JSESSIONID=randomid; Domain=bank.example.com; Secure; HttpOnly; SameSite=Lax
```

Valid values for the SameSite attribute are:

- Strict - when specified any request coming from the same-site will include the cookie. Otherwise, the cookie will not be included in the HTTP request.
- 严格-指定后，来自同一站点的任何请求都将包含cookie。 否则，cookie将不会包含在HTTP请求中。
- Lax - when specified cookies will be sent when coming from the same-site or when the request comes from top-level navigations and the method is idempotent. Otherwise, the cookie will not be included in the HTTP request.
- 松懈-当来自同一个站点的特定cookie或当请求来自顶级导航且方法是幂等时，将发送该cookie。 否则，cookie将不会包含在HTTP请求中。

Let’s take a look at how our example could be protected using the SameSite attribute. The bank application can protect against CSRF by specifying the SameSite attribute on the session cookie.

> 让我们看一下如何使用SameSite属性保护我们的示例。 银行应用程序可以通过在会话cookie上指定SameSite属性来防御CSRF。

在会话cookie上设置SameSite属性后，浏览器将继续发送JSESSIONID cookie和来自银行网站的请求。 但是，浏览器将不再发送带有来自邪恶网站的传输请求的JSESSIONID cookie。 由于该会话不再存在于来自邪恶网站的传输请求中，因此可以保护应用程序免受CSRF攻击。

使用SameSite属性防御CSRF攻击时，应注意一些重要注意事项。

将SameSite属性设置为Strict可以提供更强的防御能力，但会使用户感到困惑。 考虑一个保持登录到https://social.example.com托管的社交媒体网站的用户。 用户在https://email.example.org上收到一封电子邮件，其中包含指向社交媒体网站的链接。 如果用户单击该链接，则他们理应期望获得对社交媒体网站的身份验证。 但是，如果SameSite属性为Strict，则不会发送cookie，因此不会对用户进行身份验证。

另一个明显的考虑因素是，为了使SameSite属性能够保护用户，浏览器必须支持SameSite属性。 大多数现代浏览器都支持SameSite属性。 但是，可能仍未使用较旧的浏览器。

因此，通常建议将SameSite属性用作深度防御，而不是针对CSRF攻击的唯一防护。

什么时候应该使用CSRF保护？ 我们的建议是对普通用户可能由浏览器处理的任何请求使用CSRF保护。 如果仅创建非浏览器客户端使用的服务，则可能需要禁用CSRF保护。

一个常见的问题是“我需要保护由javascript发出的JSON请求吗？” 简短的答案是，这取决于, 但是，您必须非常小心，因为有些CSRF漏洞会影响JSON请求。 例如，恶意用户可以使用以下格式使用JSON创建CSRF：

```html
<form action="https://bank.example.com/transfer" method="post" enctype="text/plain">
    <input name='{"amount":100,"routingNumber":"evilsRoutingNumber","account":"evilsAccountNumber",
 "ignore_me":"' value='test"}' type='hidden'>
    <input type="submit"
        value="Win Money!"/>
</form>

```

This will produce the following JSON structure

```json
{ "amount": 100,
"routingNumber": "evilsRoutingNumber",
"account": "evilsAccountNumber",
"ignore_me": "=test"
}

```

如果应用程序未验证Content-Type，则该应用程序将被暴露。 根据设置的不同，仍然可以通过更新URL后缀以.json结尾来利用验证内容类型的Spring MVC应用程序，如下所示：

```html
<form action="https://bank.example.com/transfer.json" method="post" enctype="text/plain">
    <input name='{"amount":100,"routingNumber":"evilsRoutingNumber","account":"evilsAccountNumber",
 "ignore_me":"' value='test"}' type='hidden'>
    <input type="submit"
        value="Win Money!"/>
</form>
```

为了防止伪造登录请求，应保护HTTP请求中的登录免受CSRF攻击。 必须防止伪造登录请求，以使恶意用户无法读取受害者的敏感信息。 攻击通过以下方式执行：

恶意用户使用恶意用户的凭据执行CSRF登录。 现在，将受害者验证为恶意用户。

然后，恶意用户诱骗受害者访问受感染的网站并输入敏感信息

该信息与恶意用户的帐户相关联，因此恶意用户可以使用自己的凭据登录并查看vicitim的敏感信息



# Spring Security项目结构

##  Core — spring-security-core.jar

该模块包含核心身份验证和访问控制类和接口，远程支持和基本配置API。 使用Spring Security的任何应用程序都需要它。 它支持独立的应用程序，远程客户端，方法（服务层）安全性和JDBC用户配置。 它包含以下顶级程序包：

-  org.springframework.security.core
- org.springframework.security.access
- org.springframework.security.authentication
- org.springframework.security.provisioning

##  Remoting — spring-security-remoting.jar

该模块提供了与Spring Remoting的集成。 除非您正在编写使用Spring Remoting的远程客户端，否则您不需要这样做。 主要包是

- org.springframework.security.remoting.

##  Web — spring-security-web.jar

该模块包含过滤器和相关的Web安全基础结构代码。 它包含任何与Servlet API相关的内容。 如果需要Spring Security Web认证服务和基于URL的访问控制，则需要它。 主要包是：

- org.springframework.security.web

##  Config — spring-security-config.jar

该模块包含安全命名空间解析代码和Java配置代码。 如果您使用Spring Security XML命名空间进行配置或Spring Security的Java配置支持，则需要它。 主要包是

- org.springframework.security.config

**这些类都不打算直接在应用程序中使用。**

##  LDAP — spring-security-ldap.jar

此模块提供LDAP身份验证和供应代码。 如果您需要使用LDAP认证或管理LDAP用户条目，则为必填项。 顶级软件包是

- org.springframework.security.ldap.

##  OAuth 2.0 Core — spring-security-oauth2-core.jar

包含为OAuth 2.0授权框架和OpenID Connect Core 1.0提供支持的核心类和接口。 使用OAuth 2.0或OpenID Connect Core 1.0的应用程序（例如客户端，资源服务器和授权服务器）需要它。 顶级软件包是

-  org.springframework.security.oauth2.core.

##  OAuth 2.0 Client — spring-security-oauth2-client.jar
包含Spring Security对OAuth 2.0授权框架和OpenID Connect Core 1.0的客户端支持。 使用OAuth 2.0登录或OAuth客户端支持的应用程序需要使用它。 顶级软件包是

- org.springframework.security.oauth2.client.

## OAuth 2.0 JOSE — spring-security-oauth2-jose.jar

包含Spring Security对JOSE（Javascript Object Spring and Encryption）框架的支持。 JOSE框架旨在提供一种在各方之间安全地转移索赔的方法。 它是根据一系列规范构建的：

JSON Web Token (JWT)

JSON Web Signature (JWS)

JSON Web Encryption (JWE)

JSON Web Key (JWK)

It contains the following top-level packages:

- org.springframework.security.oauth2.jwt
-  org.springframework.security.oauth2.jose

##  OAuth 2.0 Resource Server — spring-security-oauth2-resource-server.jar
spring-security-oauth2-resource-server.jar包含Spring Security对OAuth 2.0资源服务器的支持。 它用于通过OAuth 2.0承载令牌保护API。 

-  org.springframework.security.oauth2.server.resource.

##  ACL — spring-security-acl.jar

该模块包含专门的域对象ACL实现。 它用于将安全性应用于应用程序中的特定域对象实例。 顶级软件包是

org.springframework.security.acls

## CAS — spring-security-cas.jar

该模块包含Spring Security的CAS客户端集成。 如果要对CAS单点登录服务器使用Spring Security Web认证，则应该使用它。 顶级软件包是

org.springframework.security.cas.

## OpenID — spring-security-openid.jar

该模块包含OpenID Web身份验证支持。 它用于根据外部OpenID服务器对用户进行身份验证。 顶级包是org.springframework.security.openid。 它需要OpenID4Java。

##  Test — spring-security-test.jar

This module contains support for testing with Spring Security



#  Technical Overview

## Runtime Environment

Spring Security 5.2.2.RELEASE需要Java 8 Runtime Environment或更高版本。 由于Spring Security旨在以独立的方式运行，因此无需在Java运行时环境中放置任何特殊的配置文件。 特别是，不需要配置特殊的Java身份验证和授权服务（JAAS）策略文件或将Spring Security放置在公共类路径位置。
同样，如果您使用的是EJB容器或Servlet容器，则无需在任何地方放置任何特殊的配置文件，也无需在服务器类加载器中包含Spring Security。 所有必需的文件将包含在您的应用程序中。
这种设计提供了最大的部署时间灵活性，因为您只需将目标工件（JAR，WAR或EAR）从一个系统复制到另一个系统即可立即使用。

## Core Components

从Spring Security 3.0开始，spring-security-core jar的内容减少到最低限度。 它不再包含与Web应用程序安全性，LDAP或名称空间配置有关的任何代码。 我们将在这里介绍您在核心模块中找到的一些Java类型。 它们代表了框架的组成部分，因此，如果您需要超越简单的名称空间配置，那么即使实际上不需要直接与它们进行交互，也必须了解它们的含义，这一点很重要。

## SecurityContextHolder, SecurityContext and Authentication Objects

最基本的对象是SecurityContextHolder。 我们在这里存储应用程序当前安全上下文的详细信息，其中包括当前使用该应用程序的主体的详细信息。 默认情况下，SecurityContextHolder使用ThreadLocal存储这些详细信息，这意味着即使未将安全上下文作为参数显式传递给这些方法，安全上下文也始终可用于同一执行线程中的方法。 如果在处理当前委托人的请求后要清除线程，则以这种方式使用ThreadLocal是非常安全的。 当然，Spring Security会自动为您解决此问题，因此无需担心。

由于某些应用程序使用线程的特定方式，因此它们并不完全适合使用ThreadLocal。 例如，Swing客户端可能希望Java虚拟机中的所有线程都使用相同的安全上下文。 可以在启动时为SecurityContextHolder配置策略，以指定您希望如何存储上下文。 对于独立应用程序，您将使用SecurityContextHolder.MODE_GLOBAL策略。 其他应用程序可能希望让安全线程产生的线程也采用相同的安全身份。 这是通过使用SecurityContextHolder.MODE_INHERITABLETHREADLOCAL实现的。 您可以通过两种方式从默认的SecurityContextHolder.MODE_THREADLOCAL更改模式。 第一个是设置系统属性，第二个是在SecurityContextHolder上调用静态方法。 大多数应用程序不需要更改默认值，但是如果需要，请查看JavaDoc for SecurityContextHolder以了解更多信息。

### Obtaining information about the current user

在SecurityContextHolder内部，我们存储了当前与应用程序交互的主体的详细信息。 Spring Security使用Authentication对象来表示此信息。 通常，您无需自己创建Authentication对象，但是用户查询Authentication对象相当普遍。 您可以在应用程序中的任何位置使用以下代码块来获取当前经过身份验证的用户的名称，例如：

```java
Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
if (principal instanceof UserDetails) {
String username = ((UserDetails)principal).getUsername();
} else {
String username = principal.toString();

```

调用getContext（）返回的对象是SecurityContext接口的一个实例。 这是保存在线程本地存储中的对象。 正如我们将在下面看到的那样，Spring Security中的大多数身份验证机制都将UserDetails实例返回为主体。

### The UserDetailsService

上述代码片段中需要注意的另一项内容是，您可以从Authentication对象中获取主体。 主体仅仅是一个对象。 在大多数情况下，可以将其转换为UserDetails对象。 UserDetails是Spring Security中的核心接口。 它代表一个主体，但以一种可扩展的和特定于应用程序的方式。 将UserDetails视为您自己的用户数据库和SecurityContextHolder内部Spring Security所需的适配器。 作为您自己的用户数据库中某些内容的表示，通常会将UserDetails转换为应用程序提供的原始对象，以便可以调用特定于业务的方法（如getEmail（），getEmployeeNumber（）等）。

```java
/**
 * @author ys
 **/
@Data
public class User implements UserDetails {
    private Long id;
    private String username;
    private String password;
    private String roles;
    private boolean enable;
    private List<GrantedAuthority> authorities;

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return authorities;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return this.enable;
    }
}
```

现在，您可能想知道，那么什么时候提供UserDetails对象？ 我怎么做？ 我以为您说的是声明性的，我不需要编写任何Java代码-有什么用？ 简短的答案是有一个名为UserDetailsService的特殊接口。 此接口上的唯一方法接受基于字符串的用户名参数，并返回UserDetails：

```java
UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
```

这是在Spring Security中为用户加载信息的最常见方法，只要需要有关用户的信息，就会在整个框架中使用它。

身份验证成功后，将使用UserDetails来构建存储在SecurityContextHolder中的Authentication对象（下面将对此进行详细介绍）。 好消息是，我们提供了许多UserDetailsService实现，包括一个使用内存映射（InMemoryDaoImpl）和另一个使用JDBC（JdbcDaoImpl）的实现。 但是，大多数用户倾向于编写自己的应用程序，而其实现往往只是位于代表其员工，客户或应用程序其他用户的现有数据访问对象（DAO）之上。 请记住，始终可以使用上述代码片段从SecurityContextHolder中获取您的UserDetailsService返回的任何内容。

关于UserDetailsService经常会有一些困惑。 它纯粹是用于用户数据的DAO，除了将数据提供给框架内的其他组件外，不执行其他功能。 特别是，它不对用户进行身份验证，这由AuthenticationManager完成。 在许多情况下，如果您需要自定义身份验证过程，则直接实现AuthenticationProvider更有意义。

![](D:\kkb\github\daily_notes\spring security\security.assets\验证过程.png)

### GrantedAuthority

除主体外，Authentication提供的另一个重要方法是getAuthorities（）。 此方法提供了GrantedAuthority对象数组。 毫不奇怪，GrantedAuthority是授予主体的权限。 此类权限通常是“角色”，例如ROLE_ADMINISTRATOR或ROLE_HR_SUPERVISOR。 稍后将这些角色配置为Web授权，方法授权和域对象授权。 Spring Security的其他部分能够解释这些权限，并希望它们存在。 GrantedAuthority对象通常由UserDetailsService加载。

通常，GrantedAuthority对象是应用程序范围的权限。 它们不特定于给定的域对象。 因此，您不太可能具有GrantedAuthority来表示对Employee对象编号54的许可，因为如果有成千上万的此类授权，您很快就会用完内存（或者至少导致应用程序花费很长时间） 时间来认证用户）。 当然，Spring Security是专门为满足这一通用要求而设计的，但您可以为此目的使用项目的域对象安全功能。

```java
/**
 * @author ys
 **/
@Service
public class MyUserDetailsService implements UserDetailsService {
    @Autowired
    UserMapper mapper;
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        //从数据库尝试读取该用户
        User user = mapper.findUserByUsername(username);
        //用户不存在，跑出异常
        if(user == null){
            throw new UsernameNotFoundException("用户不存在");
        }
        //将数据库形式的roles解析为UserDetails的权限集
        //AuthorityUtils.commaSeparatedStringToAuthorityList是spring Security提供的
        //该方法用于将逗号隔开的权限集字符串切割成可用权限对象列表
        //当然也可以自己实现，如用分号来隔开等，参考generateAuthorities
        user.setAuthorities(AuthorityUtils.commaSeparatedStringToAuthorityList(user.getRoles()));
        return user;
    }
    //自行实现权限转换
    //其中，SimpleGrantedAuthority是GrantedAuthority的一个实现类
    private List<GrantedAuthority> generateAuthorities(String roles){
        List<GrantedAuthority> authorities = new ArrayList<>();
        if(StringUtils.isNullOrEmpty(roles)){
            return new ArrayList<>();
        }
        Arrays.stream(roles.split(";")).forEach(s -> authorities.add(new SimpleGrantedAuthority(s)));
        return authorities;
    }
}
```

### Summary

回顾一下，到目前为止，我们已经看到了Spring Security的主要组成部分：

SecurityContextHolder, to provide access to the SecurityContext

SecurityContext, to hold the Authentication and possibly request-specific security information.

Authentication, to represent the principal in a Spring Security-specific manner

GrantedAuthority, to reflect the application-wide permissions granted to a principal.

 UserDetails, to provide the necessary information to build an Authentication object from your application’s DAOs or other source of security data.

 UserDetailsService, to create a UserDetails when passed in a String-based username (or certificate ID or the like).

现在，您已经了解了这些重复使用的组件，下面让我们仔细看看身份验证过程。

### Authentication

Spring Security可以参与许多不同的身份验证环境。 虽然我们建议人们使用Spring Security进行身份验证，而不是与现有的容器管理的身份验证集成，但是仍然支持它-与您自己的专有身份验证系统集成一样。

What is authentication in Spring Security?

让我们考虑一个大家都熟悉的标准身份验证方案。

1. 提示用户使用用户名和密码登录
2. 系统（成功）验证用户名的密码正确。
3. 获取该用户的上下文信息（他们的角色列表等）。
4. 为用户建立了安全上下文
5. 用户可能会继续执行某些操作，该操作可能会受到访问控制机制的保护，该访问控制机制会根据当前安全上下文信息检查该操作所需的权限。

前四项构成了身份验证过程，因此我们将看看它们在Spring Security中是如何发生的。

1. 获取用户名和密码，并将其组合到UsernamePasswordAuthenticationToken实例（我们在前面看到的Authentication接口的实例）中。
2. 令牌将传递到AuthenticationManager实例进行验证。
3. 身份验证管理器在身份验证成功后返回完全填充的身份验证实例。
4. 通过调用SecurityContextHolder.getContext（）。setAuthentication（…）并返回返回的身份验证对象来建立安全上下文。

从那时起，将认为用户已通过身份验证。 让我们来看一些代码作为示例。

```java
import org.springframework.security.authentication.*;
import org.springframework.security.core.*;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
public class AuthenticationExample {
private static AuthenticationManager am = new SampleAuthenticationManager();
public static void main(String[] args) throws Exception {
    BufferedReader in = new BufferedReader(new InputStreamReader(System.in));
    while(true) {
    System.out.println("Please enter your username:");
    String name = in.readLine();
    System.out.println("Please enter your password:");
    String password = in.readLine();
    try {
        Authentication request = new UsernamePasswordAuthenticationToken(name, password);
        Authentication result = am.authenticate(request);
        SecurityContextHolder.getContext().setAuthentication(result);
        break;
    } catch(AuthenticationException e) {
        System.out.println("Authentication failed: " + e.getMessage());
    }
    }
    System.out.println("Successfully authenticated. Security context contains: " +
            SecurityContextHolder.getContext().getAuthentication());
}
}
class SampleAuthenticationManager implements AuthenticationManager {
static final List<GrantedAuthority> AUTHORITIES = new ArrayList<GrantedAuthority>();
static {
    AUTHORITIES.add(new SimpleGrantedAuthority("ROLE_USER"));
}
public Authentication authenticate(Authentication auth) throws AuthenticationException {
    if (auth.getName().equals(auth.getCredentials())) {
    return new UsernamePasswordAuthenticationToken(auth.getName(),
        auth.getCredentials(), AUTHORITIES);
    }
    throw new BadCredentialsException("Bad Credentials");
}
}
```

在这里，我们编写了一个小程序，要求用户输入用户名和密码并执行上述顺序。 我们在此处实现的AuthenticationManager将对用户名和密码相同的所有用户进行身份验证。 它为每个用户分配一个角色。 上面的输出将是这样的：

```
Please enter your username:
bob
Please enter your password:
password
Authentication failed: Bad Credentials
Please enter your username:
bob
Please enter your password:
bob
Successfully authenticated. Security context contains: \
org.springframework.security.authentication.UsernamePasswordAuthenticationToken@441d0230: \
Principal: bob; Password: [PROTECTED]; \
Authenticated: true; Details: null; \
Granted Authorities: ROLE_USER

```

请注意，您通常不需要编写任何此类代码。 该过程通常在内部进行，例如在Web身份验证过滤器中。 我们刚刚在此处添加了代码，以表明在Spring Security中真正构成身份验证的问题有一个非常简单的答案。 当SecurityContextHolder包含完全填充的Authentication对象时，将对用户进行身份验证。

### Setting the SecurityContextHolder Contents Directly

实际上，Spring Security并不介意如何将Authentication对象放入SecurityContextHolder中。 唯一关键的要求是，SecurityContextHolder包含一个Authentication，它代表AbstractSecurityInterceptor（我们将在后面详细介绍）需要授权用户操作之前的主体。

您可以（而且很多用户都可以）编写自己的过滤器或MVC控制器，以提供与不基于Spring Security的身份验证系统的互操作性。例如，您可能正在使用容器管理的身份验证，该身份验证可通过ThreadLocal或JNDI位置使当前用户可用。或者，您可能在拥有传统专有身份验证系统的公司工作，这是您无法控制的公司“标准”。在这种情况下，让Spring Security正常工作并仍然提供授权功能非常容易。您所需要做的就是编写一个过滤器（或等效过滤器），该过滤器从某个位置读取第三方用户信息，构建一个Spring Security特定的Authentication对象，并将其放入SecurityContextHolder中。在这种情况下，您还需要考虑通常由内置身份验证基础结构自动处理的事情。例如，在将响应写入客户端之前，您可能需要先创建HTTP会话以缓存请求之间的上下文。

如果您想知道在实际示例中如何实现AuthenticationManager，我们将在“核心服务”一章中进行介绍。

### Authentication in a Web Application

现在，让我们探讨一下在Web应用程序中使用Spring Security的情况（未启用web.xml安全性）。 如何认证用户并建立安全上下文？

考虑一个典型的Web应用程序的身份验证过程：

1. 您访问主页，然后单击链接。
2. 请求发送到服务器，服务器确定您已请求受保护的资源。
3. 由于您目前尚未通过身份验证，因此服务器会发回响应，指示您必须进行身份验证。 响应将是HTTP响应代码，或重定向到特定网页。
4. 根据身份验证机制，您的浏览器将重定向到特定网页，以便您可以填写表格，或者浏览器将以某种方式检索您的身份（通过BASIC身份验证对话框，cookie，X.509证书等）。 ）。
5. 浏览器将响应发送回服务器。 这将是包含您填写的表单内容的HTTP POST或包含身份验证详细信息的HTTP标头。
6. 接下来，服务器将决定所提供的凭据是否有效。 如果有效，则将进行下一步。 如果它们无效，通常会要求您的浏览器再试一次（因此您返回到上面的第二步）。
7. 您尝试引起身份验证过程的原始请求将被重试。 希望您已经获得足够授权的身份验证，可以访问受保护的资源。 如果您具有足够的访问权限，则请求将成功。 否则，您将收到HTTP错误代码403，表示“禁止”。

Spring Security具有负责上述大多数步骤的不同类。 主要参与者（按照使用顺序）是ExceptionTranslationFilter，AuthenticationEntryPoint和“身份验证机制”，它们负责调用上一节中看到的AuthenticationManager。

### ExceptionTranslationFilter

ExceptionTranslationFilter是Spring Security过滤器，它负责检测抛出的任何Spring Security异常。 此类异常通常由AbstractSecurityInterceptor抛出，AbstractSecurityInterceptor是授权服务的主要提供者。 我们将在下一节中讨论AbstractSecurityInterceptor，但是现在我们只需要知道它会产生Java异常，并且对HTTP或对主体进行身份验证一无所知。 而是由ExceptionTranslationFilter提供此服务，具体负责返回错误代码403（如果主体已通过身份验证，因此仅缺少足够的访问权限-按照上面的第七步），或者启动AuthenticationEntryPoint（如果主体尚未通过身份验证并且 因此我们需要开始第三步）。

### AuthenticationEntryPoint

AuthenticationEntryPoint负责上述列表中的第三步。 可以想象，每个Web应用程序都会有一个默认的身份验证策略（嗯，可以像配置Spring Security中的几乎所有其他功能一样配置它，但是现在让我们保持简单）。 每个主要的身份验证系统都有其自己的AuthenticationEntryPoint实现，该实现通常执行步骤3中描述的操作之一。

### Authentication Mechanism

一旦浏览器提交了身份验证凭据（作为HTTP表单帖子或HTTP标头），服务器上就需要有一些东西来“收集”这些身份验证详细信息。 到目前为止，我们位于上述列表的第六步。 在Spring Security中，我们有一个特殊的名称，用于从用户代理（通常是Web浏览器）收集身份验证详细信息的功能，将其称为“身份验证机制”。 示例是基于表单的登录和基本身份验证。 一旦从用户代理收集了身份验证详细信息，便会建立一个身份验证“请求”对象，然后将其呈现给AuthenticationManager。

身份验证机制收到完全填充的Authentication对象后，它将认为请求有效，将Authentication放入SecurityContextHolder，并导致重试原始请求（上面的第七步）。 另一方面，如果AuthenticationManager拒绝了该请求，则认证机制将要求用户代理重试（上述第二步）。

### Storing the SecurityContext between requests

根据应用程序的类型，可能需要制定一种策略来存储用户操作之间的安全上下文。 在典型的Web应用程序中，用户登录一次，然后通过其会话ID进行标识。 服务器缓存持续时间会话的主体信息。 在Spring Security中，在请求之间存储SecurityContext的责任落在SecurityContextPersistenceFilter上，它默认将上下文存储为HTTP请求之间的HttpSession属性。 它将每个请求的上下文还原到SecurityContextHolder，并且至关重要的是，在请求完成时清除SecurityContextHolder。 为了安全起见，您不应直接与HttpSession进行交互。 这样做根本没有道理-始终使用SecurityContextHolder代替。

许多其他类型的应用程序（例如，无状态RESTful Web服务）不使用HTTP会话，并且将在每个请求上重新进行身份验证。 但是，将SecurityContextPersistenceFilter包含在链中以确保在每次请求后清除SecurityContextHolder仍然很重要。

> 在单个会话中接收并发请求的应用程序中，相同的SecurityContext实例将在线程之间共享。 即使正在使用ThreadLocal，它也是从HttpSession中为每个线程检索的实例。 如果您希望临时更改运行线程的上下文，则可能会产生影响。 如果仅使用SecurityContextHolder.getContext（），并在返回的上下文对象上调用setAuthentication（anAuthentication），则Authentication对象将在共享同一SecurityContext实例的所有并发线程中更改。 您可以自定义SecurityContextPersistenceFilter的行为，以为每个请求创建一个全新的SecurityContext，以防止一个线程中的更改影响另一个线程。 或者，您可以在临时更改上下文的位置创建一个新实例。 方法SecurityContextHolder.createEmptyContext（）始终返回一个新的上下文实例。

## Access-Control (Authorization) in Spring Security

在Spring Security中负责做出访问控制决策的主要接口是AccessDecisionManager。 它具有一个决策方法，该方法采用一个表示请求访问的主体的Authentication对象，一个“安全对象”（请参见下文）以及适用于该对象的安全元数据属性列表（例如，访问该对象所需的角色列表） 被授予）。

### Security and AOP Advice

如果您熟悉AOP，则会知道有不同类型的advice：before，after，throws和around。 around advice非常有用，因为advisor可以选择是否继续进行方法调用，是否修改响应以及是否引发异常。 Spring Security提供了有关方法调用以及Web请求的around advice。 我们使用Spring的标准AOP支持为方法调用提供了一个around advice，并使用标准的过滤器为Web请求提供了一个around advice。

对于不熟悉AOP的人来说，要了解的关键是Spring Security可以帮助您保护方法调用以及Web请求。 大多数人都对在其服务层上确保方法调用感兴趣。 这是因为服务层是大多数业务逻辑驻留在当前Java EE应用程序中的地方。 如果您只需要在服务层中确保方法调用的安全，那么Spring的标准AOP就足够了。 如果您需要直接保护域对象，则可能会发现AspectJ值得考虑。

您可以选择使用AspectJ或Spring AOP执行方法授权，也可以选择使用过滤器执行Web请求授权。 您可以同时使用零，一，二或三种方法。 主流用法是执行一些Web请求授权，并在服务层上执行一些Spring AOP方法调用授权。

### Secure Objects and the AbstractSecurityInterceptor

那么，什么是“安全对象”？ Spring Security使用该术语来指代可以对其应用安全性（例如授权决策）的任何对象。 最常见的示例是方法调用和Web请求。

每种受支持的安全对象类型都有其自己的拦截器类，该类是AbstractSecurityInterceptor的子类。 重要的是，在调用AbstractSecurityInterceptor时，如果主体已通过身份验证，则SecurityContextHolder将包含有效的Authentication。

AbstractSecurityInterceptor提供了用于处理安全对象请求的一致工作流，通常：

1. 查找与当前请求关联的“配置属性”
2. 将安全对象，当前的身份验证和配置属性提交到AccessDecisionManager以进行授权决策
3. （可选）更改在其下进行调用的身份验证
4. 允许进行安全对象调用（假设已授予访问权限）
5. 一旦调用返回，则调用AfterInvocationManager（如果已配置）。 如果调用引发了异常，则将不会调用AfterInvocationManager。

### What are Configuration Attributes?

可以将“配置属性”视为对AbstractSecurityInterceptor使用的类具有特殊含义的字符串。它们由框架内的ConfigAttribute接口表示。它们可以是简单的角色名称，也可以具有更复杂的含义，具体取决于AccessDecisionManager实现的复杂程度。 AbstractSecurityInterceptor配置有SecurityMetadataSource，它用于查找安全对象的属性。通常，此配置对用户隐藏。配置属性将作为安全方法的注释或安全URL的访问属性输入。例如，当我们在名称空间介绍中看到类似<intercept-url pattern ='/ secure / **'access ='ROLE_A，ROLE_B'/>的内容时，这表示配置属性ROLE_A和ROLE_B适用于Web请求匹配给定的模式。在实践中，使用默认的AccessDecisionManager配置，这意味着任何具有与这两个属性之一匹配的GrantedAuthority的人都将被允许访问。但是严格来说，它们只是属性，其解释取决于AccessDecisionManager的实现。前缀ROLE_的使用是一个标记，用于指示这些属性是角色，并且应由Spring Security的RoleVoter使用。这仅在使用基于投票者的AccessDecisionManager时才相关。在授权一章中，我们将介绍如何实现AccessDecisionManager。

### RunAsManager

假设AccessDecisionManager决定允许该请求，则AbstractSecurityInterceptor通常将继续处理该请求。 话虽如此，在极少数情况下，用户可能希望用另一个身份验证来替换SecurityContext内部的身份验证，这由AccessDecisionManager调用RunAsManager来处理。 在相当不常见的情况下，例如服务层方法需要调用远程系统并提供不同的标识时，这可能很有用。 由于Spring Security会自动将安全身份从一台服务器传播到另一台服务器（假设您使用的是正确配置的RMI或HttpInvoker远程协议客户端），因此这可能很有用

### AfterInvocationManager

在安全对象调用继续进行之后，然后返回-这可能意味着方法调用完成或过滤器链继续进行-AbstractSecurityInterceptor获得了处理调用的最后机会。 在此阶段，AbstractSecurityInterceptor对可能修改返回对象感兴趣。 我们可能希望发生这种情况，因为无法在安全对象调用的“途中”做出授权决定。 由于高度可插拔，AbstractSecurityInterceptor会将控制权传递给AfterInvocationManager，以根据需要实际修改对象。 此类甚至可以完全替换对象，或者引发异常，也可以按照其选择的任何方式对其进行更改。 调用后检查仅在调用成功的情况下执行。 如果发生异常，将跳过其他检查。



![](D:\kkb\github\daily_notes\spring security\security.assets\重要类和接口.png)

### Extending the Secure Object Model

只有打算采用全新方法来拦截和授权请求的开发人员才需要直接使用安全对象。 例如，有可能建立一个新的安全对象以保护对消息系统的呼叫。 任何需要安全性并且还提供拦截呼叫的方式（例如，围绕建议语义的AOP）都可以成为安全对象。 话虽如此，大多数Spring应用程序将完全透明地使用当前支持的三种安全对象类型（AOP Alliance MethodInvocation，AspectJ JoinPoint和Web请求FilterInvocation）。



# Core Services

现在，我们已经对Spring Security体系结构及其核心类进行了高级概述，让我们仔细研究一下一个或两个核心接口及其实现，尤其是AuthenticationManager，UserDetailsService和AccessDecisionManager。 这些内容会在本文档的其余部分中定期进行整理，因此，了解它们的配置方式和操作方式非常重要。

## The AuthenticationManager, ProviderManager and AuthenticationProvider

AuthenticationManager只是一个接口，因此实现可以是我们选择的任何东西，但是它在实践中如何工作？ 如果我们需要检查多个身份验证数据库或不同身份验证服务（例如数据库和LDAP服务器）的组合，该怎么办？

Spring Security中的默认实现称为ProviderManager，它不处理身份验证请求本身，而是委派给已配置的AuthenticationProvider的列表，依次查询每个列表以查看其是否可以执行身份验证。 每个提供程序都将抛出异常或返回完全填充的Authentication对象。 还记得我们的好朋友UserDetails和UserDetailsService吗？ 如果没有，请返回上一章并刷新您的记忆。 验证身份验证请求的最常见方法是加载相应的UserDetails，并对照用户输入的密码检查加载的密码。 这是DaoAuthenticationProvider使用的方法（请参见下文）。 构建完整填充的Authentication对象时，将使用加载的UserDetails对象（尤其是其中包含的GrantedAuthority），该对象将从成功的身份验证返回并存储在SecurityContext中。

如果使用名称空间，则会在内部创建并维护ProviderManager的实例，然后使用名称空间身份验证提供程序元素将提供程序添加到该名称空间（请参见名称空间章节）。 在这种情况下，您不应在应用程序上下文中声明ProviderManager bean。 但是，如果您不使用名称空间，则可以这样声明：

```xml
<bean id="authenticationManager"
        class="org.springframework.security.authentication.ProviderManager">
    <constructor-arg>
        <list>
            <ref local="daoAuthenticationProvider"/>
            <ref local="anonymousAuthenticationProvider"/>
            <ref local="ldapAuthenticationProvider"/>
        </list>
    </constructor-arg>
</bean>

```

在上面的示例中，我们有三个提供程序。 按照显示的顺序尝试使用它们（通过使用List暗示），每个提供程序都可以尝试进行身份验证，也可以通过简单地返回null来跳过身份验证。 如果所有实现都返回null，则ProviderManager将引发ProviderNotFoundException。 如果您想了解有关链接提供程序的更多信息，请参阅ProviderManager Javadoc。

诸如Web表单登录处理过滤器之类的身份验证机制注入了对ProviderManager的引用，并将对其进行调用以处理其身份验证请求。您所需的提供程序有时可以与身份验证机制互换，而在其他时候，它们将取决于特定的身份验证机制。例如，DaoAuthenticationProvider和LdapAuthenticationProvider与提交简单的用户名/密码身份验证请求的任何机制兼容，因此可以与基于表单的登录名或HTTP Basic身份验证一起使用。另一方面，某些身份验证机制会创建一个身份验证请求对象，该对象只能由一种类型的AuthenticationProvider解释。一个示例就是JA-SIG CAS，它使用服务票证的概念，因此只能由CasAuthenticationProvider进行身份验证。您不必太担心这一点，因为如果您忘记注册合适的提供程序，则在尝试进行身份验证时只会收到ProviderNotFoundException。

### Erasing Credentials on Successful Authentication

默认情况下（从Spring Security 3.1开始），ProviderManager将尝试从Authentication对象中清除所有敏感的凭据信息，该信息由成功的身份验证请求返回。 这样可以防止将密码之类的信息保留的时间过长。

例如，在使用用户对象的缓存来提高无状态应用程序的性能时，这可能会导致问题。 如果身份验证包含对缓存中对象的引用（例如UserDetails实例），并且已删除其凭据，则将无法再对缓存的值进行身份验证。 如果使用缓存，则需要考虑到这一点。 一个明显的解决方案是首先在缓存实现中或在创建返回的Authentication对象的AuthenticationProvider中创建对象的副本。 或者，您可以在ProviderManager上禁用deleteCredentialsAfterAuthentication属性。 有关更多信息，请参见Javadoc。

### DaoAuthenticationProvider

Spring Security实现的最简单的AuthenticationProvider是DaoAuthenticationProvider，它也是框架最早支持的之一。 它利用UserDetailsService（作为DAO）来查找用户名，密码和GrantedAuthority。 只需将UsernamePasswordAuthenticationToken中提交的密码与UserDetailsService加载的密码进行比较，即可对用户进行身份验证。 配置提供程序非常简单：

```xml
<bean id="daoAuthenticationProvider"
    class="org.springframework.security.authentication.dao.DaoAuthenticationProvider">
<property name="userDetailsService" ref="inMemoryDaoImpl"/>
<property name="passwordEncoder" ref="passwordEncoder"/>
</bean>

```

PasswordEncoder是可选的。 PasswordEncoder提供对从配置的UserDetailsService返回的UserDetails对象中提供的密码的编码和解码。 这将在下面更详细地讨论。

### UserDetailsService Implementations

如本参考指南前面所述，大多数身份验证提供程序都利用UserDetails和UserDetailsService接口。 回想一下，UserDetailsService的是一个单一方法：

```java
UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
```

返回的UserDetails是提供获取器的接口，该获取器保证以非空方式提供身份验证信息，例如用户名，密码，授予的权限以及启用或禁用用户帐户。 即使用户名和密码实际上没有用作身份验证决策的一部分，大多数身份验证提供程序也会使用UserDetailsService。 他们可能仅将返回的UserDetails对象用于其GrantedAuthority信息，因为某些其他系统（例如LDAP或X.509或CAS等）承担了实际验证凭据的责任。

鉴于UserDetailsService的实施是如此简单，因此用户应该使用自己选择的持久性策略轻松检索身份验证信息。 话虽如此，Spring Security确实包含一些有用的基本实现，我们将在下面进行介绍。

### In-Memory Authentication

创建自定义UserDetailsService实现很容易使用，该实现从所选的持久性引擎中提取信息，但是许多应用程序不需要这种复杂性。 如果您确实不想花费时间配置数据库或编写UserDetailsService实现，则在构建原型应用程序或开始集成Spring Security时尤其如此。 对于这种情况，一个简单的选择是使用安全名称空间中的user-service元素：

```xml
<user-service id="userDetailsService">
<!-- Password is prefixed with {noop} to indicate to DelegatingPasswordEncoder that
NoOpPasswordEncoder should be used. This is not safe for production, but makes reading
in samples easier. Normally passwords should be hashed using BCrypt -->
<user name="jimi" password="{noop}jimispassword" authorities="ROLE_USER, ROLE_ADMIN" />
<user name="bob" password="{noop}bobspassword" authorities="ROLE_USER" />
</user-service>

```

This also supports the use of an external properties file:

```xml
<user-service id="userDetailsService" properties="users.properties"/>
```

The properties file should contain entries in the form

```properties
username=password,grantedAuthority[,grantedAuthority][,enabled|disabled]
```

For example

```properties
jimi=jimispassword,ROLE_USER,ROLE_ADMIN,enabled
bob=bobspassword,ROLE_USER,enabled
```

### JdbcDaoImpl

Spring Security还包括一个UserDetailsService，可以从JDBC数据源获取身份验证信息。 在内部使用Spring JDBC，因此它避免了仅用于存储用户详细信息的功能齐全的对象关系映射器（ORM）的复杂性。 如果您的应用程序确实使用了ORM工具，则您可能更愿意编写自定义UserDetailsService来重用您可能已经创建的映射文件。 返回到JdbcDaoImpl，下面显示一个示例配置：

```xml
<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
<property name="driverClassName" value="org.hsqldb.jdbcDriver"/>
<property name="url" value="jdbc:hsqldb:hsql://localhost:9001"/>
<property name="username" value="sa"/>
<property name="password" value=""/>
</bean>
<bean id="userDetailsService"
    class="org.springframework.security.core.userdetails.jdbc.JdbcDaoImpl">
<property name="dataSource" ref="dataSource"/>
</bean>
```

您可以通过修改上面显示的DriverManagerDataSource来使用不同的关系数据库管理系统。 您还可以使用从JNDI获得的全局数据源，就像其他任何Spring配置一样。

### Authority Groups

默认情况下，JdbcDaoImpl假定权限直接映射到用户（请参阅数据库架构附录），从而为单个用户加载权限。 另一种方法是将权限划分为多个组，然后将组分配给用户。 有些人更喜欢使用这种方法来管理用户权限。 有关如何启用组权限的更多信息，请参见JdbcDaoImpl Javadoc。 组模式也包含在附录中。



# Authentication

## In-Memory Authentication

我们已经看到了为单个用户配置内存中身份验证的示例。 下面是配置多个用户的示例：

```java
//基于内存的多用户支持
    //spring security还支持各种来源的用户数据，包括内存、数据库、LDAP等。他们都被抽象为一个
    //UserDetailsService接口，任何实现了UserDetailsService接口的对象都可以作为认证数据源。
    //在这种设计模式下，spring Security显得尤为灵活
    // InMemoryUserDetailsManager是UserDetailsService接口的一个实现类，它将用户数据寄存到内存里
    //在一些不需要引入数据库这种数据源的系统中很有帮助
    @Bean
    public UserDetailsService userDetailsService(){
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
        //可以从配置文件或数据库中读取，这里设计可以很灵活，
        //@Autowired manger 然后manger.createUser就行了，作为架构师，应该有这方面的考虑
        //但spring security同样只是数据库的实现类，作为高级架构师，可以从性能和安全性方面来考虑
        //这里可以很妙，比如UKEY ZK 等各种数据，都可以寄存到内存中
        //这里可能还涉及了监听，定时任务等因素
        manager.createUser(User.withUsername("user").password(
                PasswordEncoderFactories.createDelegatingPasswordEncoder().encode("123")
        ).roles("USER").build());
        manager.createUser(User.withUsername("admin").password(PasswordEncoderFactories.createDelegatingPasswordEncoder().encode("123")).roles("USER","ADMIN").build());
        return manager;
    }
```

### JDBC Authentication

您可以找到更新以支持基于JDBC的身份验证。 下面的示例假定您已经在应用程序中定义了一个数据源。 jdbc-javaconfig示例提供了使用基于JDBC身份验证的完整示例。

```java
@Bean
    public UserDetailsService userDetailsService(){
        JdbcUserDetailsManager manager = new JdbcUserDetailsManager();
        manager.setDataSource(dataSource);
        // 关于There is no PasswordEncoder mapped for the id "null"
        //现如今Spring Security中密码的存储格式是“{id}…………”。
        // 前面的id是加密方式，id可以是bcrypt、sha256等，后面跟着的是加密后的密码。
        // 也就是说，程序拿到传过来的密码的时候，会首先查找被“{”和“}”包括起来的id，
        // 来确定后面的密码是被怎么样加密的，如果找不到就认为id是null。
        // 这也就是为什么我们的程序会报错：There is no PasswordEncoder mapped for the id “null”。
        // 官方文档举的例子中是各种加密方式针对同一密码加密后的存储形式，原始密码都是“password”。

        if(!manager.userExists("user"))
            manager.createUser(User.withUsername("user").password(PasswordEncoderFactories.createDelegatingPasswordEncoder().encode("123")).roles("USER").build());
        if(!manager.userExists("admin"))
            manager.createUser(User.withUsername("admin").password(PasswordEncoderFactories.createDelegatingPasswordEncoder().encode("123")).roles("USER","ADMIN").build());
        return manager;
    }
```

### LDAP Authentication

LDAP通常被组织用作用户信息的中央存储库和身份验证服务。 它还可以用于存储应用程序用户的角色信息。

关于如何配置LDAP服务器，有许多不同的方案，以便Spring Security的LDAP提供程序是完全可配置的。 它使用单独的策略接口进行身份验证和角色检索，并提供可配置为处理各种情况的默认实现。