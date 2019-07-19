## spring-security 5.1.5.RELEASE
## 13.7 CAS Authentication
### 13.7.1 概述
JA-SIG创建了一个企业级的SSO系统叫CAS，不同于其他方案，JA-SIG的主要的认证服务是开源的，广泛使用，简单的理解，不依赖平台并且支持代理功能。Spring Security完全的支持CAS，并且提供了从springsecurity单体应用部署简单的迁移到多应用部署企业级CAS服务器。
你可以了解更多的关于CAS [https://www.apereo.org](https://www.apereo.org).你也需要访问该网站下载CAS服务文件。
### 13.7.2 CAS怎样工作？
虽然CAS网站包含了详细的CAS体系结构，我们这里提供带有Spring Security上下文的常规概述。Spring Security3.x,支持CAS3。在写本文同时，CAS服务器版本是3.4。
在您的企业您需要建立CAS服务器。CAS服务器是一个简单的标准WAR文件，因此建立服务器是不难的。在WAR文件里面您可以定制自己的登录页和其他的单点登录页，展示给用户。
我们发布一个CAS3.4服务器，您需要在CAS里面的`deployerConfigContext.xml`里面指定一个`AuthenticationHandler`，`AuthenticationHandler`有一个简单的方法，这方法返回一个boolean值，用来标识是否设置有效的Credentials。`AuthenticationHandler`的实现需要关联某个类型的后端认证库，例如LDAP服务，或者数据库。CAS包括了许多`AuthenticationHandler`,开箱即用。当您下载并部署了服务WAR，被建立起来用来成功认证了那些输入用户名与密码匹配的用户，这是用于测试。
除了CAS服务器本身，其他关键参与者，当然是确保部署在您企业中的应用安全运行。这些参与者被认为是“services”。有三类“services”。是认证服务tickets，获得代理tickets，认证代理tickets。认证一个代理ticket是不同的，由于代理列表必须经过验证并且一个ticket经常的被重复利用。

### Spring Security and CAS 交互序列
在web浏览器、CAS服务器和Spring Security安全服务之间基本的交互，如下：


- web用户浏览服务公共页面，CAS和Spring Security不会参与。

- 用户最终请求的页面要么是安全的，要么是用于安全的bean中的一个（例loginController）,Spring Security’s `ExceptionTranslationFilter`会监测到`AccessDeniedException` 或者 `AuthenticationException`。

-由于用户的认证对象引起了 `AuthenticationException`,`ExceptionTranslationFilter`将调用配置的`AuthenticationEntryPoint`。如果使用CAS，它将是`CasAuthenticationEntryPoint`类。

- `CasAuthenticationEntryPoint`将用户的浏览器重定向到CAS服务器，传递一个服务参数，是一个Spring Security服务回调url（您应用地址）。例如，URL重定向地址可能是https://my.company.com/cas/login?service=https%3A%2F%2Fserver3.company.com%2Fwebapp%2Flogin/cas.
   
- 用户浏览器重定向到CAS之后，他们会被提示输入用户名密码。如果用户提供了携带着他们以前登录状态的session cookie，将不会再次登录（①我们之后将覆盖这个异常步骤，填坑。。。）。CAS将用`PasswordHandler`(如果是CAS3.0是`AuthenticationHandler`)决定是否进行用户密码校验。

- 一经登录成功，CAS将重定向用户的浏览器回到原来的服务。它将包括ticket参数，是一个不透明的字符串代表 "service ticket"。继续示例，浏览器URL被重定向可能地址 https://server3.company.com/webapp/login/cas?ticket=ST-0-ER94xMJmn6pha35CQRoZ.

- 回到服务的web应用，`CasAuthenticationFilter`总是监听到`/login/cas`的请求（这是可配置的，在此我们使用默认配置）。处理过滤器将构造一个`UsernamePasswordAuthenticationToken`代表这个服务ticket。principal将和`CasAuthenticationFilter.CAS_STATEFUL_IDENTIFIER`进行比对,当credentials是服务ticket的不透明值时。认证请求将被配置的`AuthenticationManager`处理。

- `AuthenticationManager`实现是`ProviderManager`,它使用`CasAuthenticationProvider`轮转到配置的`CasAuthenticationProvider`。`CasAuthenticationProvider`仅响应包含了CAS特性principal的`UsernamePasswordAuthenticationToken`（例如`CasAuthenticationFilter.CAS_STATEFUL_IDENTIFIER`）和`CasAuthenticationToken`(②之后讨论)。

- `CasAuthenticationProvider`将使用一个`TicketValidator`的实现来验证服务ticket。这个实现通常是一个`Cas20ServiceTicketValidator`，它是包含在CAS clinet类库里面的。如果应用要验证代理tickets，使用`Cas20ProxyTicketValidator`实现类。`TicketValidator`使用https请求CAS服务器验证服务ticket，这也可能包括一个代理回调URL，例如：https://my.company.com/cas/proxyValidate?service=https%3A%2F%2Fserver3.company.com%2Fwebapp%2Flogin/cas&ticket=ST-0-ER94xMJmn6pha35CQRoZ&pgtUrl=https://server3.company.com/webapp/login/cas/proxyreceptor.

- 验证请求被CAS服务器接收到。如果提交的服务ticket与服务URL的ticket匹配，CAS将在响应的XML里面包含username。如果代理参与了认证（在下面讨论），代理列表也包含在XML里面。

- 【可选择配置】如果请求到CAS验证服务时包含了代理回调URL（`pgtUrl`参数里面），CAS包含`pgtIou`字符串在xml响应中。`pgtIou`表示一个授权代理的ticket IOU。CAS服务器之后会创建自己的https连接回到`pgtUrl`。这是CAS服务器和声明的URL服务互相认证。https连接会用来发送一个代理授权ticket到原始的web应用。例如, https://server3.company.com/webapp/login/cas/proxyreceptor?pgtIou=PGTIOU-0-R0zlgrl4pdAQwBvJWO3vnNpevwqStbSGcq3vKB2SqSFFRnjPHt&pgtId=PGT-1-si9YkkHLrtACBo64rmsi3v2nf7cpCResXg5MpESZFArbaZiOKH.

- `Cas20TicketValidator`将解析从CAS服务器接收到的XML，然后返回给`CasAuthenticationProvider`一个包含username的`TicketResponse`，如果使用了代理，会返回代理列表和代理授权ticket IOU。

- 下一步，`CasAuthenticationProvider`将调用一个被配置的`CasProxyDecider`。`CasProxyDecider`决定了包含在`TicketResponse`中的代理列表是否是可接受的。Spring Security提供了几个实现：`RejectProxyTickets`, `AcceptAnyCasProxy` 和 `NamedCasProxyDecider`。那些列举显而易见，除了`NamedCasProxyDecider`都允许提供一个信任的代理列表。

- `CasAuthenticationProvider`请求一个`AuthenticationUserDetailsService`来加载用户包含在`Assertion`里面的GrantedAuthority对象。

- 如果没有问题，`CasAuthenticationProvider`构造一个包括`TicketResponse`中所有细节和授权的`CasAuthenticationToken`

- 返回到`CasAuthenticationFilter`，它会把创建的`CasAuthenticationToken`放置在安全上下文中。

- 用户浏览器被重定向到引发`AuthenticationException`的原始页(or依赖于自定义配置的地址).

### 13.7.3 Configuration of CAS Client