# TOPIC 5.1 — SPRING MVC CONFIGURATION: XML, JAVA CONFIG, SPRING BOOT AUTO-CONFIGURATION

---

## 1️⃣ CONCEPTUAL EXPLANATION

### Why Configuration Deserves Deep Study

Configuration is where all Spring MVC components are assembled. Understanding configuration at depth means knowing: exactly which beans each configuration style creates, what the differences between styles produce at runtime, which auto-configuration classes are conditionally activated, what `@EnableWebMvc` does internally step by step, and how Spring Boot's auto-configuration relates to manual configuration. This topic ties together every component covered in Parts 1-4.

---

### Three Configuration Eras — What Each Produces

```
ERA 1: Pure XML (Spring 2.x)
  Files: web.xml + applicationContext.xml + dispatcher-servlet.xml
  Mechanism: XmlBeanDefinitionReader + Spring XML schema namespaces
  Produces: Same runtime beans as Java config, different declaration style

ERA 2: Annotation-driven XML hybrid (Spring 3.x)
  Files: web.xml + minimal XML with <mvc:annotation-driven/>
  Mechanism: MvcNamespaceHandler registers bean definitions
  Produces: Same runtime infrastructure beans

ERA 3: Pure Java config (Spring 3.1+)
  Files: WebApplicationInitializer + @Configuration classes
  Mechanism: @EnableWebMvc → DelegatingWebMvcConfiguration
  Produces: Same runtime beans, fully type-safe

SPRING BOOT AUTO-CONFIGURATION:
  Conditional activation: @ConditionalOnMissingBean, @ConditionalOnClass
  Key class: WebMvcAutoConfiguration
  Produces: Same beans + embedded server + opinionated defaults
  Override: implements WebMvcConfigurer (without @EnableWebMvc)
```

---

### @EnableWebMvc — Complete Internal Analysis

```java
// The annotation itself:
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(DelegatingWebMvcConfiguration.class)  // ← Everything happens here
public @interface EnableWebMvc {
}

// DelegatingWebMvcConfiguration:
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration
    extends WebMvcConfigurationSupport {

    // Collects ALL WebMvcConfigurer beans from ApplicationContext
    private final WebMvcConfigurerComposite configurers =
        new WebMvcConfigurerComposite();

    @Autowired(required = false)
    public void setConfigurers(List<WebMvcConfigurer> configurers) {
        if (!CollectionUtils.isEmpty(configurers)) {
            this.configurers.addWebMvcConfigurers(configurers);
        }
    }

    // Each override delegates to ALL WebMvcConfigurer beans:
    @Override
    protected void addInterceptors(InterceptorRegistry registry) {
        this.configurers.addInterceptors(registry);
    }

    @Override
    protected void addViewControllers(
            ViewControllerRegistry registry) {
        this.configurers.addViewControllers(registry);
    }

    // ... all other WebMvcConfigurer methods delegated similarly
}
```

---

### WebMvcConfigurationSupport — The Bean Factory

`WebMvcConfigurationSupport` is the **actual bean factory** for all Spring MVC infrastructure. It defines `@Bean` methods for every special bean:

```java
public class WebMvcConfigurationSupport
    implements ApplicationContextAware, ServletContextAware {

    // ── HANDLER MAPPINGS ──────────────────────────────────────────

    @Bean
    @SuppressWarnings("deprecation")
    public RequestMappingHandlerMapping requestMappingHandlerMapping(
            @Qualifier("mvcContentNegotiationManager")
                ContentNegotiationManager contentNegotiationManager,
            @Qualifier("mvcConversionService")
                FormattingConversionService conversionService,
            @Qualifier("mvcResourceUrlProvider")
                ResourceUrlProvider resourceUrlProvider) {

        RequestMappingHandlerMapping mapping =
            createRequestMappingHandlerMapping();
        mapping.setOrder(0);
        mapping.setInterceptors(getInterceptors(
            conversionService, resourceUrlProvider));
        mapping.setContentNegotiationManager(
            contentNegotiationManager);
        mapping.setCorsConfigurations(getCorsConfigurations());

        PathMatchConfigurer pathConfig =
            getPathMatchConfigurer();
        if (pathConfig.getPatternParser() != null) {
            mapping.setPatternParser(
                pathConfig.getPatternParser());
        }

        return mapping;
    }

    @Bean
    public HandlerMapping viewControllerHandlerMapping(
            @Qualifier("mvcConversionService")
                FormattingConversionService conversionService,
            @Qualifier("mvcResourceUrlProvider")
                ResourceUrlProvider resourceUrlProvider) {
        ViewControllerRegistry registry =
            new ViewControllerRegistry(
                this.applicationContext);
        addViewControllers(registry);
        // Calls WebMvcConfigurer.addViewControllers()

        AbstractHandlerMapping handlerMapping =
            registry.buildHandlerMapping();
        if (handlerMapping == null) {
            handlerMapping = new EmptyHandlerMapping();
        }
        // ...
        return handlerMapping;
    }

    @Bean
    @Nullable
    public HandlerMapping resourceHandlerMapping(
            @Qualifier("mvcContentNegotiationManager")
                ContentNegotiationManager contentNegotiationManager,
            @Qualifier("mvcConversionService")
                FormattingConversionService conversionService,
            @Qualifier("mvcResourceUrlProvider")
                ResourceUrlProvider resourceUrlProvider) {

        ResourceHandlerRegistry registry =
            new ResourceHandlerRegistry(
                this.applicationContext,
                this.servletContext,
                contentNegotiationManager,
                getUrlPathHelper());
        addResourceHandlers(registry);
        // Calls WebMvcConfigurer.addResourceHandlers()

        AbstractHandlerMapping handlerMapping =
            registry.getHandlerMapping();
        // null if no resource handlers configured
        return handlerMapping;
    }

    // ── HANDLER ADAPTERS ──────────────────────────────────────────

    @Bean
    public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
            @Qualifier("mvcContentNegotiationManager")
                ContentNegotiationManager contentNegotiationManager,
            @Qualifier("mvcConversionService")
                FormattingConversionService conversionService,
            @Qualifier("mvcValidator")
                Validator validator) {

        RequestMappingHandlerAdapter adapter =
            createRequestMappingHandlerAdapter();
        adapter.setContentNegotiationManager(
            contentNegotiationManager);
        adapter.setMessageConverters(
            getMessageConverters());
        adapter.setWebBindingInitializer(
            getConfigurableWebBindingInitializer(
                conversionService, validator));
        adapter.setCustomArgumentResolvers(
            getArgumentResolvers());
        adapter.setCustomReturnValueHandlers(
            getReturnValueHandlers());

        // @JsonView support
        adapter.setRequestBodyAdvice(
            Collections.singletonList(
                new JsonViewRequestBodyAdvice()));
        adapter.setResponseBodyAdvice(
            Collections.singletonList(
                new JsonViewResponseBodyAdvice()));

        // Async support
        AsyncSupportConfigurer asyncConfig =
            getAsyncSupportConfigurer();
        if (asyncConfig.getTaskExecutor() != null) {
            adapter.setTaskExecutor(
                asyncConfig.getTaskExecutor());
        }
        if (asyncConfig.getTimeout() != null) {
            adapter.setAsyncRequestTimeout(
                asyncConfig.getTimeout());
        }
        adapter.setCallableInterceptors(
            asyncConfig.getCallableInterceptors());
        adapter.setDeferredResultInterceptors(
            asyncConfig.getDeferredResultInterceptors());

        return adapter;
    }

    // ── MESSAGE CONVERTERS ────────────────────────────────────────

    protected final List<HttpMessageConverter<?>>
            getMessageConverters() {
        if (this.messageConverters == null) {
            this.messageConverters = new ArrayList<>();

            // Let WebMvcConfigurer REPLACE defaults:
            configureMessageConverters(
                this.messageConverters);
            // → Calls WebMvcConfigurer.configureMessageConverters()

            if (this.messageConverters.isEmpty()) {
                // No replacement → add defaults:
                addDefaultHttpMessageConverters(
                    this.messageConverters);
            }

            // Let WebMvcConfigurer EXTEND defaults:
            extendMessageConverters(
                this.messageConverters);
            // → Calls WebMvcConfigurer.extendMessageConverters()
        }
        return this.messageConverters;
    }

    protected final void addDefaultHttpMessageConverters(
            List<HttpMessageConverter<?>> messageConverters) {
        messageConverters.add(
            new ByteArrayHttpMessageConverter());
        messageConverters.add(
            new StringHttpMessageConverter());
        messageConverters.add(
            new ResourceHttpMessageConverter());
        messageConverters.add(
            new ResourceRegionHttpMessageConverter());
        // ... more defaults

        // Jackson if on classpath:
        if (jackson2Present) {
            messageConverters.add(
                new MappingJackson2HttpMessageConverter());
        }
        if (jackson2XmlPresent) {
            messageConverters.add(
                new MappingJackson2XmlHttpMessageConverter());
        }
        // JAXB if on classpath:
        if (jaxb2Present) {
            messageConverters.add(
                new Jaxb2RootElementHttpMessageConverter());
        }
    }

    // ── EXCEPTION RESOLVERS ───────────────────────────────────────

    @Bean
    public HandlerExceptionResolverComposite
            handlerExceptionResolver(
            @Qualifier("mvcContentNegotiationManager")
                ContentNegotiationManager contentNegotiationManager) {

        List<HandlerExceptionResolver> exceptionResolvers =
            new ArrayList<>();

        // Let WebMvcConfigurer REPLACE defaults:
        configureHandlerExceptionResolvers(exceptionResolvers);

        if (exceptionResolvers.isEmpty()) {
            // No replacement → add defaults:
            addDefaultHandlerExceptionResolvers(
                exceptionResolvers, contentNegotiationManager);
        }

        // Let WebMvcConfigurer EXTEND defaults:
        extendHandlerExceptionResolvers(exceptionResolvers);

        HandlerExceptionResolverComposite composite =
            new HandlerExceptionResolverComposite();
        composite.setOrder(0);
        composite.setExceptionResolvers(exceptionResolvers);
        return composite;
    }

    protected final void addDefaultHandlerExceptionResolvers(
            List<HandlerExceptionResolver> exceptionResolvers,
            ContentNegotiationManager contentNegotiationManager) {

        // @ExceptionHandler
        ExceptionHandlerExceptionResolver exceptionHandlerResolver =
            createExceptionHandlerExceptionResolver();
        exceptionHandlerResolver.setContentNegotiationManager(
            contentNegotiationManager);
        exceptionHandlerResolver.setMessageConverters(
            getMessageConverters());
        exceptionHandlerResolver.setCustomArgumentResolvers(
            getArgumentResolvers());
        exceptionHandlerResolver.setCustomReturnValueHandlers(
            getReturnValueHandlers());
        if (jackson2Present) {
            exceptionHandlerResolver.setResponseBodyAdvice(
                Collections.singletonList(
                    new JsonViewResponseBodyAdvice()));
        }
        exceptionHandlerResolver.afterPropertiesSet();
        exceptionResolvers.add(exceptionHandlerResolver);

        // @ResponseStatus
        ResponseStatusExceptionResolver responseStatusResolver =
            new ResponseStatusExceptionResolver();
        responseStatusResolver.setMessageSource(
            this.applicationContext);
        exceptionResolvers.add(responseStatusResolver);

        // Default Spring MVC exceptions
        exceptionResolvers.add(
            new DefaultHandlerExceptionResolver());
    }

    // ── VIEW RESOLVERS ─────────────────────────────────────────────

    @Bean
    public ViewResolver mvcViewResolver(
            @Qualifier("mvcContentNegotiationManager")
                ContentNegotiationManager contentNegotiationManager) {
        ViewResolverRegistry registry =
            new ViewResolverRegistry(
                contentNegotiationManager,
                this.applicationContext);
        configureViewResolvers(registry);
        // Calls WebMvcConfigurer.configureViewResolvers()

        if (registry.getViewResolvers().isEmpty() &&
                this.applicationContext != null) {
            String[] names = BeanFactoryUtils
                .beanNamesForTypeIncludingAncestors(
                    this.applicationContext,
                    ViewResolver.class, true, false);
            if (names.length == 1) {
                registry.getViewResolvers().add(
                    new InternalResourceViewResolver());
            }
        }
        // ...
        return composite;
    }

    // ── CONVERSION SERVICE ────────────────────────────────────────

    @Bean
    public FormattingConversionService mvcConversionService() {
        FormattingConversionService conversionService =
            new DefaultFormattingConversionService();
        addFormatters(conversionService);
        // Calls WebMvcConfigurer.addFormatters()
        return conversionService;
    }

    // ── VALIDATOR ─────────────────────────────────────────────────

    @Bean
    public Validator mvcValidator() {
        Validator validator = getValidator();
        // Calls WebMvcConfigurer.getValidator()
        if (validator == null) {
            if (ClassUtils.isPresent(
                    "javax.validation.executable.ExecutableValidator",
                    getClass().getClassLoader())) {
                Class<?> clazz = ClassUtils.forName(
                    "org.springframework.validation" +
                    ".beanvalidation.OptionalValidatorFactoryBean",
                    getClass().getClassLoader());
                validator = (Validator)
                    BeanUtils.instantiateClass(clazz);
            } else {
                validator = new NoOpValidator();
            }
        }
        return validator;
    }

    // ── CONTENT NEGOTIATION ───────────────────────────────────────

    @Bean
    public ContentNegotiationManager
            mvcContentNegotiationManager() {
        ContentNegotiationConfigurer configurer =
            new ContentNegotiationConfigurer(
                this.servletContext);
        configurer.mediaType(
            "json", MediaType.APPLICATION_JSON);
        configurer.mediaType(
            "xml", MediaType.APPLICATION_XML);
        configureContentNegotiation(configurer);
        // Calls WebMvcConfigurer.configureContentNegotiation()
        return configurer.buildContentNegotiationManager();
    }
}
```

---

### Spring Boot WebMvcAutoConfiguration — Conditional Assembly

```java
@AutoConfiguration(after = {
    DispatcherServletAutoConfiguration.class,
    TaskExecutionAutoConfiguration.class,
    ValidationAutoConfiguration.class
})
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({Servlet.class, DispatcherServlet.class,
    WebMvcConfigurer.class})
// Only activates when:
// 1. This is a Servlet-based web application
// 2. Servlet, DispatcherServlet, WebMvcConfigurer are on classpath
// 3. WebMvcConfigurationSupport is NOT present (see below)
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
// ← THE CRITICAL CONDITION
// @EnableWebMvc imports WebMvcConfigurationSupport via
// DelegatingWebMvcConfiguration → this condition FAILS
// → WebMvcAutoConfiguration BACKS OFF
// → This is why @EnableWebMvc breaks Spring Boot MVC auto-config
@Import(EnableWebMvcConfiguration.class)
public class WebMvcAutoConfiguration {

    // ── INTERNAL EnableWebMvcConfiguration ─────────────────────

    @Configuration(proxyBeanMethods = false)
    @EnableConfigurationProperties(WebMvcProperties.class)
    @Import({ EnableWebMvcConfiguration.class,
              WebMvcAutoConfigurationAdapter.class })
    public static class EnableWebMvcConfiguration
            extends DelegatingWebMvcConfiguration {

        // Overrides some WebMvcConfigurationSupport behaviors:

        @Bean
        @Override
        public RequestMappingHandlerAdapter
                requestMappingHandlerAdapter(...) {
            RequestMappingHandlerAdapter adapter =
                super.requestMappingHandlerAdapter(...);
            adapter.setIgnoreDefaultModelOnRedirect(
                this.mvcProperties == null ||
                this.mvcProperties
                    .isIgnoreDefaultModelOnRedirect());
            return adapter;
        }

        @Bean
        @Override
        public WelcomePageHandlerMapping
                welcomePageHandlerMapping(
                ApplicationContext applicationContext,
                FormattingConversionService mvcConversionService,
                ResourceUrlProvider mvcResourceUrlProvider) {
            return new WelcomePageHandlerMapping(
                new TemplateAvailabilityProviders(
                    applicationContext),
                applicationContext,
                getWelcomePage(),
                this.mvcProperties.getStaticPathPattern());
        }
    }

    // ── WebMvcAutoConfigurationAdapter ─────────────────────────
    // Implements WebMvcConfigurer with opinionated defaults

    @Configuration(proxyBeanMethods = false)
    @Import(EnableWebMvcConfiguration.class)
    @EnableConfigurationProperties({
        WebMvcProperties.class,
        WebProperties.class
    })
    @Order(0)
    public static class WebMvcAutoConfigurationAdapter
            implements WebMvcConfigurer {

        @Override
        public void configureMessageConverters(
                List<HttpMessageConverter<?>> converters) {
            this.messageConvertersProvider.ifAvailable(
                customConverters ->
                    converters.addAll(customConverters.getConverters()));
        }

        @Override
        public void configureAsyncSupport(
                AsyncSupportConfigurer configurer) {
            if (this.beanFactory.containsBean(
                    TaskExecutionAutoConfiguration
                        .APPLICATION_TASK_EXECUTOR_BEAN_NAME)) {
                // Auto-configure async task executor
                Object taskExecutor = this.beanFactory.getBean(
                    TaskExecutionAutoConfiguration
                        .APPLICATION_TASK_EXECUTOR_BEAN_NAME);
                if (taskExecutor instanceof AsyncTaskExecutor ate) {
                    configurer.setTaskExecutor(ate);
                }
            }
        }

        @Override
        public void addResourceHandlers(
                ResourceHandlerRegistry registry) {
            if (!this.resourceProperties.isAddMappings()) {
                logger.debug("Default resource handling disabled");
                return;
            }

            // /webjars/** → classpath:/META-INF/resources/webjars/
            addResourceHandler(registry, "/webjars/**",
                "classpath:/META-INF/resources/webjars/");

            // /** → configured static locations
            addResourceHandler(registry,
                this.mvcProperties.getStaticPathPattern(),
                this.resourceProperties.getStaticLocations());
        }

        @Override
        public void addViewControllers(
                ViewControllerRegistry registry) {
            if (this.resourceProperties.isAddMappings()) {
                // /favicon.ico served from static locations
            }
        }

        @Override
        public void configureContentNegotiation(
                ContentNegotiationConfigurer configurer) {
            WebMvcProperties.Contentnegotiation props =
                this.mvcProperties.getContentnegotiation();
            configurer.favorParameter(
                props.isFavorParameter());
            if (props.getParameterName() != null) {
                configurer.parameterName(
                    props.getParameterName());
            }
            // Media type mappings from properties
            Map<String, MediaType> mediaTypes =
                this.mvcProperties.getContentnegotiation()
                    .getMediaTypes();
            mediaTypes.forEach(configurer::mediaType);
        }
    }
}
```

---

### DispatcherServletAutoConfiguration

```java
@AutoConfiguration(
    after = ServletWebServerFactoryAutoConfiguration.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
public class DispatcherServletAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(DefaultDispatcherServletCondition.class)
    @ConditionalOnClass(ServletRegistration.class)
    @EnableConfigurationProperties(WebMvcProperties.class)
    protected static class DispatcherServletConfiguration {

        // Creates DispatcherServlet as a Spring bean
        @Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServlet dispatcherServlet(
                WebMvcProperties webMvcProperties) {
            DispatcherServlet dispatcherServlet =
                new DispatcherServlet();
            dispatcherServlet.setDispatchOptionsRequest(
                webMvcProperties.isDispatchOptionsRequest());
            dispatcherServlet.setDispatchTraceRequest(
                webMvcProperties.isDispatchTraceRequest());
            dispatcherServlet.setThrowExceptionIfNoHandlerFound(
                webMvcProperties
                    .isThrowExceptionIfNoHandlerFound());
            dispatcherServlet.setPublishEvents(
                webMvcProperties.isPublishRequestHandledEvents());
            dispatcherServlet.setEnableLoggingRequestDetails(
                webMvcProperties.isLogRequestDetails());
            return dispatcherServlet;
        }
    }

    @Configuration(proxyBeanMethods = false)
    @Conditional(DispatcherServletRegistrationCondition.class)
    @ConditionalOnClass(ServletRegistration.class)
    @EnableConfigurationProperties(WebMvcProperties.class)
    @Import(DispatcherServletConfiguration.class)
    protected static class DispatcherServletRegistrationConfiguration {

        // Registers DispatcherServlet with embedded Tomcat
        @Bean(name =
                DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
        @ConditionalOnBean(value = DispatcherServlet.class,
                           name =
                DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServletRegistrationBean
                dispatcherServletRegistration(
                DispatcherServlet dispatcherServlet,
                WebMvcProperties webMvcProperties,
                ObjectProvider<MultipartConfigElement>
                    multipartConfig) {

            DispatcherServletRegistrationBean registration =
                new DispatcherServletRegistrationBean(
                    dispatcherServlet,
                    webMvcProperties.getServlet().getPath());
            // Default path: "/"
            registration.setName(
                DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
            registration.setLoadOnStartup(
                webMvcProperties.getServlet().getLoadOnStartup());
            multipartConfig.ifAvailable(
                registration::setMultipartConfig);
            return registration;
        }
    }
}
```

---

### Complete Configuration Comparison

**Identical runtime result — three styles:**

```java
// STYLE 1: XML Configuration
// dispatcher-servlet.xml:
<mvc:annotation-driven/>
<mvc:resources mapping="/static/**"
               location="classpath:/static/"/>
<mvc:interceptors>
    <bean class="com.example.LoggingInterceptor"/>
</mvc:interceptors>
<bean class="org.springframework.web.servlet.view
             .InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/views/"/>
    <property name="suffix" value=".jsp"/>
</bean>

// STYLE 2: Java Config
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(
            ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/");
    }

    @Override
    public void addInterceptors(InterceptorRegistry reg) {
        reg.addInterceptor(new LoggingInterceptor());
    }

    @Bean
    public InternalResourceViewResolver viewResolver() {
        InternalResourceViewResolver r =
            new InternalResourceViewResolver();
        r.setPrefix("/WEB-INF/views/");
        r.setSuffix(".jsp");
        return r;
    }
}

// STYLE 3: Spring Boot
// application.properties:
// spring.mvc.view.prefix=/WEB-INF/views/
// spring.mvc.view.suffix=.jsp
// spring.web.resources.static-locations=classpath:/static/

// Plus:
@Component
public class LoggingInterceptor implements HandlerInterceptor {}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired LoggingInterceptor loggingInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry reg) {
        reg.addInterceptor(loggingInterceptor);
    }
    // NO @EnableWebMvc — Spring Boot handles the rest
}
```

---

### WebMvcProperties — application.properties Mappings

```java
// Complete WebMvcProperties mapping:
@ConfigurationProperties(prefix = "spring.mvc")
public class WebMvcProperties {

    // DispatcherServlet settings:
    // spring.mvc.dispatch-options-request=true
    // spring.mvc.dispatch-trace-request=false
    // spring.mvc.throw-exception-if-no-handler-found=false
    // spring.mvc.publish-request-handled-events=true
    // spring.mvc.log-request-details=false

    // Static resources:
    // spring.mvc.static-path-pattern=/**
    // spring.web.resources.static-locations=classpath:/META-INF/resources/,...
    // spring.web.resources.add-mappings=true
    // spring.web.resources.cache.period=3600s
    // spring.web.resources.chain.strategy.content.enabled=true

    // View resolution:
    // spring.mvc.view.prefix=/WEB-INF/views/
    // spring.mvc.view.suffix=.jsp

    // Content negotiation:
    // spring.mvc.contentnegotiation.favor-parameter=false
    // spring.mvc.contentnegotiation.parameter-name=format
    // spring.mvc.contentnegotiation.media-types.json=application/json

    // Async:
    // spring.mvc.async.request-timeout=30000

    // Path matching:
    // spring.mvc.pathmatch.matching-strategy=path-pattern-parser
    // (Spring 6: ANT_PATH_MATCHER deprecated)

    // Locale:
    // spring.web.locale=en
    // spring.web.locale-resolver=accept-header
}
```

---

### The @EnableWebMvc in Spring Boot Problem — Full Explanation

```java
// WHY @EnableWebMvc BREAKS SPRING BOOT:

// @EnableWebMvc imports DelegatingWebMvcConfiguration
// DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport

// WebMvcAutoConfiguration has:
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
// When DelegatingWebMvcConfiguration (a WebMvcConfigurationSupport
// subclass) is in context → this condition FAILS
// → WebMvcAutoConfiguration.WebMvcAutoConfigurationAdapter
//   → DOES NOT run
// → All Boot auto-configuration LOST:
//   ✗ Jackson auto-configuration lost
//   ✗ Static resource handler lost
//   ✗ Content negotiation defaults lost
//   ✗ InternalResourceViewResolver from properties lost
//   ✗ LocaleResolver from properties lost
//   ✗ Thymeleaf view resolver lost (uses WebMvcAutoConfiguration)
//   ✗ Welcome page mapping lost

// SYMPTOMS:
// 1. @GetMapping("/products") returns 404 (no resource handler for /)
// 2. REST endpoints return 406 (no Jackson converter)
// 3. CSS/JS not served (/static/** not registered)
// 4. JSP configuration from properties ignored

// FIX: Remove @EnableWebMvc, use WebMvcConfigurer only
@Configuration
// NO @EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    // Customisations here extend Boot's auto-config
    // @EnableWebMvc would REPLACE Boot's auto-config
}
```

---

### Customisation Hierarchy in Spring Boot

```
Layer 1: WebMvcAutoConfiguration defaults
  → Static resources, Jackson, ViewResolvers, etc.
  → Controlled by application.properties

Layer 2: WebMvcConfigurer implementations
  → Extend/customise Layer 1 without replacing it
  → Multiple @Configuration classes can implement WebMvcConfigurer
  → All are discovered and called

Layer 3: @Bean overrides
  → Replace specific beans by name or type
  → @Bean RequestMappingHandlerMapping → replaces default mapping
  → Must be careful not to duplicate beans

Layer 4: @EnableWebMvc (REPLACES Layer 1)
  → Turns off all auto-configuration
  → Now responsible for everything
  → Rarely needed in Spring Boot apps
```

---

### Multiple WebMvcConfigurer Implementations

```java
// Multiple @Configuration classes can implement WebMvcConfigurer
// Spring collects ALL of them and calls all methods

@Configuration
public class SecurityWebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new SecurityInterceptor())
                .addPathPatterns("/api/**");
    }
}

@Configuration
public class CachingWebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new CacheInterceptor())
                .addPathPatterns("/**");
    }
}

@Configuration
public class CorsWebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://app.example.com");
    }
}

// DelegatingWebMvcConfiguration collects all three:
// @Autowired void setConfigurers(List<WebMvcConfigurer> configurers)
// → SecurityWebConfig, CachingWebConfig, CorsWebConfig all collected
// addInterceptors() → calls all three's addInterceptors()
// → Both SecurityInterceptor and CacheInterceptor registered
```

---

### Pure XML Configuration — Complete Example

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
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>

<!-- dispatcher-servlet.xml -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:context="http://www.springframework.org/schema/context">

    <!-- Registers RequestMappingHandlerMapping,
         RequestMappingHandlerAdapter, ExceptionResolvers,
         MessageConverters, ConversionService, Validator -->
    <mvc:annotation-driven/>

    <!-- Component scan for @Controller -->
    <context:component-scan
        base-package="com.example.web"
        use-default-filters="false">
        <context:include-filter type="annotation"
            expression="...Controller"/>
    </context:component-scan>

    <!-- View resolver -->
    <bean class="...InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/views/"/>
        <property name="suffix" value=".jsp"/>
    </bean>

    <!-- Static resources -->
    <mvc:resources mapping="/static/**"
                   location="classpath:/static/"
                   cache-period="3600"/>

    <!-- Interceptors -->
    <mvc:interceptors>
        <mvc:interceptor>
            <mvc:mapping path="/**"/>
            <bean class="com.example.LoggingInterceptor"/>
        </mvc:interceptor>
    </mvc:interceptors>

    <!-- Content negotiation -->
    <mvc:annotation-driven content-negotiation-manager=
                               "contentNegotiationManager"/>
    <bean id="contentNegotiationManager"
          class="...ContentNegotiationManagerFactoryBean">
        <property name="defaultContentType"
                  value="application/json"/>
    </bean>

    <!-- CORS -->
    <mvc:cors>
        <mvc:mapping path="/api/**"
                     allowed-origins="https://app.example.com"/>
    </mvc:cors>
</beans>
```

---

## 2️⃣ CODE EXAMPLES

### Production Spring Boot Configuration

```java
// Complete production-grade Spring Boot MVC configuration
@Configuration
@Slf4j
public class MvcConfiguration implements WebMvcConfigurer {

    @Autowired private Environment env;
    @Autowired private JwtAuthInterceptor jwtAuth;
    @Autowired private RequestLoggingInterceptor logging;
    @Autowired private RateLimitInterceptor rateLimit;

    // ── INTERCEPTORS ───────────────────────────────────────────────
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(logging)
                .addPathPatterns("/**")
                .order(1);

        registry.addInterceptor(rateLimit)
                .addPathPatterns("/api/**")
                .order(2);

        registry.addInterceptor(jwtAuth)
                .addPathPatterns("/api/**")
                .excludePathPatterns(
                    "/api/auth/**",
                    "/api/public/**")
                .order(3);
    }

    // ── CORS ───────────────────────────────────────────────────────
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        String[] allowedOrigins =
            env.getProperty("cors.allowed-origins",
                String[].class,
                new String[]{"http://localhost:3000"});

        registry.addMapping("/api/**")
                .allowedOrigins(allowedOrigins)
                .allowedMethods("GET", "POST", "PUT",
                                "DELETE", "PATCH", "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600L);
    }

    // ── CONTENT NEGOTIATION ────────────────────────────────────────
    @Override
    public void configureContentNegotiation(
            ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(
                env.getProperty(
                    "spring.mvc.contentnegotiation.favor-parameter",
                    Boolean.class, false))
            .defaultContentType(
                MediaType.APPLICATION_JSON)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML);
    }

    // ── MESSAGE CONVERTERS ─────────────────────────────────────────
    @Override
    public void extendMessageConverters(
            List<HttpMessageConverter<?>> converters) {
        // Fix StringHttpMessageConverter charset
        converters.removeIf(c ->
            c instanceof StringHttpMessageConverter);
        converters.add(0, new StringHttpMessageConverter(
            StandardCharsets.UTF_8));

        // Configure Jackson
        converters.stream()
            .filter(c -> c instanceof
                MappingJackson2HttpMessageConverter)
            .map(c -> (MappingJackson2HttpMessageConverter) c)
            .forEach(c -> {
                ObjectMapper mapper = c.getObjectMapper();
                mapper.registerModule(new JavaTimeModule());
                mapper.disable(SerializationFeature
                    .WRITE_DATES_AS_TIMESTAMPS);
                mapper.setSerializationInclusion(
                    JsonInclude.Include.NON_NULL);
                mapper.configure(
                    DeserializationFeature
                        .FAIL_ON_UNKNOWN_PROPERTIES, false);
            });
    }

    // ── STATIC RESOURCES ──────────────────────────────────────────
    @Override
    public void addResourceHandlers(
            ResourceHandlerRegistry registry) {
        boolean isProd = env.acceptsProfiles(
            Profiles.of("production"));

        registry.addResourceHandler("/assets/**")
                .addResourceLocations(
                    "classpath:/static/assets/")
                .setCacheControl(
                    isProd ?
                    CacheControl.maxAge(365, TimeUnit.DAYS)
                        .immutable() :
                    CacheControl.noCache())
                .resourceChain(isProd)
                .addResolver(
                    new VersionResourceResolver()
                    .addContentVersionStrategy("/**"));
    }

    // ── ASYNC SUPPORT ──────────────────────────────────────────────
    @Override
    public void configureAsyncSupport(
            AsyncSupportConfigurer configurer) {
        configurer.setDefaultTimeout(30_000L);
        configurer.setTaskExecutor(asyncTaskExecutor());
    }

    @Bean
    public ThreadPoolTaskExecutor asyncTaskExecutor() {
        ThreadPoolTaskExecutor executor =
            new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("mvc-async-");
        executor.initialize();
        return executor;
    }

    // ── PATH MATCHING ──────────────────────────────────────────────
    @Override
    public void configurePathMatch(
            PathMatchConfigurer configurer) {
        configurer.setUseTrailingSlashMatch(false);
        // Spring 6: disabled by default anyway
    }
}
```

---

### Replacing Specific Infrastructure Beans

```java
// Override specific Spring MVC infrastructure beans
@Configuration
public class CustomInfrastructureConfig {

    // Replace RequestMappingHandlerMapping
    @Bean
    @Primary
    public RequestMappingHandlerMapping
            requestMappingHandlerMapping(
            ContentNegotiationManager manager) {
        CustomRequestMappingHandlerMapping mapping =
            new CustomRequestMappingHandlerMapping();
        mapping.setOrder(0);
        mapping.setContentNegotiationManager(manager);
        return mapping;
    }

    // Add custom HandlerAdapter
    @Bean
    public HandlerAdapter customHandlerAdapter() {
        return new ScriptedControllerHandlerAdapter();
        // DispatcherServlet's detectAllHandlerAdapters=true
        // will discover this automatically
    }

    // Override ExceptionHandlerExceptionResolver
    @Bean
    @Order(0)
    public HandlerExceptionResolver
            customExceptionResolver() {
        ExceptionHandlerExceptionResolver resolver =
            new ExceptionHandlerExceptionResolver();
        resolver.setMessageConverters(
            messageConverters());
        resolver.afterPropertiesSet();
        return resolver;
    }
}
```

---

### Testing Configuration with @WebMvcTest

```java
// Lightweight test — only MVC layer
@WebMvcTest(controllers = ProductController.class)
// Scans only: @Controller, @ControllerAdvice,
//             @JsonComponent, Converter, Filter, WebMvcConfigurer
// Does NOT scan: @Service, @Repository, @Component
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ProductService productService;
    // Required because @Service not scanned

    @Test
    void shouldReturnProductList() throws Exception {
        given(productService.findAll())
            .willReturn(List.of(
                new Product(1L, "Widget", new BigDecimal("9.99"))));

        mockMvc.perform(get("/api/products")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(content()
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$[0].name")
                .value("Widget"))
            .andExpect(jsonPath("$[0].price")
                .value(9.99));
    }

    @Test
    void shouldReturn404ForMissingProduct() throws Exception {
        given(productService.findById(99L))
            .willReturn(Optional.empty());

        mockMvc.perform(get("/api/products/99")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isNotFound());
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
`@EnableWebMvc` is added to a `@Configuration` class in a Spring Boot application. Which of the following is the MOST accurate description of what happens?

A) `@EnableWebMvc` enhances Spring Boot's auto-configuration with additional features
B) `@EnableWebMvc` disables `WebMvcAutoConfiguration` because it registers `WebMvcConfigurationSupport`, and `WebMvcAutoConfiguration` has `@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)`
C) `@EnableWebMvc` is silently ignored in Spring Boot — Boot's configuration takes precedence
D) `@EnableWebMvc` requires re-registering all converters manually but otherwise works fine

**Answer: B**
`@EnableWebMvc` imports `DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport`. `WebMvcAutoConfiguration` checks `@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)` — condition FAILS since `WebMvcConfigurationSupport` IS present. `WebMvcAutoConfiguration` backs off entirely. Jackson converter, static resource handling, view resolvers from properties — all lost.

---

**Q2 — Select All That Apply**
`WebMvcConfigurationSupport.getMessageConverters()` calls both `configureMessageConverters()` AND `extendMessageConverters()`. Which behaviours are TRUE?

A) If `configureMessageConverters()` adds converters, defaults are NOT added
B) If `configureMessageConverters()` is empty, defaults ARE added, then `extendMessageConverters()` called
C) `extendMessageConverters()` is always called regardless of `configureMessageConverters()` result
D) Adding to `configureMessageConverters()` is equivalent to adding to `extendMessageConverters()`
E) `configureMessageConverters()` and `extendMessageConverters()` are called from different threads

**Answer: A, B, C**
A: `configureMessageConverters()` adding to the list → non-empty → defaults skipped. B: empty → `addDefaultHttpMessageConverters()` called → then `extendMessageConverters()`. C: `extendMessageConverters()` ALWAYS called (even if configureMessageConverters populated the list). D is wrong — configure may replace defaults, extend always adds to whatever is there. E is wrong — same thread, sequential.

---

**Q3 — MCQ**
Multiple `@Configuration` classes implement `WebMvcConfigurer` in a Spring Boot application. All implement `addInterceptors()`. What is the result?

A) Only the last registered `WebMvcConfigurer` is used — others are ignored
B) `IllegalStateException` — only one `WebMvcConfigurer` allowed
C) `DelegatingWebMvcConfiguration` collects all and calls all `addInterceptors()` — all interceptors are registered
D) Spring Boot only uses the first `WebMvcConfigurer` found

**Answer: C**
`DelegatingWebMvcConfiguration.setConfigurers(List<WebMvcConfigurer>)` — `@Autowired` — collects ALL `WebMvcConfigurer` beans. `WebMvcConfigurerComposite.addInterceptors()` iterates all and calls each one's `addInterceptors()`. All interceptors from all configurers are registered. This is the core pattern enabling modular configuration.

---

**Q4 — Select All That Apply**
Which beans are registered by `@EnableWebMvc` / `WebMvcConfigurationSupport` that are NOT registered without it (plain Spring beans context)?

A) `RequestMappingHandlerMapping`
B) `RequestMappingHandlerAdapter`
C) `InternalResourceViewResolver`
D) `ExceptionHandlerExceptionResolver`
E) `ContentNegotiationManager`
F) `FormattingConversionService`

**Answer: A, B, D, E, F**
C (`InternalResourceViewResolver`) is NOT registered by `@EnableWebMvc`. `WebMvcConfigurationSupport.mvcViewResolver()` only adds `InternalResourceViewResolver` if NO other `ViewResolver` is configured AND exactly one `ViewResolver` bean exists in context. It is NOT a default registration. A, B, D, E, F are all explicitly registered by `WebMvcConfigurationSupport`.

---

**Q5 — True/False**
In Spring Boot, `spring.mvc.throw-exception-if-no-handler-found=true` in `application.properties` is equivalent to calling `registration.setInitParameter("throwExceptionIfNoHandlerFound", "true")` in a plain Spring MVC `WebApplicationInitializer`.

**Answer: True**
Both set `DispatcherServlet.throwExceptionIfNoHandlerFound = true`. In Spring Boot: `DispatcherServletAutoConfiguration` creates `DispatcherServlet` and reads `WebMvcProperties` (backed by `application.properties`). In plain Spring MVC: the init-param is read by `HttpServletBean.init()` via `BeanWrapper`. Both ultimately call `dispatcherServlet.setThrowExceptionIfNoHandlerFound(true)`. The runtime behaviour is identical.

---

**Q6 — MCQ**
`<mvc:annotation-driven/>` in XML config vs `@EnableWebMvc` in Java config. Which is TRUE?

A) They produce slightly different infrastructure — `@EnableWebMvc` is more complete
B) They produce equivalent infrastructure — both delegate to `WebMvcConfigurationSupport` equivalent
C) `<mvc:annotation-driven/>` doesn't support `@ControllerAdvice` — requires `@EnableWebMvc`
D) `@EnableWebMvc` supports `@JsonView`; `<mvc:annotation-driven/>` does not

**Answer: B**
`<mvc:annotation-driven/>` is handled by `MvcNamespaceHandler` which calls `AnnotationDrivenBeanDefinitionParser`. This parser registers THE SAME beans as `WebMvcConfigurationSupport`: `RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter`, all three `HandlerExceptionResolver`s, `ContentNegotiationManager`, etc. They produce equivalent runtime infrastructure. A, C, D are all wrong.

---

**Q7 — Code Prediction**
```java
@SpringBootApplication
public class App {}

@Configuration
@EnableWebMvc  // ← This annotation
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(
            ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/");
    }
}
```
`GET /static/css/app.css` — what happens?

A) File served from classpath:/static/ with auto-configured caching
B) File served but WITHOUT Spring Boot's auto-configured caching (manual config only)
C) 404 — `@EnableWebMvc` disabled Boot's static resource auto-config AND WebConfig's addResourceHandlers IS registered
D) 404 — `@EnableWebMvc` disabled Boot's auto-config AND WebConfig's addResourceHandlers is NOT registered

**Answer: C (but with nuance)**
`@EnableWebMvc` disables `WebMvcAutoConfiguration`. Boot's default `/static/**` mapping is gone. BUT `WebConfig.addResourceHandlers()` manually registers `/static/**`. So the file IS found. However: NO `Cache-Control` headers set (WebConfig doesn't configure them), NO `VersionResourceResolver` (not configured), NO Boot auto-configuration for this handler. The file is served, but without any caching configuration. Technically: file served, but without Boot's opinionated defaults.

---

**Q8 — Select All That Apply**
Which Spring Boot application.properties settings directly configure `DispatcherServlet` properties?

A) `spring.mvc.dispatch-options-request=true`
B) `spring.mvc.throw-exception-if-no-handler-found=true`
C) `spring.mvc.view.prefix=/WEB-INF/views/`
D) `spring.mvc.publish-request-handled-events=false`
E) `spring.web.resources.static-locations=classpath:/static/`

**Answer: A, B, D**
A: `dispatchOptionsRequest` — DispatcherServlet property. B: `throwExceptionIfNoHandlerFound` — DispatcherServlet property. D: `publishEvents` → `publishRequestHandledEvents` — DispatcherServlet property. C is a `WebMvcAutoConfigurationAdapter` setting that configures `InternalResourceViewResolver`, not DispatcherServlet directly. E configures `ResourceHandlerRegistry`, not DispatcherServlet.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `@EnableWebMvc` in Spring Boot is almost always wrong**
This is the single most common Spring Boot MVC misconfiguration. Developers add `@EnableWebMvc` thinking "I need to enable Spring MVC." Spring Boot already enables it. Adding `@EnableWebMvc` silently disables Jackson, static resources, view resolver auto-configuration, and more. The correct approach: implement `WebMvcConfigurer` without `@EnableWebMvc`.

**Trap 2 — `configureMessageConverters()` vs `extendMessageConverters()` in Boot**
In Spring Boot with `WebMvcAutoConfiguration`, `extendMessageConverters()` is the safe choice. `configureMessageConverters()` replaces Boot's default converters which include auto-configured Jackson. `extendMessageConverters()` adds to them. Even in pure `@EnableWebMvc` config: `configureMessageConverters()` adding ANY converter prevents defaults from being added at all.

**Trap 3 — Multiple `WebMvcConfigurer` beans are ALL applied**
Developers sometimes define multiple `@Configuration` classes implementing `WebMvcConfigurer`, unaware that ALL are collected by `DelegatingWebMvcConfiguration`. If two configurers both call `configureContentNegotiation()`, both configurations are merged. This can cause unexpected behaviour when one configurer's settings conflict with another's. All `WebMvcConfigurer` implementations in the context affect the configuration.

**Trap 4 — `InternalResourceViewResolver` is NOT registered by `@EnableWebMvc`**
A common exam trick: `@EnableWebMvc` registers `RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter`, exception resolvers, converter service, validator, content negotiation manager — but NOT `InternalResourceViewResolver`. JSP support requires explicitly defining a `ViewResolver` bean or calling `registry.jsp()` in `configureViewResolvers()`.

**Trap 5 — `WebMvcConfigurationSupport` presence check is by type, not by name**
`@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)` checks if ANY bean of type `WebMvcConfigurationSupport` (or subclass) exists. `DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport`. So even a custom `@Configuration` class extending `WebMvcConfigurationSupport` directly disables `WebMvcAutoConfiguration` — not just `@EnableWebMvc`.

**Trap 6 — Ordering of multiple `WebMvcConfigurer` implementations**
`DelegatingWebMvcConfiguration` collects `WebMvcConfigurer` beans via `@Autowired List<WebMvcConfigurer>`. The order of the list depends on bean registration order and `@Order` annotations. For interceptors, order within each configurer's `addInterceptors()` call matters, but the order BETWEEN configurers is determined by their `@Order` values. Without `@Order`, order is undefined between configurers.

**Trap 7 — `@WebMvcTest` doesn't load `@Service` beans**
`@WebMvcTest` is a slice test — it only loads MVC-relevant beans. `@Service`, `@Repository`, `@Component` are NOT loaded. All dependencies of `@Controller` must be provided via `@MockBean`. This is by design — unit testing the web layer in isolation. Forgetting `@MockBean` for a service dependency → `UnsatisfiedDependencyException` in test.

---

## 5️⃣ SUMMARY SHEET

```
THREE CONFIGURATION STYLES — EQUIVALENT RESULTS
─────────────────────────────────────────────────────
XML:         <mvc:annotation-driven/> + XML beans
Java Config: @EnableWebMvc + @Configuration
Spring Boot: WebMvcAutoConfiguration + WebMvcConfigurer

All produce: RequestMappingHandlerMapping, RequestMappingHandlerAdapter,
             ExceptionHandlerExceptionResolver, ResponseStatusExceptionResolver,
             DefaultHandlerExceptionResolver, ContentNegotiationManager,
             FormattingConversionService, Validator, MessageConverters

@ENABLEWEBMVC INTERNALS
─────────────────────────────────────────────────────
@EnableWebMvc → @Import(DelegatingWebMvcConfiguration.class)
DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport
  → Collects ALL WebMvcConfigurer beans via @Autowired List
  → Delegates all hook methods to all WebMvcConfigurer implementations

WebMvcConfigurationSupport: the @Bean factory for all MVC infrastructure
  → requestMappingHandlerMapping()
  → requestMappingHandlerAdapter()
  → handlerExceptionResolver()
  → mvcViewResolver()
  → mvcConversionService()
  → mvcValidator()
  → mvcContentNegotiationManager()
  → resourceHandlerMapping()
  NOTE: NOT InternalResourceViewResolver (must add explicitly)

SPRING BOOT AUTO-CONFIGURATION KEY CLASSES
─────────────────────────────────────────────────────
DispatcherServletAutoConfiguration:
  → Creates DispatcherServlet @Bean
  → Creates DispatcherServletRegistrationBean
  → Maps to "/" by default

WebMvcAutoConfiguration:
  @ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
  → BACKS OFF if @EnableWebMvc present in context
  → Provides: Jackson, static resources, view resolvers, locale, etc.
  → WebMvcAutoConfigurationAdapter implements WebMvcConfigurer

@ENABLEWEBMVC IN SPRING BOOT — CRITICAL
─────────────────────────────────────────────────────
@EnableWebMvc → DelegatingWebMvcConfiguration → WebMvcConfigurationSupport
WebMvcAutoConfiguration @ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
→ CONDITION FAILS → AUTO-CONFIG BACKS OFF
→ LOST: Jackson, static resources, view resolver from properties,
        content negotiation defaults, welcome page, locale resolver

FIX: implements WebMvcConfigurer WITHOUT @EnableWebMvc

WEBMVCCONFIGURER CUSTOMISATION
─────────────────────────────────────────────────────
Multiple @Configuration classes can implement WebMvcConfigurer
ALL are collected by DelegatingWebMvcConfiguration
ALL are called for each hook method
Order between configurers: @Order annotation

configureMessageConverters(): REPLACES defaults if non-empty
extendMessageConverters():    ADDS TO existing list (always safe)
getValidator():               REPLACES default validator
configureHandlerExceptionResolvers(): REPLACES defaults
extendHandlerExceptionResolvers():    ADDS TO defaults

BEANS NOT REGISTERED BY @ENABLEWEBMVC
─────────────────────────────────────────────────────
InternalResourceViewResolver — add explicitly via @Bean or configureViewResolvers()
LocaleResolver — add explicitly
MultipartResolver — add explicitly (bean name MUST be "multipartResolver")
MessageSource — add explicitly (bean name MUST be "messageSource")

SPRING BOOT application.properties MVC SETTINGS
─────────────────────────────────────────────────────
spring.mvc.dispatch-options-request=true
spring.mvc.throw-exception-if-no-handler-found=false
spring.mvc.view.prefix=/WEB-INF/views/
spring.mvc.view.suffix=.jsp
spring.mvc.static-path-pattern=/**
spring.web.resources.static-locations=classpath:/static/,...
spring.web.resources.add-mappings=true
spring.mvc.contentnegotiation.favor-parameter=false
spring.mvc.async.request-timeout=30000

@WEBMVCTEST SLICE TEST
─────────────────────────────────────────────────────
Loads: @Controller, @ControllerAdvice, @JsonComponent, WebMvcConfigurer
Does NOT load: @Service, @Repository, @Component
Use @MockBean for: all @Controller dependencies
Use @Import for: custom configuration classes needed in test

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "@EnableWebMvc in Boot = WebMvcAutoConfiguration disabled = Jackson/static lost"
• "WebMvcConfigurationSupport is the @Bean factory — @EnableWebMvc just imports it via delegation"
• "All WebMvcConfigurer implementations collected and called — order by @Order"
• "configureMessageConverters non-empty → no defaults; empty → defaults added"
• "@WebMvcTest: only MVC beans — @MockBean all service dependencies"
• "InternalResourceViewResolver NOT registered by @EnableWebMvc — add explicitly"
• "Multiple WebMvcConfigurer beans: ALL methods called — avoid conflicting configurations"
• "spring.mvc.throw-exception-if-no-handler-found=true enables @ExceptionHandler for 404"
• "<mvc:annotation-driven/> and @EnableWebMvc produce equivalent infrastructure"
• "DelegatingWebMvcConfiguration: delegates to all WebMvcConfigurer via composite pattern"
```

---
