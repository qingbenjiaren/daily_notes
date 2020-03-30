# 初识Spring Security

## Spring Security简介

Spring Security的前身是Acegi Security，在被收纳为Spring子项目后正式更名为Spring Security。

在Java的应用安全领域，Spring Security会成为被首选推崇的解决方案，就像我们看到服务器就会联想到Linux一样顺利成章。

应用程序的安全性通常体现在两个方面：**认证和授权**

**认证**是确认某主体在某系统中是否合法、可用的过程。这里的主体既可以是登录系统的用户，也可以是接入的设备或者其他系统。

**授权**是指当主体通过认证之后，是否允许其执行某项操作的过程。

学习Spring Security并非局限于降低Java应用的安全开发成本，通过Spring Security了解常见的安全攻击手段以及对应的防护方法也尤为重要，这些是脱离具体开发语言而存在的。

## 创建一个简单的Spring Security项目

创建一个spring boot项目，在选择项目依赖的时候选中Security、web即可

写一个简单的Controller就可以进行测试了。

在引入Security依赖之后，当我们访问controller的时候，浏览器将弹出一个需要进行身份验证的对话框，要求在经过HTTP基本认证之后才能访问对应的URL资源。

事实上，绝大部分Web应用都不会选择HTTP基本认证这种方式，除安全性差、无法携带cookie等因素外，灵活性不够也是它的一个主要缺点。通常大家更愿意选择表单认证，自己实现表单登录页和验证逻辑，从而提高安全性。

## 表单认证

### 默认表单认证

新建一个WebSecurityConfig类，使其继承WebSecurityConfigurationAdapter

在给WebSecurity类中加上@EnableWebSecurity注解后，便会自动被spring发现并注册。

WebSecurityConfigurationAdapter已经默认声明了一些安全特性

- 验证所有请求
- 允许用户使用表单登录进行身份验证（Spring Security提供了一个简单的表单登录页面）
- 允许用户使用Http基本认证

### 自定义表单登录页

```java

@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .formLogin()
                .loginPage("/myLogin.html")
                //指定处理登录请求的路径
                .loginProcessingUrl("/login")
                //指定登录成功时的处理逻辑
                .successHandler((x,y,z) ->{
                    y.setContentType("application/json;charset=UTF-8");
                    y.getWriter().write("{\"error_code\":\"0\",\"message\":\"欢迎登录系统\"}");
                })
                .failureHandler((x,y,z) ->{
                    y.setContentType("application/json;charset=UTF-8");
                    y.setStatus(401);
                    y.getWriter().write("{\"error_code\":\"401\",\"name\"" + z.getClass() +"\"message\":"+ z.getMessage());
                })
            	//使登录页不设限访问
                .permitAll()
                .and()
                .csrf().disable();
    }
}

```

```html
<!DOCTYPE HTML>
<html>
    <head>
        <title>登录</title>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
        <style></style>
    </head>
    <body>
        <div>
            <h2>Access Form</h2>
            <div>
                <h1>LOGIN FORM</h1>
                <form action="login" method="post">
                    username: <input type="text" name="username" placeholder="username" />
                    password: <input type="text" name="password" placeholder="password" />
                    <input type="submit" value="Login" >
                </form>
            </div>
        </div>
    </body>
</html>
```

#### 认识HttpSecurity

HttpSecurity提供了很多配置相关的方法。例如，authorizeRequest()、formLogin()、httpBasic()和csrf()。调用这些方法之后，使用and()方法结束当前标签，上下文会回到HttpSecurity，否则链式调用的上下文将自动进入对应标签域。

authorizeRequests()方法实际上返回了一个URL拦截注册器，我们可以调用它提供的anyRequest()、anyMatchers()和regexMatchers()等方法来匹配系统的URL，并为其指定安全策略。

formLogin()方法和httpBasic()方法都声明了需要Spring Security提供的表单认证方式，分别返回对应的配置器。其中formLogin().loginPage("")指定自定义的登录页，同时，Spring Security会用""注册一个POST路由，用于接收登录请求。

csrf()方法是Spring Security提供的跨站请求伪造防护功能，当我们继承WebSecurityConfigurationAdapter的时候会默认开启csrf()方法。

## 认证和授权

对应

02-security-memory

03-security-jdbc

04-security-jdbc2

自定义认证流程

![](C:\Users\24926\Desktop\验证过程.png)