# All Ways to Configure Root and Child (Web) ApplicationContext in Spring MVC

Let's build this up systematically. There are essentially **three eras** of Spring configuration — XML-based, Java-based, and the hybrid approach — and within each era, there are specific ways to wire up both the root and child contexts. By the end, you'll have a complete map of every approach and understand *why* each one exists.

---

## The Mental Model First

Before looking at any code, understand what you're always configuring, regardless of the approach:

You are always answering two questions. First, *how do I tell `ContextLoaderListener` what beans belong in the root context?* Second, *how do I tell `DispatcherServlet` what beans belong in the child (web) context?* Every configuration approach is just a different syntax for answering those same two questions.

The "wiring point" — where you declare these answers to the servlet container — is always either `web.xml` (the traditional approach) or a class that implements `WebApplicationInitializer` (the modern, no-XML approach). Let's go through all combinations.

---

## Approach 1 — Pure XML Configuration (The Classic Way)

This is how Spring MVC was configured for its first several years. Everything is declared in `web.xml`, and the application context configuration lives in XML files.

### The web.xml file

```xml
<web-app>

    <!--
        PART 1: ROOT CONTEXT SETUP
        Tell ContextLoaderListener WHERE to find the root context config.
        This <context-param> is read by ContextLoaderListener on startup.
    -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <!-- You can list multiple files separated by comma or newline -->
        <param-value>
            /WEB-INF/spring/root-context.xml,
            /WEB-INF/spring/security-context.xml
        </param-value>
    </context-param>

    <!-- This listener actually CREATES the root ApplicationContext -->
    <listener>
        <listener-class>
            org.springframework.web.context.ContextLoaderListener
        </listener-class>
    </listener>

    <!--
        PART 2: CHILD (WEB) CONTEXT SETUP
        DispatcherServlet creates its own child WebApplicationContext.
        Its config file is resolved by convention OR by explicit init-param.
    -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>
            org.springframework.web.servlet.DispatcherServlet
        </servlet-class>
        <init-param>
            <!--
                If you omit this, Spring looks for a file named:
                /WEB-INF/{servlet-name}-servlet.xml
                So here it would look for /WEB-INF/dispatcher-servlet.xml
                by default. Explicit declaration overrides the convention.
            -->
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/spring/web-context.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

</web-app>
```

Notice the subtle difference between `<context-param>` (for root context — application-wide) and `<init-param>` (for the servlet's own context). This distinction matters because `<context-param>` is visible to all servlets via `ServletContext`, while `<init-param>` is scoped to the specific servlet.

### The root-context.xml file

```xml
<!-- /WEB-INF/spring/root-context.xml -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx">

    <!--
        Component scan for EVERYTHING except @Controller.
        This ensures services and repositories end up here,
        not in the web context.
    -->
    <context:component-scan base-package="com.example">
        <context:exclude-filter
            type="annotation"
            expression="org.springframework.stereotype.Controller"/>
    </context:component-scan>

    <!-- Infrastructure beans belong in root context -->
    <bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource">
        <property name="jdbcUrl" value="jdbc:postgresql://localhost/mydb"/>
    </bean>

    <bean id="transactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!-- Enable @Transactional processing -->
    <tx:annotation-driven transaction-manager="transactionManager"/>

</beans>
```

### The web-context.xml file

```xml
<!-- /WEB-INF/spring/web-context.xml -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc">

    <!-- Enable @RequestMapping, @PathVariable, etc. -->
    <mvc:annotation-driven/>

    <!--
        Scan ONLY @Controller classes.
        useDefaultFilters=false disables the default scan for
        @Component, @Service, @Repository — we only want controllers here.
    -->
    <context:component-scan
        base-package="com.example"
        use-default-filters="false">
        <context:include-filter
            type="annotation"
            expression="org.springframework.stereotype.Controller"/>
    </context:component-scan>

    <!-- Web-specific infrastructure -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/views/"/>
        <property name="suffix" value=".jsp"/>
    </bean>

</beans>
```

The key insight here is the **convention over configuration** principle: if you name your servlet `dispatcher` in `web.xml` and don't provide `contextConfigLocation`, Spring automatically looks for `/WEB-INF/dispatcher-servlet.xml`. This is a shortcut many older projects relied on.

---

## Approach 2 — XML web.xml + Java @Configuration Classes

This is a hybrid that became popular during the transition period when teams wanted Java config for their beans but weren't ready to abandon `web.xml` entirely. You're still using `web.xml` for bootstrapping, but you tell Spring to use a Java class instead of an XML file as the configuration source.

The trick is telling Spring *which type of context* to create — specifically, `AnnotationConfigWebApplicationContext`, which understands `@Configuration` classes rather than XML files.

```xml
<!-- web.xml — hybrid approach -->
<web-app>

    <!--
        PART 1: ROOT CONTEXT
        We tell Spring two things:
        1. Use AnnotationConfigWebApplicationContext (not the default XmlWebApplicationContext)
        2. The configuration class to use
    -->
    <context-param>
        <param-name>contextClass</param-name>
        <param-value>
            org.springframework.web.context.support.AnnotationConfigWebApplicationContext
        </param-value>
    </context-param>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <!-- Instead of an XML file path, this is now a fully-qualified class name -->
        <param-value>com.example.config.RootConfig</param-value>
    </context-param>

    <listener>
        <listener-class>
            org.springframework.web.context.ContextLoaderListener
        </listener-class>
    </listener>

    <!--
        PART 2: CHILD (WEB) CONTEXT
        Same idea — tell DispatcherServlet to use Java config
    -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>
            org.springframework.web.servlet.DispatcherServlet
        </servlet-class>
        <init-param>
            <param-name>contextClass</param-name>
            <param-value>
                org.springframework.web.context.support.AnnotationConfigWebApplicationContext
            </param-value>
        </init-param>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <!-- Again, a class name instead of an XML path -->
            <param-value>com.example.config.WebConfig</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

</web-app>
```

And then your `RootConfig` and `WebConfig` are normal `@Configuration` classes exactly like you saw in the previous topic.

```java
// RootConfig.java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    excludeFilters = @Filter(type = FilterType.ANNOTATION, classes = Controller.class)
)
@EnableTransactionManagement
public class RootConfig {

    @Bean
    public DataSource dataSource() { /* ... */ }

    @Bean
    public PlatformTransactionManager transactionManager() { /* ... */ }
}
```

```java
// WebConfig.java
@Configuration
@EnableWebMvc
@ComponentScan(
    basePackages = "com.example",
    useDefaultFilters = false,
    includeFilters = @Filter(type = FilterType.ANNOTATION, classes = Controller.class)
)
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver r = new InternalResourceViewResolver();
        r.setPrefix("/WEB-INF/views/");
        r.setSuffix(".jsp");
        return r;
    }
}
```

The important thing to notice in the hybrid approach is the `contextClass` parameter. Without it, `ContextLoaderListener` defaults to creating an `XmlWebApplicationContext`. By explicitly declaring `AnnotationConfigWebApplicationContext`, you're switching the *type* of container it creates, which then understands `@Configuration` classes as input rather than XML file paths.

---

## Approach 3 — Pure Java Configuration with WebApplicationInitializer (No web.xml)

Starting from Servlet 3.0 (Tomcat 7+), the servlet specification introduced the `ServletContainerInitializer` interface, which allows a library to provide startup code that the servlet container automatically discovers and runs — no `web.xml` needed. Spring leverages this with its `SpringServletContainerInitializer`, which in turn looks for implementations of Spring's own `WebApplicationInitializer` interface.

This means you can replace `web.xml` entirely with a Java class. The lowest-level way to do this is to implement `WebApplicationInitializer` directly.

```java
// This class replaces web.xml entirely.
// Spring's SpringServletContainerInitializer discovers and runs it on startup.
public class ManualWebAppInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {

        // ─── STEP 1: Create and configure the ROOT context ───────────────────
        AnnotationConfigWebApplicationContext rootContext =
            new AnnotationConfigWebApplicationContext();
        rootContext.register(RootConfig.class); // Tell it which @Configuration class to use

        // Register ContextLoaderListener — this manages the root context lifecycle
        // (creates it on startup, destroys it on shutdown)
        servletContext.addListener(new ContextLoaderListener(rootContext));

        // ─── STEP 2: Create and configure the CHILD (web) context ─────────────
        AnnotationConfigWebApplicationContext webContext =
            new AnnotationConfigWebApplicationContext();
        webContext.register(WebConfig.class);

        // ─── STEP 3: Create DispatcherServlet with the child context ──────────
        DispatcherServlet dispatcherServlet = new DispatcherServlet(webContext);

        // Register it with the servlet container programmatically
        ServletRegistration.Dynamic registration =
            servletContext.addServlet("dispatcher", dispatcherServlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}
```

This is the most verbose but also the most transparent approach — you can see every wire being connected. The root context goes into `ContextLoaderListener`, the web context goes into `DispatcherServlet`, and you register both with the `ServletContext` manually. Nothing is hidden.

---

## Approach 4 — AbstractContextLoaderInitializer (Partial Abstraction)

Spring noticed that the `WebApplicationInitializer` approach, while powerful, involved a lot of repetitive boilerplate. So it provided a hierarchy of abstract base classes. The first level of abstraction handles only the root context setup.

```java
// This base class handles ContextLoaderListener setup for you.
// You only need to provide the root context.
public class MyWebAppInitializer extends AbstractContextLoaderInitializer {

    @Override
    protected WebApplicationContext createRootApplicationContext() {
        // You create and configure the root context here.
        // The base class handles registering it with ContextLoaderListener.
        AnnotationConfigWebApplicationContext rootContext =
            new AnnotationConfigWebApplicationContext();
        rootContext.register(RootConfig.class);
        return rootContext;
    }

    // Notice: you still need to set up DispatcherServlet yourself
    // if you extend this class directly. It only helps with the root context.
}
```

This partial abstraction is rarely used directly because it doesn't help with the `DispatcherServlet` setup. It's more of a stepping stone to the next level.

---

## Approach 5 — AbstractDispatcherServletInitializer (More Abstraction)

This class extends the previous one and also handles the `DispatcherServlet` registration. This is the level most applications would use if they want fine-grained control while still eliminating boilerplate.

```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    // ROOT CONTEXT — handled by the parent class's ContextLoaderListener wiring
    @Override
    protected WebApplicationContext createRootApplicationContext() {
        AnnotationConfigWebApplicationContext rootContext =
            new AnnotationConfigWebApplicationContext();
        rootContext.register(RootConfig.class);
        return rootContext;
    }

    // CHILD (WEB) CONTEXT — the base class creates DispatcherServlet with this
    @Override
    protected WebApplicationContext createServletApplicationContext() {
        AnnotationConfigWebApplicationContext webContext =
            new AnnotationConfigWebApplicationContext();
        webContext.register(WebConfig.class);
        return webContext;
    }

    // Which URLs should DispatcherServlet handle?
    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }

    // Optional: add filters to the DispatcherServlet
    @Override
    protected Filter[] getServletFilters() {
        return new Filter[]{new CharacterEncodingFilter("UTF-8", true)};
    }
}
```

The base class uses the context you return from `createRootApplicationContext()` to register a `ContextLoaderListener`, and uses the context from `createServletApplicationContext()` to instantiate `DispatcherServlet`. You've gone from 40 lines of manual wiring to about 15 lines of focused configuration.

---

## Approach 6 — AbstractAnnotationConfigDispatcherServletInitializer (The Most Common Modern Way)

This is the highest-level abstraction Spring provides, and it's what you'll see in most modern plain-Spring (non-Boot) applications. It assumes you're using `@Configuration` classes (not XML), and reduces your setup to simply *naming* your config classes.

```java
// This is the most concise way to configure the two-context hierarchy.
// Extend this class and provide three things:
//   1. Which @Configuration classes build the ROOT context
//   2. Which @Configuration classes build the CHILD context
//   3. Which URL patterns DispatcherServlet should handle
public class MyWebAppInitializer
    extends AbstractAnnotationConfigDispatcherServletInitializer {

    // ROOT context configuration classes
    // ContextLoaderListener will create an AnnotationConfigWebApplicationContext
    // using these classes — no need to instantiate it yourself
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[]{
            RootConfig.class,
            SecurityConfig.class,  // Multiple config classes are supported
            PersistenceConfig.class
        };
    }

    // CHILD (web) context configuration classes
    // DispatcherServlet will create its context using these
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[]{WebConfig.class};
    }

    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }

    // --- Optional customizations ---

    // Give the servlet a specific name (default is "dispatcher")
    @Override
    protected String getServletName() {
        return "myApp";
    }

    // Add servlet filters
    @Override
    protected Filter[] getServletFilters() {
        CharacterEncodingFilter encodingFilter = new CharacterEncodingFilter();
        encodingFilter.setEncoding("UTF-8");
        encodingFilter.setForceEncoding(true);
        return new Filter[]{encodingFilter};
    }

    // Whether DispatcherServlet should handle 404s itself (true)
    // or pass them to the container (false)
    @Override
    protected boolean isAsyncSupported() {
        return true;
    }
}
```

What's happening under the hood is exactly the same as Approach 3 — `ContextLoaderListener` gets the root context, `DispatcherServlet` gets the child context. But the base class handles all the instantiation and wiring. Your class just declares *what* goes where, not *how* to wire it.

---

## Approach 7 — Splitting Configuration Across Multiple Classes (Modular Java Config)

In real applications, you rarely put everything in a single `RootConfig` class. Spring's `@Import` and `@ComponentScan` give you clean ways to split configuration modularly. This isn't a separate bootstrapping approach — it works with any of the Java config methods above — but it's an important configuration pattern to know.

```java
// RootConfig acts as the entry point and imports focused sub-configurations
@Configuration
@Import({
    PersistenceConfig.class,   // DataSource, JPA, transactions
    SecurityConfig.class,       // Spring Security
    CachingConfig.class,        // Cache manager
    MessagingConfig.class       // JMS / messaging
})
@ComponentScan(
    basePackages = "com.example",
    excludeFilters = @Filter(type = FilterType.ANNOTATION, classes = Controller.class)
)
public class RootConfig {
    // RootConfig itself might be empty — it's just an aggregator
}
```

```java
// PersistenceConfig handles only persistence concerns
@Configuration
@EnableTransactionManagement
public class PersistenceConfig {

    // Reads from application.properties or environment
    @Value("${db.url}")
    private String dbUrl;

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(dbUrl);
        return ds;
    }

    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

```java
// WebConfig can also be split — here it uses @EnableWebMvc
// and delegates detailed MVC customization to helper methods
@Configuration
@EnableWebMvc
@ComponentScan(
    basePackages = "com.example.web",
    useDefaultFilters = false,
    includeFilters = @Filter(type = FilterType.ANNOTATION, classes = Controller.class)
)
public class WebConfig implements WebMvcConfigurer {

    // Resource handling — static files
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("/WEB-INF/static/");
    }

    // View resolution
    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver r = new InternalResourceViewResolver();
        r.setPrefix("/WEB-INF/views/");
        r.setSuffix(".jsp");
        return r;
    }

    // CORS configuration
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://myapp.com");
    }
}
```

---

## Approach 8 — Property Placeholder and Environment-Based Configuration

Another important configuration pattern is externalizing values so your config works in different environments (dev, staging, production). This applies to both the root and web contexts.

```java
@Configuration
@PropertySource("classpath:application.properties")  // Load properties file
@EnableTransactionManagement
public class RootConfig {

    // Spring's Environment abstraction — injects the resolved property value
    @Autowired
    private Environment env;

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        // Reads from application.properties — falls back to default if missing
        ds.setJdbcUrl(env.getProperty("db.url", "jdbc:h2:mem:testdb"));
        ds.setUsername(env.getRequiredProperty("db.username")); // Throws if missing
        ds.setPassword(env.getRequiredProperty("db.password"));
        return ds;
    }
}
```

```java
// Alternatively, use @Value for field-level injection of properties
@Configuration
@PropertySource({"classpath:app.properties",
                 "classpath:app-${spring.profiles.active}.properties"})
public class PersistenceConfig {

    @Value("${db.url}")
    private String dbUrl;

    @Value("${db.pool.maxSize:10}") // 10 is the default if property is absent
    private int maxPoolSize;

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(dbUrl);
        ds.setMaximumPoolSize(maxPoolSize);
        return ds;
    }
}
```

---

## Approach 9 — Profile-Specific Configuration

Spring Profiles let you conditionally activate entire configuration classes based on the runtime environment. This is particularly valuable for swapping datasources between development and production.

```java
// This entire class is only activated when the "dev" profile is active
@Configuration
@Profile("dev")
public class DevDataSourceConfig {

    @Bean
    public DataSource dataSource() {
        // In dev, use an in-memory H2 database
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .addScript("classpath:schema.sql")
            .addScript("classpath:test-data.sql")
            .build();
    }
}

@Configuration
@Profile("prod")
public class ProdDataSourceConfig {

    @Bean
    public DataSource dataSource() {
        // In production, use the real database
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(System.getenv("DATABASE_URL"));
        return ds;
    }
}
```

You activate a profile by setting `spring.profiles.active` as a system property (`-Dspring.profiles.active=dev`) or as a `ServletContext` init parameter in `web.xml`.

---

## Approach 10 — Programmatic Context Configuration Using GenericWebApplicationContext

This is the most raw, low-level approach — rarely used in production but worth knowing. It lets you configure a context entirely programmatically, registering bean definitions without any annotations or XML.

```java
public class ProgrammaticWebAppInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {

        // GenericWebApplicationContext lets you register beans programmatically
        GenericWebApplicationContext rootContext = new GenericWebApplicationContext();

        // Register a bean definition manually
        RootBeanDefinition serviceDef = new RootBeanDefinition(ProductService.class);
        rootContext.registerBeanDefinition("productService", serviceDef);

        // You can mix this with component scanning too
        ClassPathBeanDefinitionScanner scanner =
            new ClassPathBeanDefinitionScanner(rootContext);
        scanner.addExcludeFilter(
            new AnnotationTypeFilter(Controller.class));
        scanner.scan("com.example");

        servletContext.addListener(new ContextLoaderListener(rootContext));

        // Child context setup (same pattern)
        GenericWebApplicationContext webContext = new GenericWebApplicationContext();
        // ... register web beans ...

        DispatcherServlet servlet = new DispatcherServlet(webContext);
        ServletRegistration.Dynamic reg = servletContext.addServlet("dispatcher", servlet);
        reg.setLoadOnStartup(1);
        reg.addMapping("/");
    }
}
```

This is mostly used in framework development or testing infrastructure — not application code. But understanding it solidifies your mental model of what the other approaches are doing behind the scenes.

---

## The Complete Picture

To tie it all together, here is how all the approaches map onto the two fundamental questions:

Every approach is answering *"how do I configure the root context?"* and *"how do I configure the child context?"* The oldest approaches (pure XML, hybrid) use `web.xml` as the declaration point. The modern approaches replace `web.xml` with Java classes that implement `WebApplicationInitializer` at varying levels of abstraction — from the verbose but transparent manual wiring of Approach 3, all the way up to the clean and concise `AbstractAnnotationConfigDispatcherServletInitializer` of Approach 6.

In practice, for a new plain-Spring MVC project today (without Spring Boot), **Approach 6** (`AbstractAnnotationConfigDispatcherServletInitializer`) with modular `@Configuration` classes (Approach 7) and externalized properties (Approach 8) is the recommended combination. It's the most expressive, least repetitive, and maps most directly to what Spring Boot itself does — just made explicit.

Would you like to go deeper into any of these approaches, or move on to how `DispatcherServlet` itself uses the child context to resolve handlers and views?
