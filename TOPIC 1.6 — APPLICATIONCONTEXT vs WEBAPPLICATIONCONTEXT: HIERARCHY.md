# TOPIC 1.6 — APPLICATIONCONTEXT vs WEBAPPLICATIONCONTEXT: HIERARCHY

---

## 1️⃣ CONCEPTUAL EXPLANATION

### Why This Topic Is Architecturally Critical

The two-context hierarchy is one of the most **misunderstood and most tested** concepts in Spring MVC certification. Bugs caused by misunderstanding this hierarchy are among the most common in real enterprise applications — beans not found, AOP not applied, transactions not working in web layer, duplicate bean instantiation.

You must understand this hierarchy with surgical precision.

---

### The Core Concept — Two Containers, One Hierarchy

Spring MVC runs with **two separate Spring IoC containers** in a parent-child relationship:

```
┌─────────────────────────────────────────────────────────┐
│              ROOT ApplicationContext                    │
│         (Parent — created by ContextLoaderListener)    │
│                                                         │
│  Contains:                                              │
│  • @Service beans                                       │
│  • @Repository beans                                    │
│  • @Component beans (non-web)                           │
│  • DataSource, TransactionManager                       │
│  • Security configuration                               │
│  • Any bean shared across the whole application         │
│                                                         │
│  Stored in: ServletContext attribute                    │
│  Key: WebApplicationContext.ROOT_WEB_APPLICATION_       │
│       CONTEXT_ATTRIBUTE                                 │
└────────────────────┬────────────────────────────────────┘
                     │ parent reference
                     │ child CAN see parent beans
                     │ parent CANNOT see child beans
                     ▼
┌─────────────────────────────────────────────────────────┐
│           CHILD WebApplicationContext                   │
│      (Created by DispatcherServlet during init())      │
│                                                         │
│  Contains:                                              │
│  • @Controller beans                                    │
│  • ViewResolver beans                                   │
│  • HandlerMapping beans                                 │
│  • HandlerAdapter beans                                 │
│  • Interceptors                                         │
│  • Web-specific beans only                              │
│                                                         │
│  Stored in: ServletContext attribute                    │
│  Key: FrameworkServlet.SERVLET_CONTEXT_PREFIX +         │
│       servlet-name + WebApplicationContext.class        │
└─────────────────────────────────────────────────────────┘
```

---

### ApplicationContext — The Root Interface

`ApplicationContext` is Spring's central interface for the IoC container. Its full interface hierarchy:

```
BeanFactory (basic container)
    └── ApplicationContext
            ├── ListableBeanFactory
            ├── HierarchicalBeanFactory  ← parent-child support
            ├── MessageSource            ← i18n
            ├── ApplicationEventPublisher ← events
            ├── ResourcePatternResolver  ← resource loading
            └── EnvironmentCapable       ← environment/profiles

WebApplicationContext extends ApplicationContext
    └── ConfigurableWebApplicationContext
            └── AnnotationConfigWebApplicationContext  ← Java config
            └── XmlWebApplicationContext               ← XML config
            └── GenericWebApplicationContext           ← programmatic
```

`WebApplicationContext` adds:

```java
public interface WebApplicationContext extends ApplicationContext {
    
    String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = 
        WebApplicationContext.class.getName() + ".ROOT";
    
    // Scopes added by WebApplicationContext
    String SCOPE_REQUEST = "request";
    String SCOPE_SESSION = "session";
    String SCOPE_APPLICATION = "application";
    String SCOPE_GLOBAL_SESSION = "globalSession"; // Portlet legacy
    
    // Access to Servlet infrastructure
    ServletContext getServletContext();
}
```

The addition of `request`, `session`, and `application` scopes is what makes `WebApplicationContext` different from a plain `ApplicationContext`. A plain `ApplicationContext` cannot create request-scoped beans.

---

### Parent-Child Lookup Rules — The Exact Algorithm

When a bean is requested from the child context:

```
Child Context receives getBean("productService") request
        │
        ▼
Step 1: Look in child context's own bean registry
        │
        ├── Found? → Return it. DONE.
        │
        └── Not found?
                │
                ▼
        Step 2: Delegate to parent context
                │
                ├── Found in parent? → Return it. DONE.
                │
                └── Not found in parent?
                        │
                        ▼
                Throw NoSuchBeanDefinitionException
```

**Parent context cannot look into child:**
```
Parent Context receives getBean("homeController") request
        │
        ▼
Look in parent's own registry → NOT FOUND
        │
        ▼
Parent has NO child reference → cannot delegate down
        │
        ▼
Throw NoSuchBeanDefinitionException
```

This is a **strict one-way visibility**. Child sees parent. Parent is blind to child.

---

### Why This Hierarchy Exists — The Design Rationale

**Reason 1 — Multiple DispatcherServlets**
An enterprise app can have multiple `DispatcherServlet` instances (API servlet, admin servlet, etc.). Each has its OWN child context with its own controllers. But ALL share the SAME root context (services, repositories). The hierarchy enables this sharing without duplication.

```
Root Context: ProductService, OrderService, UserRepository (shared)
     │
     ├── API Child Context: ProductApiController, OrderApiController
     │
     └── Admin Child Context: AdminController, ReportController
```

**Reason 2 — Separation of Concerns**
Web layer beans (controllers, view resolvers) should not be visible to the service layer. A service should never depend on a controller — that would be an architectural violation. The hierarchy enforces this at the container level.

**Reason 3 — AOP and Transaction Scoping**
`@Transactional` on a service bean works because `TransactionManager` is in the root context, and the proxy is created there. If a service is accidentally defined in the child context instead, the transaction interceptor from the root context may not wrap it correctly, causing silent transaction failures.

---

### ContextLoaderListener — Creating the Root Context

```java
// What ContextLoaderListener does internally (simplified)
public class ContextLoaderListener implements ServletContextListener {
    
    @Override
    public void contextInitialized(ServletContextEvent event) {
        ServletContext servletContext = event.getServletContext();
        
        // Reads contextConfigLocation from <context-param>
        String configLocation = 
            servletContext.getInitParameter("contextConfigLocation");
        
        // Creates root WebApplicationContext
        WebApplicationContext rootContext = 
            createWebApplicationContext(servletContext);
        
        // Stores in ServletContext — makes it findable by DispatcherServlet
        servletContext.setAttribute(
            WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE,
            rootContext
        );
        
        // Refreshes context — instantiates all singleton beans
        ((ConfigurableApplicationContext) rootContext).refresh();
    }
    
    @Override
    public void contextDestroyed(ServletContextEvent event) {
        // Closes root context — destroys all beans, calls @PreDestroy
        WebApplicationContextUtils
            .getWebApplicationContext(event.getServletContext())
            .close();
    }
}
```

---

### DispatcherServlet — Creating the Child Context

```java
// Inside FrameworkServlet (parent of DispatcherServlet) — simplified
protected WebApplicationContext initWebApplicationContext() {
    
    // Step 1: Find root context (created by ContextLoaderListener)
    WebApplicationContext rootContext =
        WebApplicationContextUtils.getWebApplicationContext(
            getServletContext()
        );
    // This reads ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE from ServletContext
    
    // Step 2: Create child context
    WebApplicationContext wac = createWebApplicationContext(rootContext);
    // rootContext is set as the PARENT of the new child context
    
    // Step 3: Configure and refresh child context
    // Reads contextConfigLocation from <init-param>
    configureAndRefreshWebApplicationContext(wac);
    
    // Step 4: Store child context in ServletContext
    String attrName = getServletContextAttributeName();
    getServletContext().setAttribute(attrName, wac);
    
    return wac;
}
```

---

### The Duplicate Bean Problem — Most Common Enterprise Bug

```
// WRONG — Services defined in BOTH contexts
dispatcher-servlet.xml:
  <context:component-scan base-package="com.example"/>  ← Scans EVERYTHING

root-context.xml:
  <context:component-scan base-package="com.example"/>  ← Scans EVERYTHING again
```

**Result:**
- `ProductService` bean exists in root context AND child context
- `@Transactional` proxy for `ProductService` in root context exists
- Child context creates its OWN `ProductService` — NOT proxied by transaction manager
- Controller autowires from child context → gets NON-TRANSACTIONAL service
- Transactions silently don't work

**Correct approach:**
```
root-context.xml:
  <context:component-scan base-package="com.example">
    <context:exclude-filter type="annotation" 
      expression="org.springframework.stereotype.Controller"/>
  </context:component-scan>
  ← Scans everything EXCEPT @Controller

dispatcher-servlet.xml:
  <context:component-scan base-package="com.example" 
    use-default-filters="false">
    <context:include-filter type="annotation" 
      expression="org.springframework.stereotype.Controller"/>
  </context:component-scan>
  ← Scans ONLY @Controller
```

---

### Spring Boot — Single Context (No Hierarchy)

Spring Boot eliminates the two-context hierarchy entirely:

```
Spring Boot Application
        │
        ▼
Single AnnotationConfigServletWebServerApplicationContext
  (extends WebApplicationContext)
        │
        Contains EVERYTHING:
        • @Controller, @Service, @Repository
        • ViewResolvers, HandlerMappings
        • DataSource, TransactionManager
        • DispatcherServlet (registered as bean)
        • Embedded Tomcat (managed as bean)
```

**In Spring Boot:**
- `ContextLoaderListener` is NOT used
- There is NO root context / child context split
- ONE context contains all beans
- `DispatcherServlet` is registered as a bean WITHIN this context
- This simplification eliminates the duplicate bean problem entirely

This is a key reason Spring Boot is easier — the hierarchy bug class simply doesn't exist.

---

### WebApplicationContextUtils — Programmatic Access

```java
// Access root context from anywhere that has ServletContext
WebApplicationContext rootContext = 
    WebApplicationContextUtils.getWebApplicationContext(servletContext);

// Access root context — throws exception if not found (safer)
WebApplicationContext rootContext = 
    WebApplicationContextUtils.getRequiredWebApplicationContext(servletContext);

// Access from within a Spring-managed bean
@Autowired
private WebApplicationContext webApplicationContext;

// Access from request (set by DispatcherServlet)
WebApplicationContext wac = (WebApplicationContext) 
    request.getAttribute(DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE);
// This gives you the CHILD context (DispatcherServlet's own context)
```

---

### Bean Visibility Matrix — Complete Rules

| Bean Location | Visible To | Not Visible To |
|---|---|---|
| Root context | Root context beans, Child context beans (all) | Nothing outside app |
| Child context | Child context beans | Root context beans, Other child contexts |
| Child context bean | Can @Autowire from root context | Cannot @Autowire from sibling child context |
| Root context bean | Can only @Autowire from root context | Cannot @Autowire from any child context |

---

### Scope Registration — Where Scopes Are Available

```
Root ApplicationContext (plain)
  ├── Scopes: singleton, prototype
  └── NO request, session, application scopes

Child WebApplicationContext
  ├── Scopes: singleton, prototype (inherited)
  ├── Scopes: request, session, application (web-specific)
  └── These scopes registered by WebApplicationContext refresh
```

**Trap:** If you try to define a `@RequestScope` bean in the root context (before WebApplicationContext registers the scope), it will fail. Request-scoped beans must be in the child WebApplicationContext.

---

### ApplicationContext Events — Fired Separately

Both contexts fire their own events independently:

```
Root context refresh complete → ContextRefreshedEvent (from root)
Child context refresh complete → ContextRefreshedEvent (from child)

If a @EventListener listens for ContextRefreshedEvent:
  → It fires TWICE — once per context refresh
  → This is a common bug in initialization code
```

Fix:
```java
@EventListener
public void onContextRefresh(ContextRefreshedEvent event) {
    // Check which context fired the event
    if (event.getApplicationContext().getParent() == null) {
        // This is the ROOT context — no parent
        performRootInitialization();
    }
}
```

---

### Memory & Performance Implications

- Two contexts = two bean registries in memory — potential for wasted memory if beans are duplicated
- Child context holds reference to parent — parent is NOT GC'd until child is destroyed
- `ContextLoaderListener` root context: created at app startup, destroyed at shutdown
- Child context: created when `DispatcherServlet` initializes, destroyed when servlet is destroyed
- If `load-on-startup` is not set: child context created on FIRST request — cold start latency
- Bean lookup across parent-child: child checks self first (fast HashMap lookup), then delegates to parent (another HashMap lookup) — negligible overhead

---

### Practical Identification — How to Know Which Context a Bean Is In

```java
@Component
public class ContextInspector {
    
    @Autowired
    private ApplicationContext context;
    
    public void inspect() {
        // Is this a WebApplicationContext?
        if (context instanceof WebApplicationContext) {
            WebApplicationContext wac = (WebApplicationContext) context;
            System.out.println("ServletContext: " + wac.getServletContext());
        }
        
        // Is there a parent? If yes, this is the child context
        ApplicationContext parent = context.getParent();
        if (parent != null) {
            System.out.println("This is CHILD context");
            System.out.println("Parent: " + parent.getClass().getName());
        } else {
            System.out.println("This is ROOT context (no parent)");
        }
        
        // What beans are directly in this context?
        String[] localBeans = context.getBeanDefinitionNames();
        // Note: this does NOT include parent context beans
    }
}
```

---

## 2️⃣ CODE EXAMPLES

### Classic Two-Context Setup — XML

```xml
<!-- web.xml -->
<web-app>

    <!-- ROOT context configuration -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/root-context.xml</param-value>
    </context-param>
    
    <!-- Creates ROOT ApplicationContext -->
    <listener>
        <listener-class>
            org.springframework.web.context.ContextLoaderListener
        </listener-class>
    </listener>

    <!-- CHILD context — DispatcherServlet -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>
            org.springframework.web.servlet.DispatcherServlet
        </servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/dispatcher-servlet.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

</web-app>
```

```xml
<!-- root-context.xml — Services, Repositories, Infrastructure -->
<beans>
    <context:component-scan base-package="com.example">
        <!-- EXCLUDE controllers from root context -->
        <context:exclude-filter type="annotation"
            expression="org.springframework.stereotype.Controller"/>
    </context:component-scan>
    
    <bean id="dataSource" class="...HikariDataSource">...</bean>
    <bean id="transactionManager" class="...DataSourceTransactionManager">...</bean>
    <tx:annotation-driven transaction-manager="transactionManager"/>
</beans>
```

```xml
<!-- dispatcher-servlet.xml — Controllers, Web Infrastructure -->
<beans>
    <mvc:annotation-driven/>
    
    <!-- ONLY scan controllers -->
    <context:component-scan base-package="com.example" 
                            use-default-filters="false">
        <context:include-filter type="annotation"
            expression="org.springframework.stereotype.Controller"/>
    </context:component-scan>
    
    <bean class="...InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/views/"/>
        <property name="suffix" value=".jsp"/>
    </bean>
</beans>
```

---

### Classic Two-Context Setup — Java Config

```java
public class AppInitializer 
    extends AbstractAnnotationConfigDispatcherServletInitializer {

    // ROOT context configuration class
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{RootConfig.class};
    }

    // CHILD (DispatcherServlet) context configuration class
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{WebConfig.class};
    }

    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }
}
```

```java
// RootConfig — Root ApplicationContext
@Configuration
@ComponentScan(
    basePackages = "com.example",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION,
        classes = Controller.class
    )
)
@EnableTransactionManagement
public class RootConfig {

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost/mydb");
        return ds;
    }

    @Bean
    public PlatformTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dataSource());
    }
}
```

```java
// WebConfig — Child WebApplicationContext
@Configuration
@EnableWebMvc
@ComponentScan(
    basePackages = "com.example",
    useDefaultFilters = false,  // Only scan what we include
    includeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION,
        classes = Controller.class
    )
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

```java
// Controller — lives in CHILD context
// CAN autowire ProductService from ROOT context
@Controller
public class ProductController {

    // Autowired from ROOT context — parent lookup succeeds
    @Autowired
    private ProductService productService;

    @GetMapping("/products")
    public String list(Model model) {
        model.addAttribute("products", productService.findAll());
        return "product/list";
    }
}
```

```java
// Service — lives in ROOT context
// CANNOT autowire any @Controller
@Service
@Transactional
public class ProductService {

    @Autowired
    private ProductRepository repository; // Also in ROOT — works fine

    // @Autowired ProductController controller; // FAILS — child not visible to parent
    
    public List<Product> findAll() {
        return repository.findAll();
    }
}
```

---

### Duplicate Bean Bug — Demonstration

```java
// BUG: Both configs scan everything
@Configuration
@ComponentScan("com.example") // Scans @Service, @Repository, @Controller
public class RootConfig { }

@Configuration
@ComponentScan("com.example") // Scans SAME packages — DUPLICATE BEANS
@EnableWebMvc
public class WebConfig { }

// Result:
// ProductService in ROOT context — has @Transactional proxy
// ProductService in CHILD context — plain bean, NO transaction proxy
// ProductController autowires from CHILD → gets NON-TRANSACTIONAL ProductService
// Transactions silently fail — no error thrown
```

---

### Detecting Context Identity at Runtime

```java
@RestController
@RequestMapping("/debug")
public class ContextDebugController {

    @Autowired
    private ApplicationContext context;

    @GetMapping("/context")
    public Map<String, Object> contextInfo() {
        Map<String, Object> info = new LinkedHashMap<>();
        
        info.put("contextClass", context.getClass().getSimpleName());
        info.put("hasParent", context.getParent() != null);
        info.put("parentClass", context.getParent() != null ? 
            context.getParent().getClass().getSimpleName() : "none");
        info.put("beanCount", context.getBeanDefinitionCount());
        info.put("isWebContext", context instanceof WebApplicationContext);
        
        return info;
        // Output: {contextClass: AnnotationConfigWebApplicationContext,
        //          hasParent: true,
        //          parentClass: AnnotationConfigWebApplicationContext,
        //          beanCount: 15,
        //          isWebContext: true}
    }
}
```

---

### ContextRefreshedEvent — Fires Twice Trap

```java
// BUGGY — runs twice in two-context setup
@Component
public class StartupInitializer {

    @EventListener(ContextRefreshedEvent.class)
    public void onStartup(ContextRefreshedEvent event) {
        // Called once for ROOT context refresh
        // Called AGAIN for CHILD context refresh
        expensiveInitialization(); // RUNS TWICE
    }
}

// CORRECT — check for root context
@Component
public class StartupInitializer {

    @EventListener(ContextRefreshedEvent.class)
    public void onStartup(ContextRefreshedEvent event) {
        if (event.getApplicationContext().getParent() == null) {
            // Only root context has no parent
            expensiveInitialization(); // Runs ONCE
        }
    }
}
```

---

### Spring Boot — Single Context Verification

```java
// In a Spring Boot application
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = 
            SpringApplication.run(App.class, args);
        
        System.out.println(ctx.getClass().getName());
        // AnnotationConfigServletWebServerApplicationContext
        
        System.out.println(ctx.getParent());
        // null — no parent context in Spring Boot
        
        // DispatcherServlet is a BEAN inside this single context
        DispatcherServlet ds = ctx.getBean(DispatcherServlet.class);
        System.out.println(ds); // found — it's a bean
    }
}
```

---

### Edge Case — Request Scope in Wrong Context

```java
// In ROOT context config (RootConfig.java)
@Bean
@RequestScope // FAILS if root context is not a WebApplicationContext
public RequestScopedBean requestBean() {
    return new RequestScopedBean();
}

// Error at startup:
// IllegalStateException: No Scope registered for scope name 'request'
// Root ApplicationContext doesn't register web scopes

// CORRECT: Define request-scoped beans in WebConfig (child context)
// OR in Spring Boot (single WebApplicationContext) — works anywhere
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
In a two-context Spring MVC setup, a `@Service` bean defined in the root context needs to be `@Autowired` into a `@Controller` in the child context. What happens?

A) `NoSuchBeanDefinitionException` — parent beans not visible to child
B) The injection succeeds — child context delegates to parent when bean not found locally
C) A duplicate bean is created in the child context
D) `BeanCreationException` — cross-context injection is forbidden

**Answer: B**
Child context lookup delegates to parent when the bean is not found locally. This is the fundamental parent-child lookup rule. Controllers in child context can always inject services from root context.

---

**Q2 — Select All That Apply**
Which of the following are TRUE about Spring Boot's ApplicationContext?

A) Uses a two-context hierarchy like plain Spring MVC
B) Uses a single `AnnotationConfigServletWebServerApplicationContext`
C) `ContextLoaderListener` is used to create the root context
D) `DispatcherServlet` is a bean within the single context
E) Request and session scopes are available throughout the context

**Answer: B, D, E**
Spring Boot uses a SINGLE context — no hierarchy. `ContextLoaderListener` is not used. `DispatcherServlet` is registered as a bean (`DispatcherServletAutoConfiguration`). Since the single context IS a `WebApplicationContext`, all web scopes are registered.

---

**Q3 — Scenario**
A developer configures `RootConfig` with `@ComponentScan("com.example")` (scanning everything) and `WebConfig` also with `@ComponentScan("com.example")`. The app has `ProductService` annotated with `@Service @Transactional`. What is the likely runtime symptom?

A) `BeanDefinitionStoreException` — duplicate bean definitions
B) Transactions on `ProductService` silently fail when called from the controller
C) `NoSuchBeanDefinitionException` for `ProductService`
D) Application fails to start with circular dependency error

**Answer: B**
Both contexts create `ProductService`. The root context creates it with `@Transactional` proxy. The child context creates it as a plain bean (no transaction proxy — the `TransactionManager` may not be in child context). The controller autowires from child context's own registry (finds it locally — doesn't delegate to parent). Result: plain, non-transactional service. No startup error. Silent failure.

---

**Q4 — MCQ**
Where does `ContextLoaderListener` store the root `ApplicationContext` so that `DispatcherServlet` can find it?

A) In a static `ThreadLocal` variable
B) In a `ConcurrentHashMap` keyed by application name
C) As a `ServletContext` attribute under `WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`
D) As an `HttpSession` attribute

**Answer: C**
This is the exact mechanism. `ContextLoaderListener` calls `servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, rootContext)`. `DispatcherServlet` (via `WebApplicationContextUtils.getWebApplicationContext(servletContext)`) reads it from `ServletContext` and sets it as the parent of the child context.

---

**Q5 — Code Prediction**
```java
// Root context config
@Configuration
@ComponentScan(basePackages = "com.example",
    excludeFilters = @Filter(type = ANNOTATION, classes = Controller.class))
public class RootConfig { }

// Child context config  
@Configuration
@ComponentScan(basePackages = "com.example",
    useDefaultFilters = false,
    includeFilters = @Filter(type = ANNOTATION, classes = Controller.class))
public class WebConfig { }

// Service class
@Service
public class UserService { }

// Controller
@Controller
public class UserController {
    @Autowired UserService userService;
}
```
Where does `UserService` exist, and does the `@Autowired` succeed?

A) `UserService` in child only — autowire fails
B) `UserService` in root only — autowire succeeds via parent delegation
C) `UserService` in both — autowire uses child's instance
D) `UserService` in neither — both configs exclude it

**Answer: B**
`RootConfig` scans everything except `@Controller` → `UserService` is in root context. `WebConfig` scans ONLY `@Controller` → `UserService` NOT in child. `UserController` requests `UserService` → not in child → delegates to parent → found in root. Injection succeeds. This is the correctly configured two-context setup.

---

**Q6 — True/False**
A bean annotated with `@RequestScope` can be defined in the root `ApplicationContext` created by `ContextLoaderListener`.

**Answer: False**
The root context created by `ContextLoaderListener` is a `WebApplicationContext` (it implements the interface), but the `request` scope is registered during `WebApplicationContext.refresh()` — specifically by `WebApplicationContextUtils.registerWebApplicationScopes()`. This does register request scope in the root context too — but the scope requires an active HTTP request (via `RequestContextHolder`). In practice, request-scoped beans should be in the child context, and defining them in root context while ContextLoaderListener creates it causes scope unavailability errors in non-web tests. **Correction for precision:** Root WebApplicationContext DOES register web scopes, but accessing request-scoped beans outside of a request throws `IllegalStateException`. The practical trap is defining them in root context and testing outside a request context.

---

**Q7 — Ordering**
Order these context lifecycle events from first to last:

- Child context (DispatcherServlet) refreshed
- Root context refreshed
- Root context `ContextRefreshedEvent` fired
- `ContextLoaderListener.contextInitialized()` called
- Child context `ContextRefreshedEvent` fired
- `DispatcherServlet.init()` called

**Correct Order:**
1. `ContextLoaderListener.contextInitialized()` called
2. Root context refreshed
3. Root context `ContextRefreshedEvent` fired
4. `DispatcherServlet.init()` called
5. Child context (DispatcherServlet) refreshed
6. Child context `ContextRefreshedEvent` fired

---

**Q8 — Tricky MCQ**
A `@EventListener` for `ContextRefreshedEvent` is placed on a bean in the root context. How many times does it fire in a standard two-context Spring MVC setup?

A) Once — only when root context refreshes
B) Twice — once for root, once for child context
C) Once — only when child context refreshes
D) Zero — root context beans don't receive child context events

**Answer: A**
This is the nuanced answer. The listener is in the ROOT context. `ContextRefreshedEvent` propagates UP the context hierarchy — child events propagate to parent listeners. So a listener in the ROOT context receives BOTH root AND child `ContextRefreshedEvent`. But a listener in the CHILD context only receives CHILD events (no downward propagation).

Wait — this means the answer is **B**: the root context listener fires twice — once for root refresh, once for child refresh event propagating up. This is precisely the trap that requires the `event.getApplicationContext().getParent() == null` check.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — "Spring Boot uses two contexts"**
Spring Boot uses ONE context. The two-context hierarchy is a plain Spring MVC concept. Boot's single `AnnotationConfigServletWebServerApplicationContext` contains everything. Many candidates apply two-context rules to Boot — causing confusion.

**Trap 2 — The silent transaction failure**
Duplicate component scanning is the #1 real-world Spring MVC bug. When services exist in both contexts, the controller autowires the child's non-proxied instance. `@Transactional` is silently ignored. No exception. No warning. Data integrity bugs ensue. The fix is precise component scan filters.

**Trap 3 — ContextRefreshedEvent fires twice**
With two contexts, `ContextRefreshedEvent` fires once per context. Listeners in the root context receive both events (child events propagate up). Candidates assume it fires once. Code that runs "on startup" via this event runs twice. Database initialization, cache warming, scheduled task registration — all execute twice.

**Trap 4 — "Parent context cannot be WebApplicationContext"**
Both root AND child contexts are `WebApplicationContext` implementations in a standard Spring MVC setup. `ContextLoaderListener` creates a `XmlWebApplicationContext` or `AnnotationConfigWebApplicationContext` — both implement `WebApplicationContext`. The distinction is child vs parent role, not type.

**Trap 5 — Child bean overrides parent bean of same name**
If a bean named `productService` exists in BOTH root and child context, and the controller requests `productService` — the child's bean is found first (local registry checked before parent). No error. The root's transactional proxy is bypassed. This reinforces why duplicate scanning must be prevented.

**Trap 6 — `context.getBeanDefinitionNames()` scope**
`applicationContext.getBeanDefinitionNames()` returns ONLY beans in THAT context — not parent beans. Many developers inspect this and wonder why their service beans are "missing" when they're actually in the parent context. Use `BeanFactoryUtils.beanNamesForTypeIncludingAncestors()` to search the full hierarchy.

---

## 5️⃣ SUMMARY SHEET

```
TWO-CONTEXT HIERARCHY (Plain Spring MVC)
─────────────────────────────────────────────────────
ROOT ApplicationContext
  Created by: ContextLoaderListener
  Config:     contextConfigLocation <context-param>
  Contains:   @Service, @Repository, @Component, infrastructure
  Stored in:  ServletContext attribute (ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE)
  
CHILD WebApplicationContext
  Created by: DispatcherServlet.init()
  Config:     contextConfigLocation <init-param>
  Contains:   @Controller, ViewResolvers, HandlerMappings
  Parent:     ROOT context
  Stored in:  ServletContext attribute (servlet-name based key)

VISIBILITY RULES
─────────────────────────────────────────────────────
Child → Parent: YES (child delegates to parent when bean not found locally)
Parent → Child: NO  (parent has no child reference)
Child → Sibling: NO (sibling child contexts are invisible to each other)

BEAN LOOKUP ALGORITHM (Child Context)
─────────────────────────────────────────────────────
getBean("X") on child:
  1. Check child's own registry → found? Return.
  2. Delegate to parent registry → found? Return.
  3. NoSuchBeanDefinitionException

DUPLICATE SCAN BUG
─────────────────────────────────────────────────────
WRONG: Both configs scan com.example.*
  → Services in BOTH contexts
  → Controller gets child's non-transactional service
  → Transactions silently fail

CORRECT:
  Root: @ComponentScan + excludeFilters(Controller.class)
  Web:  @ComponentScan + useDefaultFilters=false + includeFilters(Controller.class)

SPRING BOOT DIFFERENCE
─────────────────────────────────────────────────────
Plain MVC:    TWO contexts (Root + Child WebApplicationContext)
Spring Boot:  ONE context (AnnotationConfigServletWebServerApplicationContext)
Boot:         ContextLoaderListener NOT used
Boot:         DispatcherServlet IS a bean in the single context
Boot:         No duplicate scan problem — single context cannot duplicate itself

CONTEXT REFRESHED EVENT TRAP
─────────────────────────────────────────────────────
Two contexts → ContextRefreshedEvent fires TWICE
Root listener receives: root event + child event (propagates up)
Fix: if (event.getApplicationContext().getParent() == null) { ... }

KEY INTERFACES
─────────────────────────────────────────────────────
ApplicationContext        → Core IoC container
WebApplicationContext     → ApplicationContext + ServletContext + web scopes
  └── Adds: request, session, application scope registration
  └── Adds: getServletContext()

getBeanDefinitionNames()   → Only THIS context's beans (not parent)
BeanFactoryUtils           → Methods to search full hierarchy

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "Child context sees parent beans — parent cannot see child beans"
• "ContextLoaderListener creates root context, DispatcherServlet creates child"
• "Duplicate component scanning causes silent transaction failures"
• "Spring Boot uses ONE context — no hierarchy, no duplicate scan problem"
• "ContextRefreshedEvent fires twice in two-context setup"
• "Root context stored in ServletContext attribute — how DispatcherServlet finds it"
• "getBeanDefinitionNames() does NOT include parent context beans"
• "Both contexts implement WebApplicationContext — distinction is parent/child role"
```

---
