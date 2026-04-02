# TOPIC 2.2 — WEBAPPLICATIONCONTEXT BOOTSTRAPPING

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What Bootstrapping Means

Bootstrapping is the **complete process of bringing a WebApplicationContext from zero to fully operational** — from the moment a configuration source is identified to the moment all singleton beans are instantiated, all post-processors have run, all AOP proxies are created, and the context is publishing events. It is not just "reading config" — it is the full lifecycle of the container itself coming alive.

Understanding bootstrapping at this depth answers critical questions: Why does `@Autowired` work? Why does `@Transactional` create a proxy? When exactly are beans created? What happens if a bean fails? Why does circular dependency cause an exception?

---

### The Central Class — AbstractApplicationContext.refresh()

Every `WebApplicationContext` implementation ultimately calls `AbstractApplicationContext.refresh()`. This single method **is** the bootstrap process. It is `synchronized` — only one thread can refresh a context at a time.

```java
// AbstractApplicationContext.refresh() — the complete bootstrap
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {

        StartupStep contextRefresh = 
            this.applicationStartup.start("spring.context.refresh");

        // Phase 1: Preparation
        prepareRefresh();

        // Phase 2: BeanFactory creation/refresh
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Phase 3: BeanFactory preparation
        prepareBeanFactory(beanFactory);

        try {
            // Phase 4: Post-processing (subclass hook)
            postProcessBeanFactory(beanFactory);

            StartupStep beanPostProcess = 
                this.applicationStartup.start("spring.context.beans.post-process");

            // Phase 5: BeanFactoryPostProcessors
            invokeBeanFactoryPostProcessors(beanFactory);

            // Phase 6: BeanPostProcessors registration
            registerBeanPostProcessors(beanFactory);

            beanPostProcess.end();

            // Phase 7: MessageSource (i18n)
            initMessageSource();

            // Phase 8: Event multicaster
            initApplicationEventMulticaster();

            // Phase 9: Subclass hook (creates embedded server in Boot)
            onRefresh();

            // Phase 10: Register listeners
            registerListeners();

            // Phase 11: INSTANTIATE ALL SINGLETONS
            finishBeanFactoryInitialization(beanFactory);

            // Phase 12: Finish
            finishRefresh();

        } catch (BeansException ex) {
            // Destroy already-created beans on error
            destroyBeans();
            // Reset 'active' flag
            cancelRefresh(ex);
            throw ex;
        } finally {
            resetCommonCaches();
            contextRefresh.end();
        }
    }
}
```

Each phase is critical. We now examine every phase precisely.

---

### Phase 1 — prepareRefresh()

```java
protected void prepareRefresh() {
    // Record start time
    this.startupDate = System.currentTimeMillis();

    // Set state flags
    this.closed.set(false);
    this.active.set(true);

    // Log startup
    if (logger.isDebugEnabled()) {
        logger.debug("Refreshing " + this);
    }

    // Hook: subclasses can initialise property sources
    // Web context: adds ServletContext and ServletConfig property sources
    initPropertySources();
    // AbstractRefreshableWebApplicationContext.initPropertySources():
    // WebApplicationContextUtils.initServletPropertySources(
    //     this.environment.getPropertySources(),
    //     this.servletContext,
    //     this.servletConfig
    // );
    // → Adds "servletContextInitParams" PropertySource
    // → Adds "servletConfigInitParams" PropertySource
    // → Now @Value("${contextConfigLocation}") works

    // Validate required properties
    getEnvironment().validateRequiredProperties();
    // Throws MissingRequiredPropertiesException if @Value required props missing

    // Initialise early listeners set
    if (this.earlyApplicationListeners == null) {
        this.earlyApplicationListeners = 
            new LinkedHashSet<>(this.applicationListeners);
    } else {
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
    }

    // Initialise early events set
    this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

**What `initPropertySources()` does for WebApplicationContext:**
- Adds `ServletContext` init-params as a `PropertySource` named `"servletContextInitParams"`
- Adds `ServletConfig` init-params as a `PropertySource` named `"servletConfigInitParams"`
- These are added at the CORRECT priority position in the `Environment`'s `MutablePropertySources`

This is why `@Value("${myContextParam}")` works when the value comes from a `<context-param>` in `web.xml`.

---

### Phase 2 — obtainFreshBeanFactory()

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    // For XML: parses XML, creates DefaultListableBeanFactory,
    //          registers all BeanDefinitions
    // For Annotation: creates DefaultListableBeanFactory,
    //                  registers @Configuration classes

    return getBeanFactory();
}
```

For `XmlWebApplicationContext`:
```
refreshBeanFactory()
  → Creates new DefaultListableBeanFactory
  → XmlBeanDefinitionReader reads XML files
  → Parses <bean>, <mvc:annotation-driven/>, <context:component-scan/>
  → All elements converted to BeanDefinition objects
  → BeanDefinitions registered in DefaultListableBeanFactory
  → Result: factory has bean definitions but NO instantiated beans
```

For `AnnotationConfigWebApplicationContext`:
```
refreshBeanFactory()
  → Creates new DefaultListableBeanFactory
  → Registers @Configuration class as BeanDefinition
  → Does NOT yet process @Bean methods or @ComponentScan
  → That happens in Phase 5 (BeanFactoryPostProcessors)
```

**Critical:** After Phase 2, there are **BeanDefinitions** — blueprints for beans. No actual bean instances exist yet. BeanDefinitions are metadata: class name, scope, constructor args, property values, init method, destroy method, lazy flag, etc.

---

### Phase 3 — prepareBeanFactory()

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {

    // Set classloader for bean class loading
    beanFactory.setBeanClassLoader(getClassLoader());

    // Set expression language resolver (#{...} in @Value)
    beanFactory.setBeanExpressionResolver(
        new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));

    // Add PropertyEditor support for type conversion
    beanFactory.addPropertyEditorRegistrar(
        new ResourceEditorRegistrar(this, getEnvironment()));

    // CRITICAL: Register ApplicationContextAwareProcessor
    // This BeanPostProcessor handles:
    // - ApplicationContextAware.setApplicationContext()
    // - EnvironmentAware.setEnvironment()
    // - BeanFactoryAware.setBeanFactory()
    // - MessageSourceAware.setMessageSource()
    // - ResourceLoaderAware.setResourceLoader()
    // - ApplicationEventPublisherAware.setApplicationEventPublisher()
    beanFactory.addBeanPostProcessor(
        new ApplicationContextAwareProcessor(this));

    // Ignore these interfaces for autowiring
    // (They are set by ApplicationContextAwareProcessor, not @Autowired)
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    // ... etc

    // Register resolvable dependencies
    // When @Autowired BeanFactory field encountered → inject THIS factory
    beanFactory.registerResolvableDependency(
        BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(
        ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(
        ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(
        ApplicationContext.class, this);
    // This is why @Autowired ApplicationContext works in any bean

    // Register LoadTimeWeaverAwareProcessor (for AspectJ LTW)
    if (!NativeDetector.inNativeImage() && 
        beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(
            new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(
            new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register environment beans
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    // Also registers: systemProperties, systemEnvironment, applicationStartup
}
```

After Phase 3:
- `@Autowired ApplicationContext` works
- `@Autowired Environment` works
- `ApplicationContextAware` beans will receive context reference
- PropertyEditors are registered for type conversion

---

### Phase 4 — postProcessBeanFactory() — Web-Specific

This is where `WebApplicationContext` adds web-specific infrastructure:

```java
// AbstractRefreshableWebApplicationContext.postProcessBeanFactory()
@Override
protected void postProcessBeanFactory(
        ConfigurableListableBeanFactory beanFactory) {

    // Register ServletContextAwareProcessor
    // Handles ServletContextAware and ServletConfigAware interfaces
    beanFactory.addBeanPostProcessor(
        new ServletContextAwareProcessor(
            this.servletContext, this.servletConfig));

    // These are set by processor, not autowired
    beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    beanFactory.ignoreDependencyInterface(ServletConfigAware.class);

    // Register web-specific scopes: request, session, application
    WebApplicationContextUtils.registerWebApplicationScopes(
        beanFactory, this.servletContext);
    // After this: @RequestScope, @SessionScope, @ApplicationScope work

    // Register ServletContext, ServletConfig, contextParameters,
    // contextAttributes as resolvable dependencies/beans
    WebApplicationContextUtils.registerEnvironmentBeans(
        beanFactory, this.servletContext, this.servletConfig);
    // After this: @Autowired ServletContext works in any bean
}
```

**`registerWebApplicationScopes()` registers:**
- `"request"` scope → `RequestScope` implementation
- `"session"` scope → `SessionScope` implementation
- `"application"` scope → `ServletContextScope` implementation

Each scope implementation knows how to:
1. Create a new bean instance for the scope
2. Store it in the appropriate carrier (request attributes, session, ServletContext)
3. Register a destruction callback (for `@RequestScope` cleanup)

---

### Phase 5 — invokeBeanFactoryPostProcessors() — The Most Complex Phase

This phase processes `BeanFactoryPostProcessor` implementations. The most important is `ConfigurationClassPostProcessor`:

```
invokeBeanFactoryPostProcessors(beanFactory)
        │
        ▼
PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()
        │
        ├── Find all BeanDefinitionRegistryPostProcessor beans
        │     (ConfigurationClassPostProcessor is one)
        │
        ├── ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()
        │     │
        │     ├── Finds all @Configuration classes
        │     ├── Processes @ComponentScan → scans packages
        │     │     → Finds @Controller, @Service, @Repository, @Component
        │     │     → Registers their BeanDefinitions
        │     │
        │     ├── Processes @Import → imports @Configuration classes
        │     ├── Processes @ImportResource → imports XML
        │     ├── Processes @Bean methods → registers method BeanDefinitions
        │     ├── Processes @PropertySource → loads .properties files
        │     └── Processes @Conditional → evaluates conditions
        │
        ├── After all BeanDefinitionRegistryPostProcessors:
        │     BeanFactory now has ALL BeanDefinitions (complete picture)
        │
        └── Invoke BeanFactoryPostProcessors (read-only phase):
              PropertySourcesPlaceholderConfigurer resolves ${...} placeholders
              → All @Value("${...}") values are now resolvable
```

**This is why the order matters:**
- `@ComponentScan` results in new `BeanDefinition` registrations
- `@PropertySource` loads properties BEFORE `${...}` placeholders are resolved
- `@Conditional` evaluates conditions against the BeanDefinition registry state at that moment
- `@Bean` methods on `@Configuration` classes are registered here

**After Phase 5:** The BeanFactory has a **complete and final** set of BeanDefinitions. No new BeanDefinitions will be added after this point.

---

### Phase 6 — registerBeanPostProcessors()

```java
protected void registerBeanPostProcessors(
        ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate
        .registerBeanPostProcessors(beanFactory, this);
}
```

This finds ALL `BeanPostProcessor` beans and registers them **in order**:

```
BeanPostProcessors registered (in this priority order):

1. PriorityOrdered BeanPostProcessors:
   - AutowiredAnnotationBeanPostProcessor
     → Processes @Autowired, @Value, @Inject
   - CommonAnnotationBeanPostProcessor
     → Processes @Resource, @PostConstruct, @PreDestroy

2. Ordered BeanPostProcessors:
   - AnnotationAwareAspectJAutoProxyCreator (if AOP enabled)
     → Creates CGLIB/JDK proxies for @Transactional, @Cacheable, @Async, AOP
   - PersistenceAnnotationBeanPostProcessor (if JPA)
     → Processes @PersistenceContext, @PersistenceUnit

3. Non-ordered BeanPostProcessors

4. Internal BeanPostProcessors (MergedBeanDefinitionPostProcessor)
```

**Critical:** `BeanPostProcessor` beans are instantiated during this phase — BEFORE regular beans. This has a consequence: any bean that `BeanPostProcessor` depends on gets instantiated early, potentially before AOP proxying is set up. This causes the "Bean not eligible for auto-proxying" warning.

---

### Phase 7 & 8 — MessageSource and Event Multicaster

```java
// Phase 7: i18n support
protected void initMessageSource() {
    if (beanFactory.containsLocalBean("messageSource")) {
        this.messageSource = beanFactory.getBean(
            "messageSource", MessageSource.class);
    } else {
        // Default: DelegatingMessageSource — empty, delegates to parent
        this.messageSource = new DelegatingMessageSource();
        beanFactory.registerSingleton("messageSource", this.messageSource);
    }
}

// Phase 8: Event publishing infrastructure
protected void initApplicationEventMulticaster() {
    if (beanFactory.containsLocalBean("applicationEventMulticaster")) {
        this.applicationEventMulticaster = beanFactory.getBean(
            "applicationEventMulticaster", ApplicationEventMulticaster.class);
    } else {
        // Default: SimpleApplicationEventMulticaster (synchronous)
        this.applicationEventMulticaster = 
            new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(
            "applicationEventMulticaster", 
            this.applicationEventMulticaster);
    }
}
```

**Default `SimpleApplicationEventMulticaster` is synchronous.** Events are published synchronously on the same thread. For async event handling, you must configure a `TaskExecutor` on it.

---

### Phase 9 — onRefresh() — Subclass Hook

For plain `WebApplicationContext`: no-op (or minimal).

For Spring Boot's `ServletWebServerApplicationContext`:
```java
@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        // Creates embedded Tomcat/Jetty/Undertow
        createWebServer();
        // → TomcatServletWebServerFactory.getWebServer()
        // → Creates Tomcat instance
        // → Configures connectors, context path, etc.
        // → Does NOT start Tomcat yet (started in finishRefresh())
    } catch (Throwable ex) {
        throw new ApplicationContextException(
            "Unable to start web server", ex);
    }
}
```

This is why in Spring Boot, the embedded server is **created** in `onRefresh()` but **started** in `finishRefresh()` — after all beans are ready.

---

### Phase 10 — registerListeners()

```java
protected void registerListeners() {
    // Register statically specified listeners first
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }

    // Find ApplicationListener beans (but don't instantiate them yet)
    String[] listenerBeanNames = getBeanNamesForType(
        ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        getApplicationEventMulticaster()
            .addApplicationListenerBean(listenerBeanName);
    }

    // Publish early events that were buffered before multicaster was ready
    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
    this.earlyApplicationEvents = null;
    if (!CollectionUtils.isEmpty(earlyEventsToProcess)) {
        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
            getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
    }
}
```

**Listeners are registered by NAME** — they aren't instantiated here. They'll be instantiated lazily on first event or during `finishBeanFactoryInitialization`.

---

### Phase 11 — finishBeanFactoryInitialization() — Bean Instantiation

**This is where all your beans actually come to life.** This is the longest phase at runtime.

```java
protected void finishBeanFactoryInitialization(
        ConfigurableListableBeanFactory beanFactory) {

    // Set ConversionService if defined
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
        beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, 
            ConversionService.class)) {
        beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, 
                ConversionService.class));
    }

    // Register EmbeddedValueResolver for @Value
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(
            strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // Initialize LoadTimeWeaverAware beans early (AspectJ LTW)
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(
        LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // Stop using temp classloader
    beanFactory.setTempClassLoader(null);

    // Freeze configuration — no more BeanDefinition changes
    beanFactory.freezeConfiguration();

    // INSTANTIATE ALL NON-LAZY SINGLETON BEANS
    beanFactory.preInstantiateSingletons();
}
```

**`preInstantiateSingletons()` — what happens for each singleton bean:**

```
For each BeanDefinition in registry:
  │
  ├── Is it a FactoryBean?
  │     → Get the FactoryBean itself (name: "&beanName")
  │     → If eager (not lazy), also get the product
  │
  └── Is it a regular bean?
        │
        ▼
  DefaultListableBeanFactory.getBean(beanName)
        │
        ▼
  AbstractBeanFactory.doGetBean()
        │
        ├── Check singleton cache (singletonObjects map) → already there? return it
        │
        ├── Check parent BeanFactory → found? return it
        │
        ├── Get BeanDefinition (merged with parent definition)
        │
        ├── Resolve dependencies (@DependsOn) → instantiate dependencies first
        │
        └── createBean(beanName, mbd, args)
              │
              ├── resolveBeforeInstantiation()
              │     → BeanPostProcessor.postProcessBeforeInstantiation()
              │     → If returns non-null: short-circuit (rarely used)
              │
              └── doCreateBean()
                    │
                    ├── instantiateBean() or instantiateUsingFactoryMethod()
                    │     → Choose constructor via ConstructorResolver
                    │     → Invoke constructor (possibly with @Autowired args)
                    │     → Raw bean instance created (no proxying yet)
                    │
                    ├── applyMergedBeanDefinitionPostProcessors()
                    │     → CommonAnnotationBeanPostProcessor discovers @PostConstruct
                    │     → AutowiredAnnotationBeanPostProcessor discovers @Autowired fields
                    │
                    ├── addSingletonFactory() ← CIRCULAR DEPENDENCY HANDLING
                    │     → Adds early reference to singleton cache
                    │     → Allows circular @Autowired to resolve
                    │
                    ├── populateBean()
                    │     → AutowiredAnnotationBeanPostProcessor.postProcessProperties()
                    │     → Resolves @Autowired fields → calls getBean() for each dependency
                    │     → Sets field values via reflection
                    │
                    └── initializeBean()
                          │
                          ├── invokeAwareMethods()
                          │     → BeanNameAware.setBeanName()
                          │     → BeanFactoryAware.setBeanFactory()
                          │     → ApplicationContextAware.setApplicationContext()
                          │
                          ├── applyBeanPostProcessorsBeforeInitialization()
                          │     → CommonAnnotationBeanPostProcessor:
                          │       calls @PostConstruct methods
                          │
                          ├── invokeInitMethods()
                          │     → InitializingBean.afterPropertiesSet()
                          │     → @Bean(initMethod="...")
                          │
                          └── applyBeanPostProcessorsAfterInitialization()
                                → AnnotationAwareAspectJAutoProxyCreator:
                                  creates CGLIB/JDK proxy if @Transactional/@Cacheable/AOP
                                → The PROXY replaces the raw bean in singleton cache
```

**This sequence for EVERY non-lazy singleton bean is the heart of Spring's IoC container.**

---

### Phase 12 — finishRefresh()

```java
protected void finishRefresh() {
    // Clear resource caches (loaded during refresh — no longer needed)
    clearResourceCaches();

    // Initialize LifecycleProcessor
    initLifecycleProcessor();

    // Start all Lifecycle beans (SmartLifecycle with autostart=true)
    getLifecycleProcessor().onRefresh();
    // → @Scheduled tasks start here (ScheduledAnnotationBeanPostProcessor)
    // → Spring Boot: Tomcat STARTS HERE (after all beans ready)

    // Publish ContextRefreshedEvent
    publishEvent(new ContextRefreshedEvent(this));
    // → @EventListener(ContextRefreshedEvent.class) fires here
    // → In two-context setup: fires once per context

    // Register with LiveBeansView MBean (if enabled)
    if (!NativeDetector.inNativeImage()) {
        LiveBeansView.registerApplicationContext(this);
    }
}
```

**`ContextRefreshedEvent` is published HERE** — the absolute last step of refresh. This is why `@EventListener(ContextRefreshedEvent.class)` is reliable for "run after all beans are ready" logic.

In Spring Boot: `TomcatWebServer.start()` is called during `getLifecycleProcessor().onRefresh()` — meaning Tomcat starts AFTER all beans are instantiated but BEFORE `ContextRefreshedEvent` is published.

---

### Circular Dependency Resolution — The Three-Level Cache

Spring handles circular dependencies in singleton beans via a three-level cache:

```
Level 1: singletonObjects        (ConcurrentHashMap) — fully initialised beans
Level 2: earlySingletonObjects   (HashMap)           — early exposed (proxied) beans  
Level 3: singletonFactories      (HashMap)           — ObjectFactory for early exposure
```

Resolution for circular `A → B → A`:

```
getBean("A")
  → A not in L1 (not yet created)
  → Start creating A
  → instantiateBean(A) → raw A instance
  → addSingletonFactory("A", () -> getEarlyBeanReference(A))  [ADD TO L3]
  → populateBean(A) → needs B → getBean("B")
      → B not in L1
      → Start creating B
      → instantiateBean(B) → raw B instance
      → addSingletonFactory("B", ...)  [ADD TO L3]
      → populateBean(B) → needs A → getBean("A")
          → A not in L1
          → A not in L2
          → A IS in L3 → call ObjectFactory → getEarlyBeanReference(A)
            (creates proxy if needed for AOP)
          → Put early A in L2, remove from L3
          → Return early A reference to B
      → B.fieldA = early A reference (OK — it's the same object reference)
      → initializeBean(B) → @PostConstruct
      → AOP proxy for B created (if needed)
      → B added to L1
  → A gets fully initialised B
  → initializeBean(A)
  → AOP proxy for A created
  → A added to L1, removed from L2
```

**What circular dependency CANNOT be resolved:**
- **Constructor injection cycles:** `A(B b)` and `B(A a)` — cannot create A without B, cannot create B without A. No early reference possible. Throws `BeanCurrentlyInCreationException`.
- **Prototype scope cycles:** Prototype beans don't use the singleton cache. Each `getBean()` creates a new instance — infinite recursion. Throws `BeanCurrentlyInCreationException`.

---

### Environment and PropertySource Hierarchy for WebApplicationContext

```
Environment PropertySource priority (highest to lowest):

1. ServletConfig init-params        (servletConfigInitParams)
2. ServletContext init-params       (servletContextInitParams)  
3. JNDI attributes                  (jndiProperties)
4. Java System properties           (systemProperties)
5. OS environment variables         (systemEnvironment)
6. @PropertySource files            (added by ConfigurationClassPostProcessor)
7. Default properties               (defaultProperties)

This means: ServletConfig init-params OVERRIDE system properties
            @PropertySource files are LOWER priority than system props
```

---

### WebApplicationContext-Specific BeanDefinitions

Beyond what a plain `ApplicationContext` registers, a `WebApplicationContext` automatically registers:

```
Beans auto-registered in WebApplicationContext:
─────────────────────────────────────────────────────────
"servletContext"      → ServletContext (the actual instance)
"servletConfig"       → ServletConfig (if available)
"contextParameters"   → Map<String, String> of ServletContext init-params
"contextAttributes"   → Map<String, Object> of ServletContext attributes

Resolvable dependencies (not beans but injectable):
BeanFactory           → the BeanFactory itself
ResourceLoader        → the ApplicationContext
ApplicationEventPublisher → the ApplicationContext
ApplicationContext    → the ApplicationContext
```

This is why `@Autowired ServletContext servletContext` works in any Spring-managed bean inside a web application.

---

## 2️⃣ CODE EXAMPLES

### Observing Bootstrap Phases via BeanFactoryPostProcessor

```java
// Runs during Phase 5 — after all BeanDefinitions registered
// but BEFORE any beans instantiated
@Component
public class BeanRegistryInspector implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(
            ConfigurableListableBeanFactory beanFactory) {

        System.out.println("=== BeanFactoryPostProcessor called ===");
        System.out.println("Total BeanDefinitions: " + 
            beanFactory.getBeanDefinitionCount());

        // All @ComponentScan, @Bean, @Import results are HERE
        String[] names = beanFactory.getBeanDefinitionNames();
        for (String name : names) {
            BeanDefinition bd = beanFactory.getBeanDefinition(name);
            System.out.println("  Bean: " + name + 
                " [scope=" + bd.getScope() + 
                ", lazy=" + bd.isLazyInit() + "]");
        }
        // No beans are instantiated yet at this point
    }
}
```

---

### Observing Bean Instantiation via BeanPostProcessor

```java
// Runs during Phase 11 — wraps EVERY bean instantiation
@Component
public class InstantiationObserver implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Called AFTER constructor + @Autowired, BEFORE @PostConstruct
        System.out.println("Before init: " + beanName + 
            " [" + bean.getClass().getSimpleName() + "]");
        return bean; // MUST return bean (or replacement)
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Called AFTER @PostConstruct + afterPropertiesSet()
        // AOP proxy creation happens HERE (in AnnotationAwareAspectJAutoProxyCreator)
        System.out.println("After init: " + beanName + 
            " [" + bean.getClass().getSimpleName() + "]");
        // If AOP applies: bean is now a PROXY here
        return bean; // Returning proxy here replaces raw bean
    }
}
```

---

### Understanding @Autowired Resolution During populateBean()

```java
@Service
public class OrderService {

    // Resolved during populateBean() → AutowiredAnnotationBeanPostProcessor
    // Calls getBean(ProductRepository.class) internally
    // If ProductRepository not yet created → creates it first
    @Autowired
    private ProductRepository productRepository;

    // Resolved same way — but List collects ALL implementations
    @Autowired
    private List<OrderValidator> validators;

    // Optional injection — no exception if missing
    @Autowired(required = false)
    private Optional<DiscountService> discountService;

    // @Value resolved by AutowiredAnnotationBeanPostProcessor
    // Placeholder resolved by EmbeddedValueResolver added in Phase 11
    @Value("${order.max-items:100}")
    private int maxItems;

    @PostConstruct
    public void init() {
        // Called AFTER all @Autowired fields are set
        // Called BEFORE AOP proxy creation
        // Safe to use productRepository here
        System.out.println("OrderService initialised, " +
            "validators: " + validators.size());
    }
}
```

---

### Circular Dependency — Constructor vs Field

```java
// FAILS: Constructor injection circular dependency
// BeanCurrentlyInCreationException at startup
@Service
public class ServiceA {
    private final ServiceB serviceB;
    @Autowired
    public ServiceA(ServiceB serviceB) { // Cannot create A without B
        this.serviceB = serviceB;
    }
}

@Service
public class ServiceB {
    private final ServiceA serviceA;
    @Autowired
    public ServiceB(ServiceA serviceA) { // Cannot create B without A
        this.serviceA = serviceA;
    }
}
// Error: The dependencies of some of the beans in the application 
// context form a cycle: serviceA -> serviceB -> serviceA

// WORKS (but discouraged): Field injection circular dependency
@Service
public class ServiceA {
    @Autowired ServiceB serviceB; // Field injection uses early reference
}

@Service
public class ServiceB {
    @Autowired ServiceA serviceA; // Gets early reference to A (L3 cache)
}
// Spring resolves via three-level cache — works but indicates design problem
```

---

### ContextRefreshedEvent — Correct Usage

```java
// Fires ONCE per context (twice in two-context setup)
@Component
public class ApplicationStartupRunner {

    @EventListener(ContextRefreshedEvent.class)
    public void onRefresh(ContextRefreshedEvent event) {
        // Root context has no parent
        if (event.getApplicationContext().getParent() == null) {
            performOneTimeInit();
        }
    }

    // Alternative: ApplicationRunner (Spring Boot only — fires once)
    // @Component class ApplicationStartup implements ApplicationRunner {
    //     public void run(ApplicationArguments args) { ... }
    // }
}
```

---

### Custom Scope Registration in WebApplicationContext

```java
// Custom scope that resets every hour
@Configuration
public class CustomScopeConfig {

    @Bean
    public static CustomScopeConfigurer customScopeConfigurer() {
        CustomScopeConfigurer configurer = new CustomScopeConfigurer();
        configurer.addScope("hourly", new HourlyScope());
        // Registered during Phase 5 (BeanFactoryPostProcessor)
        // Available for @Scope("hourly") before Phase 11 (bean instantiation)
        return configurer;
    }
}

@Component
@Scope("hourly")
public class HourlyCacheBean {
    // New instance created every hour — custom scope manages lifecycle
}
```

---

### Edge Case — BeanPostProcessor Dependency Trap

```java
// PROBLEM: BeanPostProcessor depends on regular bean
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {

    @Autowired
    private RegularService regularService; // Instantiated in Phase 6
    // regularService is created EARLY (in Phase 6)
    // Before AOP proxying is set up
    // If regularService has @Transactional → NO PROXY applied
    // Spring logs: "Bean 'regularService' of type [...] is not eligible
    //               for getting processed by all BeanPostProcessors"
}

// CORRECT: BeanPostProcessor should use BeanFactory lookup lazily
@Component
public class SafeBeanPostProcessor implements BeanPostProcessor, 
    BeanFactoryAware {

    private BeanFactory beanFactory;

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String name) {
        // Lazy lookup — only when actually processing
        RegularService service = beanFactory.getBean(RegularService.class);
        return bean;
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
At which phase of `AbstractApplicationContext.refresh()` are `@ComponentScan` packages actually scanned and `@Controller`, `@Service`, `@Repository` beans registered as `BeanDefinition`s?

A) Phase 2 — `obtainFreshBeanFactory()`
B) Phase 3 — `prepareBeanFactory()`
C) Phase 5 — `invokeBeanFactoryPostProcessors()`
D) Phase 11 — `finishBeanFactoryInitialization()`

**Answer: C**
`ConfigurationClassPostProcessor` (a `BeanDefinitionRegistryPostProcessor`) runs during Phase 5. It processes `@ComponentScan` directives, scans packages, and registers all discovered bean classes as `BeanDefinition`s. This is the complete answer: bean DEFINITIONS are registered in Phase 5, but bean INSTANCES are created in Phase 11.

---

**Q2 — Select All That Apply**
Which of the following are registered by `WebApplicationContextUtils.registerWebApplicationScopes()` during `postProcessBeanFactory()` (Phase 4)?

A) `"request"` scope
B) `"singleton"` scope
C) `"session"` scope
D) `"prototype"` scope
E) `"application"` scope

**Answer: A, C, E**
`singleton` and `prototype` are built-in BeanFactory scopes registered before the WebApplicationContext exists. `request`, `session`, and `application` are web-specific scopes added by `registerWebApplicationScopes()` during the web-specific Phase 4.

---

**Q3 — Ordering**
Order these events during `AbstractApplicationContext.refresh()`:

- `@PostConstruct` methods called on beans
- `ContextRefreshedEvent` published
- `@ComponentScan` packages scanned
- `BeanPostProcessor` beans instantiated and registered
- `ServletContext` property sources added to `Environment`
- AOP proxies created for `@Transactional` beans

**Correct Order:**
1. `ServletContext` property sources added to `Environment` (Phase 1 — prepareRefresh)
2. `@ComponentScan` packages scanned (Phase 5 — BeanFactoryPostProcessors)
3. `BeanPostProcessor` beans instantiated and registered (Phase 6)
4. `@PostConstruct` methods called on beans (Phase 11 — during initializeBean)
5. AOP proxies created for `@Transactional` beans (Phase 11 — after @PostConstruct)
6. `ContextRefreshedEvent` published (Phase 12 — finishRefresh)

---

**Q4 — MCQ**
Why does circular dependency resolution work with field `@Autowired` but NOT with constructor injection in singleton beans?

A) Field injection uses reflection to bypass Java's object construction rules
B) Constructor injection requires all dependencies before instantiation; field injection uses a three-level cache with early bean references
C) Spring detects constructor cycles at compile time and fails fast
D) Field injection creates multiple instances to break the cycle

**Answer: B**
With constructor injection, Spring needs the dependency to INSTANTIATE the bean — the dependency must exist before `new A(b)` can be called. With field injection, Spring can create a raw instance via no-arg constructor FIRST, register it as an early reference in the L3 singleton factory cache, then populate fields. Bean B gets the early reference to A (same object, just not fully initialised). This resolves the cycle.

---

**Q5 — Code Prediction**
```java
@Service
@Transactional
public class UserService {

    @Autowired
    private UserRepository repo;

    @PostConstruct
    public void init() {
        System.out.println("Class: " + 
            this.getClass().getSimpleName());
    }
}
```
When `init()` runs, what does `this.getClass().getSimpleName()` print?

A) `"UserService"`
B) `"UserService$$SpringCGLIB$$0"` or similar proxy class name
C) `"EnhancedUserService"`
D) `"TransactionalProxy"`

**Answer: A**
`@PostConstruct` is called during `initializeBean()` → `applyBeanPostProcessorsBeforeInitialization()`. AOP proxy creation happens in `applyBeanPostProcessorsAfterInitialization()` — AFTER `@PostConstruct`. So when `init()` runs, `this` is the raw `UserService` instance, not yet proxied. `this.getClass()` returns `UserService`. After `init()` returns, the proxy is created.

---

**Q6 — True/False**
A `BeanFactoryPostProcessor` can create new bean instances by calling `beanFactory.getBean()` during its execution.

**Answer: False — strongly discouraged and dangerous.**
`BeanFactoryPostProcessor` runs during Phase 5. At this point, `BeanPostProcessor` beans have NOT yet been registered (that's Phase 6). If a `BeanFactoryPostProcessor` calls `getBean()` to get a regular bean, that bean is instantiated EARLY — before `BeanPostProcessor`s are registered — so AOP proxies, `@Autowired` injection, `@PostConstruct` may not be applied correctly. The bean is "not eligible for auto-proxying." Spring logs a warning. Always keep `BeanFactoryPostProcessor` implementations limited to modifying `BeanDefinition`s only.

---

**Q7 — MCQ**
In Spring Boot, at which point does the embedded Tomcat server START (begin accepting connections)?

A) During `obtainFreshBeanFactory()` (Phase 2)
B) During `onRefresh()` (Phase 9)
C) During `finishBeanFactoryInitialization()` — when `TomcatWebServer` bean is created (Phase 11)
D) During `finishRefresh()` — when `SmartLifecycle` beans are started (Phase 12)

**Answer: D**
`onRefresh()` (Phase 9) CREATES the `TomcatWebServer` but does NOT start it. The server starts during `finishRefresh()` when `getLifecycleProcessor().onRefresh()` is called — this triggers `SmartLifecycle.start()` on `WebServerStartStopLifecycle`, which calls `TomcatWebServer.start()`. This happens AFTER all singleton beans are instantiated (Phase 11), ensuring all beans are ready before the first request can arrive.

---

**Q8 — Scenario**
A developer registers a `BeanPostProcessor` that `@Autowired`s a `DataSource` bean. At startup, Spring logs: `"Bean 'dataSource' of type [...] is not eligible for getting processed by all BeanPostProcessors"`. Why?

A) `DataSource` is not a Spring-managed bean
B) `DataSource` was instantiated early (in Phase 6) to satisfy the `BeanPostProcessor`'s `@Autowired`, before all `BeanPostProcessor`s were registered
C) `DataSource` has a circular dependency with the `BeanPostProcessor`
D) `BeanPostProcessor` beans cannot `@Autowired` other beans

**Answer: B**
During Phase 6, Spring instantiates all `BeanPostProcessor` beans. If a `BeanPostProcessor` has `@Autowired DataSource`, Spring must instantiate `DataSource` during Phase 6 to satisfy the dependency. But Phase 6 is REGISTERING `BeanPostProcessor`s — not all are registered yet. The early-created `DataSource` bean may miss post-processing by later-registered `BeanPostProcessor`s (like AOP proxy creator). Spring warns about this. The fix: use lazy `BeanFactory.getBean()` lookup inside the processor instead of `@Autowired`.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — "Beans are created when context is created"**
Beans are NOT created when the context object is instantiated (`new AnnotationConfigWebApplicationContext()`). They are created during `refresh()` — specifically Phase 11 (`finishBeanFactoryInitialization`). Creating the context object and refreshing it are two separate steps. In Spring Boot this distinction is invisible, but in programmatic context usage it's critical.

**Trap 2 — "@PostConstruct runs after AOP proxy is created"**
The opposite is true. `@PostConstruct` runs BEFORE AOP proxy creation. During `initializeBean()`, the sequence is: Aware interfaces → `@PostConstruct` → `afterPropertiesSet()` → `@Bean(initMethod)` → then AFTER all of those: `postProcessAfterInitialization()` creates the proxy. So inside `@PostConstruct`, `this` is the raw bean. Any `this.method()` call inside `@PostConstruct` bypasses the proxy.

**Trap 3 — "ContextRefreshedEvent means all beans are ready"**
Mostly true but with one nuance: `@Lazy` beans are NOT instantiated during Phase 11. They are created on first `getBean()` call after context refresh. `ContextRefreshedEvent` fires after Phase 11 — but lazy beans are not yet created. If your listener depends on a lazy bean existing, it may not yet.

**Trap 4 — "BeanPostProcessor can safely call getBean()"**
Calling `beanFactory.getBean()` from inside a `BeanFactoryPostProcessor` is extremely dangerous — it causes early instantiation of the requested bean before all `BeanPostProcessor`s are registered, potentially breaking AOP, transactions, and other infrastructure. `BeanFactoryPostProcessor` should ONLY modify `BeanDefinition` objects.

**Trap 5 — "Constructor injection is safer than field injection"**
For circular dependencies specifically, constructor injection is STRICTER — it fails fast with `BeanCurrentlyInCreationException`. Field injection silently resolves cycles using the three-level cache. Neither is inherently "safe" — the circular dependency itself is the design problem. Constructor injection just makes the problem visible at startup rather than silently working around it.

**Trap 6 — "SimpleApplicationEventMulticaster is async"**
The DEFAULT `SimpleApplicationEventMulticaster` is SYNCHRONOUS. `@EventListener` methods run on the same thread as the event publisher. If an event listener throws, the publisher sees the exception. For async events, configure a `TaskExecutor` on the multicaster or use `@Async` on the listener. Assuming events are async leads to thread-safety and exception-handling bugs.

**Trap 7 — "Phase order can be customised"**
The phases of `refresh()` are fixed and synchronised. You CANNOT reorder them. You can ADD behavior at specific hook points (`BeanFactoryPostProcessor`, `BeanPostProcessor`, `ApplicationListener`, `SmartLifecycle`) but the phases themselves execute in fixed order. Exam questions sometimes imply customisation of phase order — this is not possible.

---

## 5️⃣ SUMMARY SHEET

```
REFRESH() PHASE SEQUENCE
─────────────────────────────────────────────────────
Phase 1:  prepareRefresh()
            → Sets active flag, startupDate
            → initPropertySources() — adds ServletContext/Config params to Environment
            → Validates required properties

Phase 2:  obtainFreshBeanFactory()
            → Creates DefaultListableBeanFactory
            → Reads XML / registers @Configuration classes
            → Registers BeanDefinitions (NO bean instances yet)

Phase 3:  prepareBeanFactory()
            → ApplicationContextAwareProcessor registered
            → Resolvable deps: BeanFactory, ApplicationContext, etc.
            → Environment, systemProperties beans registered

Phase 4:  postProcessBeanFactory()  [Web-specific]
            → ServletContextAwareProcessor registered
            → Web scopes registered: request, session, application
            → ServletContext, ServletConfig registered as beans

Phase 5:  invokeBeanFactoryPostProcessors()
            → ConfigurationClassPostProcessor runs:
              @ComponentScan → discovers all @Controller/@Service/@Repository
              @Bean methods → registers BeanDefinitions
              @PropertySource → loads .properties files
              @Conditional → evaluates conditions
            → PropertySourcesPlaceholderConfigurer → resolves ${...}
            → FINAL state of BeanDefinition registry

Phase 6:  registerBeanPostProcessors()
            → AutowiredAnnotationBeanPostProcessor
            → CommonAnnotationBeanPostProcessor (@PostConstruct/@PreDestroy)
            → AnnotationAwareAspectJAutoProxyCreator (AOP)
            → BeanPostProcessor beans instantiated EARLY here

Phase 7:  initMessageSource()     → i18n support
Phase 8:  initApplicationEventMulticaster() → synchronous by default

Phase 9:  onRefresh()             → Spring Boot: creates embedded server (not started)

Phase 10: registerListeners()     → Registers ApplicationListener beans by name

Phase 11: finishBeanFactoryInitialization()
            → freezeConfiguration() — no more BeanDefinition changes
            → preInstantiateSingletons() — INSTANTIATES ALL SINGLETONS:
              For each bean:
              constructor/factory → populateBean (@Autowired) →
              Aware methods → @PostConstruct → afterPropertiesSet →
              initMethod → AOP proxy creation (postProcessAfterInit)

Phase 12: finishRefresh()
            → SmartLifecycle.start() — Spring Boot: Tomcat STARTS here
            → ContextRefreshedEvent published — LAST step

BEAN CREATION SEQUENCE (per bean, in Phase 11)
─────────────────────────────────────────────────────
1. Constructor invoked (or factory method)
2. addSingletonFactory() — early reference for circular dep resolution
3. populateBean() — @Autowired fields set via reflection
4. invokeAwareMethods() — BeanNameAware, BeanFactoryAware, ApplicationContextAware
5. @PostConstruct — CommonAnnotationBeanPostProcessor.postProcessBeforeInit()
6. afterPropertiesSet() — InitializingBean
7. @Bean(initMethod) — custom init method
8. AOP proxy created — postProcessAfterInitialization()
   (proxy replaces raw bean in singleton cache)

THREE-LEVEL SINGLETON CACHE (circular dependency)
─────────────────────────────────────────────────────
L1: singletonObjects      → fully initialised beans (final state)
L2: earlySingletonObjects → early-exposed proxied beans
L3: singletonFactories    → ObjectFactory for early reference

Constructor injection circular → FAILS (BeanCurrentlyInCreationException)
Field injection circular → WORKS via L3 cache (but indicates design problem)
Prototype circular → ALWAYS FAILS

WEB-SPECIFIC ADDITIONS vs PLAIN APPLICATIONCONTEXT
─────────────────────────────────────────────────────
Added scopes:   request, session, application
Added beans:    servletContext, servletConfig, contextParameters, contextAttributes
Added props:    servletContextInitParams, servletConfigInitParams PropertySources
Added processor: ServletContextAwareProcessor

ENVIRONMENT PROPERTY PRIORITY (highest to lowest)
─────────────────────────────────────────────────────
1. ServletConfig init-params
2. ServletContext init-params
3. JNDI attributes
4. Java System properties (-D flags)
5. OS environment variables
6. @PropertySource files
7. Default properties

KEY TIMING FACTS
─────────────────────────────────────────────────────
@PostConstruct        → Phase 11, BEFORE AOP proxy
AOP proxy creation    → Phase 11, AFTER @PostConstruct
ContextRefreshedEvent → Phase 12, AFTER ALL beans ready
Tomcat START (Boot)   → Phase 12, BEFORE ContextRefreshedEvent
@Lazy beans           → NOT created in Phase 11, created on first getBean()

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "@ComponentScan executes in Phase 5 (BeanFactoryPostProcessor) — not Phase 11"
• "@PostConstruct fires BEFORE AOP proxy creation — this is NOT the proxy"
• "ContextRefreshedEvent is the LAST thing Phase 12 does — all beans ready"
• "Spring Boot: Tomcat starts in Phase 12 — AFTER all beans instantiated"
• "BeanFactoryPostProcessor should never call getBean() — causes early instantiation"
• "Default event multicaster is SYNCHRONOUS — @EventListener blocks the publisher"
• "Constructor circular deps fail fast; field injection uses 3-level cache"
• "Phase 2 registers BeanDefinitions; Phase 11 creates bean instances"
• "BeanPostProcessor beans instantiated in Phase 6 — before regular beans"
• "refresh() is synchronized — only one thread bootstraps a context"
```

---
