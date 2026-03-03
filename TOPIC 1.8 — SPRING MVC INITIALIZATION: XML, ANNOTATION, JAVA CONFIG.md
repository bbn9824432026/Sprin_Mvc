# TOPIC 1.8 — SPRING MVC INITIALIZATION: XML, ANNOTATION, JAVA CONFIG

---

## 1️⃣ CONCEPTUAL EXPLANATION

### Why Three Initialization Styles Exist

Spring MVC has evolved through three distinct configuration eras. Each style is still valid and appears in production codebases today. Certification questions mix all three — often asking you to identify what a config does, spot bugs across styles, or predict behavior differences. You must be fluent in all three.

```
Era 1 (2003-2007): XML-only configuration
  → web.xml + applicationContext.xml + dispatcher-servlet.xml
  → Verbose, explicit, no compile-time safety

Era 2 (2007-2012): Annotation-driven + XML hybrid  
  → web.xml still required (Servlet 2.x)
  → Spring annotations (@Controller, @RequestMapping) + XML namespace
  → <mvc:annotation-driven/> introduced

Era 3 (2012-present): Pure Java configuration
  → Servlet 3.0+ allows zero web.xml
  → WebApplicationInitializer / AbstractAnnotationConfigDispatcherServletInitializer
  → @Configuration + @EnableWebMvc + WebMvcConfigurer
  → Full compile-time safety, IDE support, refactoring-safe
```

---

### Style 1 — Pure XML Configuration

The original style. Every bean, every mapping, every configuration is in XML files.

**Components involved:**
- `web.xml` — Servlet deployment descriptor
- `applicationContext.xml` — Root context (or named via `contextConfigLocation`)
- `dispatcher-servlet.xml` — Child context (named `{servlet-name}-servlet.xml`)

**Key XML namespaces:**
```xml
xmlns:mvc="http://www.springframework.org/schema/mvc"
xmlns:context="http://www.springframework.org/schema/context"
xmlns:tx="http://www.springframework.org/schema/tx"
xmlns:aop="http://www.springframework.org/schema/aop"
```

**`<mvc:annotation-driven/>`** is the critical element. It registers:
- `RequestMappingHandlerMapping` (replaces `DefaultAnnotationHandlerMapping`)
- `RequestMappingHandlerAdapter` (replaces `AnnotationMethodHandlerAdapter`)
- `ExceptionHandlerExceptionResolver`
- `ResponseStatusExceptionResolver`
- `DefaultHandlerExceptionResolver`
- All default `HttpMessageConverter` instances (Jackson, JAXB, String, etc.)
- `ConversionService`
- `Validator` (JSR-303 if on classpath)

Without `<mvc:annotation-driven/>`, `@RequestMapping`, `@ResponseBody`, `@Valid` do NOT work properly.

---

### What `<mvc:annotation-driven/>` Registers — Precise Bean List

```
RequestMappingHandlerMapping (order=0)
  └── with ContentNegotiationManager
  └── with ConversionService
  └── with Validator

RequestMappingHandlerAdapter
  └── with MessageConverters:
        StringHttpMessageConverter
        ByteArrayHttpMessageConverter
        ResourceHttpMessageConverter
        SourceHttpMessageConverter
        FormHttpMessageConverter
        MappingJackson2HttpMessageConverter (if Jackson on classpath)
        Jaxb2RootElementHttpMessageConverter (if JAXB on classpath)
  └── with ConversionService (DefaultFormattingConversionService)
  └── with Validator (LocalValidatorFactoryBean if JSR-303 on classpath)

ExceptionHandlerExceptionResolver (order=1)
ResponseStatusExceptionResolver (order=2)
DefaultHandlerExceptionResolver (order=3)

HandlerMappingIntrospector
BeanNameUrlHandlerMapping (order=2, always registered)
```

---

### Style 2 — Annotation-Driven with XML (Hybrid)

Most common in codebases written 2008-2015. `web.xml` still present but Spring annotations do the heavy lifting.

`@Controller`, `@Service`, `@Repository` are scanned via `<context:component-scan/>`. `@RequestMapping` methods are discovered via `<mvc:annotation-driven/>`.

This style completely replaced manual `SimpleUrlHandlerMapping` and `BeanNameUrlHandlerMapping` XML configuration for controllers.

---

### Style 3 — Pure Java Configuration

Introduced with Servlet 3.0 (2009) and Spring 3.1 (2011). Zero XML files required.

**The Entry Point — `WebApplicationInitializer`**

```java
public interface WebApplicationInitializer {
    void onStartup(ServletContext servletContext) throws ServletException;
}
```

The Servlet 3.0 `ServletContainerInitializer` mechanism discovers Spring's implementation:

```
META-INF/services/jakarta.servlet.ServletContainerInitializer
  contains: org.springframework.web.SpringServletContainerInitializer
```

`SpringServletContainerInitializer` scans the classpath for `WebApplicationInitializer` implementations and calls `onStartup()` on each.

**The Hierarchy of Initializer Classes:**

```
WebApplicationInitializer (interface)
    └── AbstractContextLoaderInitializer
            └── AbstractDispatcherServletInitializer
                    └── AbstractAnnotationConfigDispatcherServletInitializer
                            ← Most commonly used
```

---

### `AbstractAnnotationConfigDispatcherServletInitializer` — Internal Analysis

This class does ALL the web.xml work programmatically:

```java
public abstract class AbstractAnnotationConfigDispatcherServletInitializer
    extends AbstractDispatcherServletInitializer {

    // You implement these:
    protected abstract Class<?>[] getRootConfigClasses();
    protected abstract Class<?>[] getServletConfigClasses();
    protected abstract String[] getServletMappings();

    // Creates Root ApplicationContext
    @Override
    protected WebApplicationContext createRootApplicationContext() {
        Class<?>[] configClasses = getRootConfigClasses();
        if (configClasses != null && configClasses.length > 0) {
            AnnotationConfigWebApplicationContext context =
                new AnnotationConfigWebApplicationContext();
            context.register(configClasses);
            return context;
        }
        return null; // No root context if null returned
    }

    // Creates Child WebApplicationContext
    @Override
    protected WebApplicationContext createServletApplicationContext() {
        AnnotationConfigWebApplicationContext context =
            new AnnotationConfigWebApplicationContext();
        Class<?>[] configClasses = getServletConfigClasses();
        if (configClasses != null) {
            context.register(configClasses);
        }
        return context;
    }
}
```

```java
// AbstractDispatcherServletInitializer — what it does:
public abstract class AbstractDispatcherServletInitializer
    extends AbstractContextLoaderInitializer {

    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        super.onStartup(servletContext); // Registers ContextLoaderListener

        // Creates DispatcherServlet
        WebApplicationContext servletAppContext = 
            createServletApplicationContext();
        
        DispatcherServlet dispatcherServlet = 
            createDispatcherServlet(servletAppContext);

        // Registers with Servlet container
        ServletRegistration.Dynamic registration = 
            servletContext.addServlet(getServletName(), dispatcherServlet);
        
        registration.setLoadOnStartup(1); // Always load-on-startup=1
        registration.addMapping(getServletMappings());
        registration.setAsyncSupported(isAsyncSupported());

        // Registers filters
        Filter[] filters = getServletFilters();
        if (filters != null) {
            for (Filter filter : filters) {
                registerServletFilter(servletContext, filter);
            }
        }
    }
}
```

---

### `@EnableWebMvc` — What It Does Precisely

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class) // ← The key
public @interface EnableWebMvc { }
```

`@EnableWebMvc` imports `DelegatingWebMvcConfiguration`:

```
DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport
```

`WebMvcConfigurationSupport` is the Java equivalent of `<mvc:annotation-driven/>`. It defines `@Bean` methods for ALL the same infrastructure beans:

```java
// Inside WebMvcConfigurationSupport (simplified):
@Bean
public RequestMappingHandlerMapping requestMappingHandlerMapping() { ... }

@Bean
public RequestMappingHandlerAdapter requestMappingHandlerAdapter() { ... }

@Bean
public HandlerExceptionResolverComposite handlerExceptionResolver() { ... }

@Bean
public ContentNegotiationManager contentNegotiationManager() { ... }

@Bean
public FormattingConversionService mvcConversionService() { ... }

@Bean
public Validator mvcValidator() { ... }
```

`DelegatingWebMvcConfiguration` extends this and **delegates** customization to `WebMvcConfigurer` implementations:

```java
public class DelegatingWebMvcConfiguration 
    extends WebMvcConfigurationSupport {

    // Collects ALL WebMvcConfigurer beans from context
    @Autowired(required = false)
    public void setConfigurers(List<WebMvcConfigurer> configurers) {
        if (!CollectionUtils.isEmpty(configurers)) {
            this.configurers.addWebMvcConfigurers(configurers);
        }
    }

    // Delegates to all WebMvcConfigurer implementations
    @Override
    protected void addInterceptors(InterceptorRegistry registry) {
        this.configurers.addInterceptors(registry);
    }
    // Same for addCorsMappings, configureViewResolvers, etc.
}
```

This is why `WebMvcConfigurer` works — `DelegatingWebMvcConfiguration` calls all its methods automatically.

---

### `WebMvcConfigurer` — Complete API

Every method corresponds to a specific Spring MVC feature:

```java
public interface WebMvcConfigurer {

    // PATH MATCHING
    void configurePathMatch(PathMatchConfigurer configurer);
    
    // CONTENT NEGOTIATION
    void configureContentNegotiation(ContentNegotiationConfigurer configurer);
    
    // ASYNC SUPPORT
    void configureAsyncSupport(AsyncSupportConfigurer configurer);
    
    // DEFAULT SERVLET
    void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer);
    
    // CONVERTERS AND FORMATTERS
    void addFormatters(FormatterRegistry registry);
    
    // INTERCEPTORS
    void addInterceptors(InterceptorRegistry registry);
    
    // STATIC RESOURCES
    void addResourceHandlers(ResourceHandlerRegistry registry);
    
    // CORS
    void addCorsMappings(CorsRegistry registry);
    
    // VIEW CONTROLLERS (simple redirect/forward with no controller class)
    void addViewControllers(ViewControllerRegistry registry);
    
    // VIEW RESOLVERS
    void configureViewResolvers(ViewResolverRegistry registry);
    
    // ARGUMENT RESOLVERS
    void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers);
    
    // RETURN VALUE HANDLERS
    void addReturnValueHandlers(List<HandlerMethodReturnValueHandler> handlers);
    
    // MESSAGE CONVERTERS
    void configureMessageConverters(List<HttpMessageConverter<?>> converters);
    void extendMessageConverters(List<HttpMessageConverter<?>> converters);
    
    // EXCEPTION HANDLERS
    void configureHandlerExceptionResolvers(
        List<HandlerExceptionResolver> resolvers);
    void extendHandlerExceptionResolvers(
        List<HandlerExceptionResolver> resolvers);
    
    // VALIDATOR
    Validator getValidator();
    
    // MESSAGE CODES RESOLVER
    MessageCodesResolver getMessageCodesResolver();
}
```

**Critical distinction:**
- `configureMessageConverters()` — **replaces** all default converters with your list
- `extendMessageConverters()` — **adds to** or modifies the default list

This difference is a certification trap. Using `configureMessageConverters()` removes Jackson — JSON stops working.

---

### XML Namespace vs Java Config Equivalents

| XML | Java Config Equivalent |
|---|---|
| `<mvc:annotation-driven/>` | `@EnableWebMvc` |
| `<context:component-scan/>` | `@ComponentScan` |
| `<mvc:interceptors/>` | `addInterceptors()` in `WebMvcConfigurer` |
| `<mvc:resources/>` | `addResourceHandlers()` in `WebMvcConfigurer` |
| `<mvc:default-servlet-handler/>` | `configureDefaultServletHandling()` |
| `<mvc:cors/>` | `addCorsMappings()` |
| `<mvc:view-controller/>` | `addViewControllers()` |
| `<bean class="InternalResourceViewResolver"/>` | `configureViewResolvers()` or `@Bean` |
| `<tx:annotation-driven/>` | `@EnableTransactionManagement` |
| `<mvc:content-negotiation/>` | `configureContentNegotiation()` |

---

### Three-Way Comparison — Same Feature, Three Styles

**Adding an interceptor — all three styles:**

```xml
<!-- XML Style -->
<mvc:interceptors>
    <mvc:interceptor>
        <mvc:mapping path="/api/**"/>
        <mvc:exclude-mapping path="/api/public/**"/>
        <bean class="com.example.LoggingInterceptor"/>
    </mvc:interceptor>
</mvc:interceptors>
```

```java
// Annotation-driven (XML declares namespace, Java class is interceptor)
// web.xml still exists, dispatcher-servlet.xml has <mvc:annotation-driven/>
// But interceptor itself is a Java class registered in XML
```

```java
// Pure Java Config
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoggingInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/public/**");
    }
}
```

---

### `configureMessageConverters` vs `extendMessageConverters` — Critical Trap

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // WRONG — replaces ALL default converters
    // After this: no String converter, no ByteArray converter,
    // no Jackson converter — ONLY your custom one
    @Override
    public void configureMessageConverters(
            List<HttpMessageConverter<?>> converters) {
        converters.add(new MappingJackson2HttpMessageConverter());
        // ALL defaults gone — this is the ONLY converter now
    }

    // CORRECT — adds to existing defaults
    @Override
    public void extendMessageConverters(
            List<HttpMessageConverter<?>> converters) {
        // Defaults still present
        // Add your custom converter at specific position
        converters.add(0, new MyCustomConverter()); // First priority
    }
}
```

---

### Servlet 3.0 Discovery Mechanism — How Zero-XML Works

```
JVM starts Tomcat
        │
        ▼
Tomcat's ClassLoader scans all JARs for:
  META-INF/services/jakarta.servlet.ServletContainerInitializer
        │
        ▼
Finds spring-web.jar's entry:
  org.springframework.web.SpringServletContainerInitializer
        │
        ▼
SpringServletContainerInitializer.onStartup() called by Tomcat
        │
        ▼
Scans classpath for WebApplicationInitializer implementations
(uses @HandlesTypes(WebApplicationInitializer.class))
        │
        ▼
Finds your MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer
        │
        ▼
Calls myWebAppInitializer.onStartup(servletContext)
        │
        ▼
ContextLoaderListener registered programmatically
DispatcherServlet registered programmatically
Filters registered programmatically
        │
        ▼
Normal startup continues
```

This is why NO `web.xml` is needed in Servlet 3.0+. The container finds Spring via `ServiceLoader`.

---

### `AnnotationConfigWebApplicationContext` vs `XmlWebApplicationContext`

| Feature | `AnnotationConfigWebApplicationContext` | `XmlWebApplicationContext` |
|---|---|---|
| Config input | `@Configuration` classes | XML files |
| Registration | `context.register(MyConfig.class)` | `context.setConfigLocation("classpath:...")` |
| Default location | None — must specify | `/WEB-INF/{name}-servlet.xml` |
| Bean discovery | `@ComponentScan` | `<context:component-scan/>` |
| Namespace | `@EnableWebMvc` | `<mvc:annotation-driven/>` |
| Profile activation | `@ActiveProfiles` / Environment | `<beans profile="...">` |

---

### Spring Boot Initialization — The Full Picture

Spring Boot's initialization completely bypasses all of the above:

```java
@SpringBootApplication
// = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

Internally `SpringApplication.run()`:

```
1. Creates SpringApplication instance
2. Determines WebApplicationType (SERVLET / REACTIVE / NONE)
3. Loads SpringFactories — finds AutoConfiguration classes
4. Creates AnnotationConfigServletWebServerApplicationContext
5. Prepares environment (reads application.properties, env vars)
6. Runs ApplicationContextInitializers
7. Calls context.refresh()
   a. Processes @SpringBootApplication (@ComponentScan → finds beans)
   b. Runs AutoConfigurations:
      - DispatcherServletAutoConfiguration → registers DispatcherServlet bean
      - WebMvcAutoConfiguration → configures MVC infrastructure
      - EmbeddedWebServerFactoryCustomizerAutoConfiguration → configures Tomcat
   c. Instantiates all singletons
   d. EmbeddedWebServerFactory creates and starts Tomcat
   e. DispatcherServlet registered with Tomcat via ServletRegistrationBean
8. Publishes ApplicationReadyEvent
9. Returns ConfigurableApplicationContext
```

**Key difference from plain MVC:** `DispatcherServlet` is a Spring BEAN registered via `DispatcherServletAutoConfiguration.DispatcherServletRegistrationConfiguration`. It is NOT created by Tomcat calling `init()` — Spring manages its lifecycle.

---

### Profile-Based Configuration — Across All Three Styles

```xml
<!-- XML style profiles -->
<beans profile="development">
    <bean id="dataSource" class="EmbeddedDatabaseBuilder..."/>
</beans>
<beans profile="production">
    <bean id="dataSource" class="JndiDataSourceLookup..."/>
</beans>
```

```java
// Java Config profiles
@Configuration
@Profile("development")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder().build();
    }
}

@Configuration
@Profile("production")
public class ProdConfig {
    @Bean
    public DataSource dataSource() throws NamingException {
        return (DataSource) new JndiTemplate().lookup("java:comp/env/jdbc/myDS");
    }
}
```

---

### Common Initialization Errors and Their Root Causes

```
Error 1: "No mapping found for HTTP request with URI [/] in DispatcherServlet"
Cause: <mvc:annotation-driven/> missing or @EnableWebMvc missing
       → RequestMappingHandlerMapping not registered
       → @RequestMapping methods not discovered

Error 2: HttpMediaTypeNotAcceptableException (406)
Cause: Jackson not on classpath but @ResponseBody used
       → No JSON MessageConverter registered
       → Cannot produce application/json

Error 3: "No qualifying bean of type 'ProductService'"
Cause: Component scan in root context excluded service package
       OR service is @Controller scoped only (wrong stereotype)

Error 4: "Cannot serialize" / HttpMessageNotWritableException
Cause: configureMessageConverters() used instead of extendMessageConverters()
       → Replaced all converters including Jackson

Error 5: "No WebApplicationContext found"
Cause: ContextLoaderListener not registered
       WebApplicationContextUtils.getRequiredWebApplicationContext() called

Error 6: BeanCreationException during child context refresh
Cause: @Controller tries to @Autowire bean that's in child context only
       AND that bean isn't in root context either
```

---

## 2️⃣ CODE EXAMPLES

### Style 1 — Complete Pure XML Setup

```xml
<!-- web.xml -->
<web-app>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/root-context.xml</param-value>
    </context-param>
    <listener>
        <listener-class>
            org.springframework.web.context.ContextLoaderListener
        </listener-class>
    </listener>
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>
            org.springframework.web.servlet.DispatcherServlet
        </servlet-class>
        <load-on-startup>1</load-on-startup>
        <!-- Uses default: /WEB-INF/dispatcher-servlet.xml -->
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

```xml
<!-- /WEB-INF/root-context.xml -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="...">

    <context:component-scan base-package="com.example">
        <context:exclude-filter type="annotation"
            expression="org.springframework.stereotype.Controller"/>
    </context:component-scan>

    <bean id="dataSource" 
          class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="org.postgresql.Driver"/>
        <property name="url" value="jdbc:postgresql://localhost/mydb"/>
        <property name="username" value="user"/>
        <property name="password" value="pass"/>
    </bean>

    <bean id="transactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <tx:annotation-driven transaction-manager="transactionManager"/>

</beans>
```

```xml
<!-- /WEB-INF/dispatcher-servlet.xml -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="...">

    <!-- THE most important element — registers MVC infrastructure -->
    <mvc:annotation-driven/>

    <!-- Only controllers in child context -->
    <context:component-scan base-package="com.example"
                            use-default-filters="false">
        <context:include-filter type="annotation"
            expression="org.springframework.stereotype.Controller"/>
    </context:component-scan>

    <!-- View resolver -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/views/"/>
        <property name="suffix" value=".jsp"/>
        <property name="order" value="2"/>
    </bean>

    <!-- Static resources -->
    <mvc:resources mapping="/static/**" location="/static/"/>

    <!-- Default servlet for unmatched requests -->
    <mvc:default-servlet-handler/>

    <!-- Interceptors -->
    <mvc:interceptors>
        <bean class="com.example.interceptor.LoggingInterceptor"/>
    </mvc:interceptors>

</beans>
```

---

### Style 2 — Annotation-Driven Hybrid

```xml
<!-- web.xml same as Style 1 -->

<!-- dispatcher-servlet.xml — uses annotations, minimal XML -->
<beans>
    <mvc:annotation-driven>
        <!-- Customize message converters within annotation-driven -->
        <mvc:message-converters>
            <bean class="org.springframework.http.converter.json
                         .MappingJackson2HttpMessageConverter">
                <property name="objectMapper">
                    <bean class="com.fasterxml.jackson.databind.ObjectMapper">
                        <property name="dateFormat">
                            <bean class="java.text.SimpleDateFormat">
                                <constructor-arg value="yyyy-MM-dd"/>
                            </bean>
                        </property>
                    </bean>
                </property>
            </bean>
        </mvc:message-converters>
    </mvc:annotation-driven>

    <context:component-scan base-package="com.example.web"/>

    <bean class="...InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/views/"/>
        <property name="suffix" value=".jsp"/>
    </bean>
</beans>
```

```java
// Controllers are Java classes — annotations do the mapping
@Controller
@RequestMapping("/products")
public class ProductController {

    @Autowired  // Resolved from root context
    private ProductService productService;

    @GetMapping
    public String list(Model model) {
        model.addAttribute("products", productService.findAll());
        return "product/list";
    }

    @PostMapping
    public String create(@Valid @ModelAttribute Product product,
                         BindingResult result) {
        if (result.hasErrors()) return "product/form";
        productService.save(product);
        return "redirect:/products";
    }
}
```

---

### Style 3 — Complete Pure Java Config

```java
// WebAppInitializer.java — replaces web.xml
public class WebAppInitializer
    extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{RootConfig.class};
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{WebConfig.class};
    }

    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }

    @Override
    protected Filter[] getServletFilters() {
        CharacterEncodingFilter cef = new CharacterEncodingFilter();
        cef.setEncoding("UTF-8");
        cef.setForceEncoding(true);

        HiddenHttpMethodFilter hmf = new HiddenHttpMethodFilter();
        // Allows PUT/DELETE from HTML forms via _method parameter

        return new Filter[]{cef, hmf};
    }

    @Override
    protected void customizeRegistration(
            ServletRegistration.Dynamic registration) {
        registration.setInitParameter(
            "throwExceptionIfNoHandlerFound", "true");
        registration.setMultipartConfig(
            new MultipartConfigElement(
                "/tmp",        // location
                5_242_880L,    // maxFileSize: 5MB
                20_971_520L,   // maxRequestSize: 20MB
                0              // fileSizeThreshold
            )
        );
    }
}
```

```java
// RootConfig.java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION,
        classes = {Controller.class, ControllerAdvice.class}
    )
)
@EnableTransactionManagement
@PropertySource("classpath:application.properties")
public class RootConfig {

    @Autowired
    private Environment env;

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(env.getProperty("db.url"));
        config.setUsername(env.getProperty("db.username"));
        config.setPassword(env.getProperty("db.password"));
        config.setMaximumPoolSize(20);
        return new HikariDataSource(config);
    }

    @Bean
    public PlatformTransactionManager transactionManager(DataSource ds) {
        return new DataSourceTransactionManager(ds);
    }
}
```

```java
// WebConfig.java — complete production-grade configuration
@Configuration
@EnableWebMvc
@ComponentScan(
    basePackages = "com.example",
    useDefaultFilters = false,
    includeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION,
        classes = {Controller.class, ControllerAdvice.class}
    )
)
public class WebConfig implements WebMvcConfigurer {

    // ── VIEW RESOLUTION ────────────────────────────────────
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.jsp("/WEB-INF/views/", ".jsp");
        // Equivalent to InternalResourceViewResolver bean
    }

    // ── STATIC RESOURCES ──────────────────────────────────
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("/static/", "classpath:/static/")
                .setCachePeriod(3600)
                .resourceChain(true)
                .addResolver(new VersionResourceResolver()
                    .addContentVersionStrategy("/**"));
    }

    // ── INTERCEPTORS ──────────────────────────────────────
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoggingInterceptor())
                .addPathPatterns("/**")
                .excludePathPatterns("/static/**", "/error");

        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/admin/**");
    }

    // ── CORS ──────────────────────────────────────────────
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://app.example.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }

    // ── MESSAGE CONVERTERS ────────────────────────────────
    @Override
    public void extendMessageConverters(
            List<HttpMessageConverter<?>> converters) {
        // Find and customize Jackson converter
        converters.stream()
            .filter(c -> c instanceof MappingJackson2HttpMessageConverter)
            .map(c -> (MappingJackson2HttpMessageConverter) c)
            .findFirst()
            .ifPresent(c -> {
                ObjectMapper mapper = c.getObjectMapper();
                mapper.setSerializationInclusion(
                    JsonInclude.Include.NON_NULL);
                mapper.configure(
                    DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, 
                    false);
            });
    }

    // ── CONTENT NEGOTIATION ───────────────────────────────
    @Override
    public void configureContentNegotiation(
            ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(true)
            .parameterName("format")
            .defaultContentType(MediaType.APPLICATION_JSON)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML);
    }

    // ── ASYNC SUPPORT ─────────────────────────────────────
    @Override
    public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
        configurer.setDefaultTimeout(30_000L); // 30 seconds
        configurer.setTaskExecutor(asyncTaskExecutor());
    }

    @Bean
    public ThreadPoolTaskExecutor asyncTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        return executor;
    }

    // ── VIEW CONTROLLERS (no @Controller needed) ──────────
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("home");
        registry.addViewController("/login").setViewName("auth/login");
        registry.addRedirectViewController("/home", "/");
        registry.addStatusController("/health", HttpStatus.OK);
    }

    // ── DEFAULT SERVLET ───────────────────────────────────
    @Override
    public void configureDefaultServletHandling(
            DefaultServletHandlerConfigurer configurer) {
        configurer.enable(); // Delegates unmatched to container default servlet
    }

    // ── PATH MATCHING ─────────────────────────────────────
    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer.setUseTrailingSlashMatch(false);
        // /products and /products/ treated as different URLs
    }
}
```

---

### Edge Case — configureMessageConverters Removes Defaults

```java
@Configuration
@EnableWebMvc
public class BrokenConfig implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(
            List<HttpMessageConverter<?>> converters) {
        // BUG: This REPLACES all defaults
        // After this, only XML converter exists
        // All REST endpoints returning JSON → 406 Not Acceptable
        converters.add(new MappingJackson2XmlHttpMessageConverter());
    }
}

// FIX: Use extendMessageConverters() to ADD to defaults
@Override
public void extendMessageConverters(
        List<HttpMessageConverter<?>> converters) {
    converters.add(new MappingJackson2XmlHttpMessageConverter());
    // JSON still works — XML support added
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
What does `<mvc:annotation-driven/>` register that enables `@RequestMapping` methods to be discovered and invoked?

A) `DefaultAnnotationHandlerMapping` and `AnnotationMethodHandlerAdapter`
B) `RequestMappingHandlerMapping` and `RequestMappingHandlerAdapter`
C) `BeanNameUrlHandlerMapping` and `SimpleControllerHandlerAdapter`
D) `RouterFunctionMapping` and `HandlerFunctionAdapter`

**Answer: B**
`<mvc:annotation-driven/>` registers `RequestMappingHandlerMapping` and `RequestMappingHandlerAdapter` — the modern replacements for the deprecated `DefaultAnnotationHandlerMapping` and `AnnotationMethodHandlerAdapter`. Without these, `@RequestMapping` methods are not processed.

---

**Q2 — Select All That Apply**
Which beans are registered by `@EnableWebMvc` / `<mvc:annotation-driven/>`?

A) `RequestMappingHandlerMapping`
B) `InternalResourceViewResolver`
C) `ExceptionHandlerExceptionResolver`
D) `MappingJackson2HttpMessageConverter` (if Jackson on classpath)
E) `ContextLoaderListener`
F) `DefaultFormattingConversionService`

**Answer: A, C, D, F**
`InternalResourceViewResolver` is NOT registered by `@EnableWebMvc` — you must define it separately. `ContextLoaderListener` is a Servlet listener, not a Spring bean. A, C, D (conditional), F are all registered.

---

**Q3 — Code Prediction**
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(
            List<HttpMessageConverter<?>> converters) {
        converters.add(new StringHttpMessageConverter());
    }
}

@RestController
public class TestController {
    @GetMapping("/data")
    public Product getData() {
        return new Product("Widget", 9.99);
    }
}
```
A GET to `/data` with `Accept: application/json`. What happens?

A) Returns JSON representation of Product
B) HTTP 406 Not Acceptable
C) HTTP 500 — Jackson not configured
D) Returns string "Product{...}"

**Answer: B**
`configureMessageConverters()` REPLACES all default converters. After this call, only `StringHttpMessageConverter` exists. Request asks for `application/json`. No converter can produce `application/json` from `Product`. Result: 406 Not Acceptable.

---

**Q4 — MCQ**
Which mechanism allows Spring MVC to work without `web.xml` in a Servlet 3.0+ container?

A) Spring scans for `@WebServlet` annotations at startup
B) `SpringServletContainerInitializer` is discovered via `ServiceLoader` from `META-INF/services`
C) `AbstractDispatcherServletInitializer` auto-registers itself via reflection
D) Tomcat has built-in Spring MVC support since version 7

**Answer: B**
The Java `ServiceLoader` mechanism reads `META-INF/services/jakarta.servlet.ServletContainerInitializer`. Spring's `spring-web.jar` includes this file pointing to `SpringServletContainerInitializer`. Tomcat finds it, calls `onStartup()`, which discovers `WebApplicationInitializer` implementations and triggers Spring's programmatic servlet registration.

---

**Q5 — Scenario**
A developer uses `configureViewResolvers()` in `WebMvcConfigurer` AND also defines an `InternalResourceViewResolver` `@Bean` method in the same config class. Both point to different prefixes. What happens?

A) Both view resolvers active — ordered resolution applies
B) `@Bean` definition wins — `configureViewResolvers()` ignored
C) `configureViewResolvers()` wins — `@Bean` definition ignored
D) `BeanDefinitionStoreException` — conflicting view resolver definitions

**Answer: A**
Both are registered. Spring MVC chains all `ViewResolver` beans. `configureViewResolvers()` registers via `WebMvcConfigurationSupport`. The `@Bean` method registers a separate bean. Both exist, both are tried. Order matters — if not specified, unpredictable which resolves first. This is a configuration mistake that doesn't cause errors but causes confusing behavior.

---

**Q6 — True/False**
`AbstractAnnotationConfigDispatcherServletInitializer` always sets `load-on-startup=1` for `DispatcherServlet`.

**Answer: True**
`AbstractDispatcherServletInitializer.onStartup()` calls `registration.setLoadOnStartup(1)` unconditionally. There is no way to change this without overriding the method. The child context always initializes at startup, never on first request. This is intentional — startup failures are immediately visible.

---

**Q7 — Select All That Apply**
Which of the following correctly describe differences between `configureMessageConverters()` and `extendMessageConverters()` in `WebMvcConfigurer`?

A) `configureMessageConverters()` prevents default converters from being added
B) `extendMessageConverters()` is called after defaults are added — allows modification
C) Both methods have identical behavior
D) Using `configureMessageConverters()` with an empty list results in no converters
E) `extendMessageConverters()` replaces all converters with your list

**Answer: A, B, D**
C is wrong — completely different behavior. E is wrong — `extendMessageConverters()` modifies the existing list, not replaces. A is correct — `configureMessageConverters()` populates the converter list and skips default population. B is correct — `extendMessageConverters()` is invoked after defaults are added. D is correct — empty `configureMessageConverters()` = no converters at all.

---

**Q8 — Ordering**
Order these Java Config initialization events from first to last:

- `WebMvcConfigurer.addInterceptors()` called
- `AnnotationConfigWebApplicationContext` created for root
- `context.refresh()` called on child context
- `WebApplicationInitializer.onStartup()` called
- `DispatcherServlet` registered with `ServletContext`
- `initStrategies()` called

**Correct Order:**
1. `WebApplicationInitializer.onStartup()` called
2. `AnnotationConfigWebApplicationContext` created for root
3. `DispatcherServlet` registered with `ServletContext`
4. `context.refresh()` called on child context
5. `WebMvcConfigurer.addInterceptors()` called (during child context refresh)
6. `initStrategies()` called (after child context refresh)

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `<mvc:annotation-driven/>` is optional**
Without it, `@RequestMapping` methods are NOT discovered by the modern handler mapping. Spring falls back to `DispatcherServlet.properties` defaults, which DO include `RequestMappingHandlerMapping` — so `@RequestMapping` still works at the basic level. But `@ResponseBody`, `@Valid`, `@ExceptionHandler` advanced features DO require the converters and resolvers registered by `<mvc:annotation-driven/>`. This nuance trips many candidates.

**Trap 2 — `@EnableWebMvc` in Spring Boot disables auto-config**
Covered in 1.5 but critical here: adding `@EnableWebMvc` to any `@Configuration` class in Spring Boot disables `WebMvcAutoConfiguration`. The symptom: Jackson stops working, static resources stop serving, view resolvers disappear. The fix: `implements WebMvcConfigurer` without `@EnableWebMvc`.

**Trap 3 — `configureMessageConverters` vs `extendMessageConverters`**
This is the most commonly tested `WebMvcConfigurer` trap. `configure...` = replace. `extend...` = add to. Mixing them up removes Jackson from the converter chain — all REST endpoints return 406.

**Trap 4 — `@ControllerAdvice` in component scan filter**
Many developers only exclude `@Controller` from the root context scan but forget `@ControllerAdvice`. `@ControllerAdvice` is NOT annotated with `@Controller` — it's annotated with `@Component`. If it ends up in the root context, it won't be found by the child context's `ExceptionHandlerExceptionResolver` (which only looks in the child context). Global exception handling silently fails for all exceptions.

**Trap 5 — `addViewControllers` vs `@Controller`**
`addViewControllers()` registers URL-to-view mappings with no Java code. But these are handled by `ParameterizableViewController` — they can only forward to views. They cannot add model attributes, access services, or do any logic. Examiners present a `addViewController` setup and ask why model attributes aren't available in the view — because there's no controller to add them.

**Trap 6 — Multiple `WebApplicationInitializer` discovery order**
If multiple `WebApplicationInitializer` classes exist, they are sorted by `@Order` / `Ordered`. Unordered initializers run in unpredictable order. This matters when one initializer depends on setup from another. Always annotate with `@Order` when using multiple initializers.

---

## 5️⃣ SUMMARY SHEET

```
THREE CONFIGURATION STYLES
─────────────────────────────────────────────────────
XML:        web.xml + root-context.xml + dispatcher-servlet.xml
            <mvc:annotation-driven/> required
            <context:component-scan/> for bean discovery

Hybrid:     web.xml + XML namespace + Java @Controller classes
            Annotations on classes, XML for infrastructure

Java Config: WebApplicationInitializer (replaces web.xml)
             @Configuration + @EnableWebMvc (replaces XML namespace)
             WebMvcConfigurer (customization API)
             AbstractAnnotationConfigDispatcherServletInitializer (convenience)

XML ↔ JAVA CONFIG EQUIVALENTS
─────────────────────────────────────────────────────
<mvc:annotation-driven/>        → @EnableWebMvc
<context:component-scan/>       → @ComponentScan
<mvc:interceptors/>             → addInterceptors()
<mvc:resources/>                → addResourceHandlers()
<mvc:default-servlet-handler/>  → configureDefaultServletHandling()
<mvc:cors/>                     → addCorsMappings()
<tx:annotation-driven/>         → @EnableTransactionManagement

WHAT @EnableWebMvc / <mvc:annotation-driven/> REGISTERS
─────────────────────────────────────────────────────
RequestMappingHandlerMapping     (discovers @RequestMapping)
RequestMappingHandlerAdapter     (invokes @RequestMapping methods)
ExceptionHandlerExceptionResolver (@ExceptionHandler support)
ResponseStatusExceptionResolver  (@ResponseStatus support)
DefaultHandlerExceptionResolver  (standard Spring MVC exceptions)
HttpMessageConverters            (Jackson, String, ByteArray, etc.)
ConversionService                (type conversion)
Validator                        (JSR-303 if on classpath)

DOES NOT register: InternalResourceViewResolver

WEBMVCCONFIGURER KEY DISTINCTIONS
─────────────────────────────────────────────────────
configureMessageConverters()  → REPLACES all default converters (danger!)
extendMessageConverters()     → ADDS TO default converters (safe)
configureHandlerExceptionResolvers() → REPLACES defaults
extendHandlerExceptionResolvers()    → ADDS TO defaults

ZERO-XML MECHANISM
─────────────────────────────────────────────────────
spring-web.jar/META-INF/services/jakarta.servlet.ServletContainerInitializer
  → SpringServletContainerInitializer
  → Finds WebApplicationInitializer implementations
  → Calls onStartup() → programmatic servlet/filter registration

AbstractAnnotationConfigDispatcherServletInitializer:
  → Always sets load-on-startup=1
  → Creates AnnotationConfigWebApplicationContext for both contexts
  → getRootConfigClasses() returning null = no root context (valid)

SPRING BOOT MVC INITIALIZATION
─────────────────────────────────────────────────────
SpringApplication.run()
  → Single AnnotationConfigServletWebServerApplicationContext
  → DispatcherServletAutoConfiguration → DispatcherServlet as @Bean
  → WebMvcAutoConfiguration → MVC infrastructure
  → EmbeddedWebServerFactory → Tomcat/Jetty/Undertow
  → NO web.xml, NO ContextLoaderListener, NO hierarchy

Boot customization: implements WebMvcConfigurer (NO @EnableWebMvc)

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "<mvc:annotation-driven/> registers HandlerMapping+Adapter+ExceptionResolvers+Converters"
• "configureMessageConverters() replaces defaults — use extendMessageConverters() to add"
• "Zero-XML works via ServiceLoader finding SpringServletContainerInitializer"
• "@EnableWebMvc = imports DelegatingWebMvcConfiguration = Java equivalent of annotation-driven"
• "AbstractAnnotationConfigDispatcherServletInitializer always sets load-on-startup=1"
• "InternalResourceViewResolver NOT registered by @EnableWebMvc — define it separately"
• "@ControllerAdvice must be in child context — not just @Controller"
• "addViewControllers() = no Java class needed = no model attributes possible"
```

---

## PART 1 COMPLETE ✅

You have now mastered all 8 topics of **Part 1 — Foundations & Architecture**:

```
1.1 What is Spring MVC — Philosophy & Design Goals          ✅
1.2 MVC Pattern — Model, View, Controller Deep Dive         ✅
1.3 Servlet API Fundamentals                                ✅
1.4 DispatcherServlet — The Front Controller                ✅
1.5 Spring MVC vs Spring Boot vs Spring WebFlux             ✅
1.6 ApplicationContext vs WebApplicationContext Hierarchy   ✅
1.7 ContextLoaderListener vs DispatcherServlet Context      ✅
1.8 Spring MVC Initialization — XML, Annotation, Java Config ✅
```

---
