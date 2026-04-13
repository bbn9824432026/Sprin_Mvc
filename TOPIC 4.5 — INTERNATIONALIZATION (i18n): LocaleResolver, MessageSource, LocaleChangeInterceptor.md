# TOPIC 4.5 — INTERNATIONALIZATION (i18n): LocaleResolver, MessageSource, LocaleChangeInterceptor

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What i18n Really Is in Spring MVC

Internationalization in Spring MVC is not simply "loading translated strings." It is a complete system involving: locale detection and storage, message resolution with fallback chains, parameterized message interpolation, validation message localization, and view-layer integration. The system has precise ordering rules, interaction patterns between components, and subtle behaviours that distinguish mastery from surface knowledge.

---

### The i18n Architecture — Four Interconnected Components

```
HTTP Request
     │
     ▼
┌────────────────────────────────────────────────────┐
│  LocaleResolver (detects/stores current Locale)    │
│  AcceptHeaderLocaleResolver (from Accept-Language) │
│  SessionLocaleResolver (from HttpSession)          │
│  CookieLocaleResolver (from Cookie)               │
│  FixedLocaleResolver (always same Locale)          │
└────────────────────┬───────────────────────────────┘
                     │ Sets Locale in
                     ▼
┌────────────────────────────────────────────────────┐
│  LocaleContextHolder (ThreadLocal)                 │
│  Available throughout request processing chain     │
│  LocaleContextHolder.getLocale() → current Locale  │
└────────────────────┬───────────────────────────────┘
                     │ Used by
                     ▼
┌────────────────────────────────────────────────────┐
│  MessageSource (resolves message codes to text)    │
│  ResourceBundleMessageSource                       │
│  ReloadableResourceBundleMessageSource             │
│  StaticMessageSource (for testing)                 │
└────────────────────┬───────────────────────────────┘
                     │ Used by
                     ▼
┌────────────────────────────────────────────────────┐
│  View Layer (templates use MessageSource)          │
│  JSP: <spring:message code="..."/>                 │
│  Thymeleaf: #{message.key}                         │
│  FreeMarker: ${spring.getMessage("key")}           │
│  Controller: MessageSource.getMessage(code, args,  │
│              locale)                               │
└────────────────────────────────────────────────────┘
     ▲
     │ Changed by
┌────────────────────────────────────────────────────┐
│  LocaleChangeInterceptor                           │
│  Reads ?lang=fr from request                       │
│  Calls localeResolver.setLocale(request, response, │
│         Locale.FRENCH)                             │
└────────────────────────────────────────────────────┘
```

---

### LocaleResolver Interface — Complete Analysis

```java
public interface LocaleResolver {

    // Resolve the current locale from the request
    // Called by FrameworkServlet.processRequest()
    // Sets LocaleContextHolder
    Locale resolveLocale(HttpServletRequest request);

    // Change the current locale
    // Called by LocaleChangeInterceptor
    // Some implementations throw UnsupportedOperationException
    void setLocale(HttpServletRequest request,
                   @Nullable HttpServletResponse response,
                   @Nullable Locale locale);
}

// LocaleContextResolver (extends LocaleResolver)
// Also provides TimeZone information
public interface LocaleContextResolver extends LocaleResolver {

    // Returns LocaleContext (Locale + optional TimeZone)
    LocaleContext resolveLocaleContext(
        HttpServletRequest request);

    // Sets both Locale AND TimeZone
    void setLocaleContext(HttpServletRequest request,
                          @Nullable HttpServletResponse response,
                          @Nullable LocaleContext localeContext);
}
```

---

### AcceptHeaderLocaleResolver — HTTP Standard Approach

```java
public class AcceptHeaderLocaleResolver
    implements LocaleContextResolver {

    // Configured supported locales — if none set, accept anything
    @Nullable
    private List<Locale> supportedLocales;

    // Default locale when no match found
    @Nullable
    private Locale defaultLocale;

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        Locale defaultLocale = getDefaultLocale();

        // No Accept-Language header → use default
        if (defaultLocale != null &&
                request.getHeader("Accept-Language") == null) {
            return defaultLocale;
        }

        // Get Locale from Accept-Language header
        Locale requestLocale = request.getLocale();
        // request.getLocale() parses Accept-Language internally

        List<Locale> supportedLocales = getSupportedLocales();

        if (supportedLocales.isEmpty() ||
                supportedLocales.contains(requestLocale)) {
            return requestLocale;
        }

        // Find best match from supported locales
        Locale languageMatch = null;
        for (Locale locale : request.getLocales()) {
            if (supportedLocales.contains(locale)) {
                // Exact match (language + country)
                if (languageMatch == null ||
                        languageMatch.getLanguage().equals(
                            locale.getLanguage())) {
                    return locale;
                }
            }
            if (languageMatch == null) {
                // Language-only match (e.g., "fr" matches "fr-FR")
                for (Locale supportedLocale : supportedLocales) {
                    if (!StringUtils.hasLength(
                            supportedLocale.getCountry()) &&
                            supportedLocale.getLanguage().equals(
                                locale.getLanguage())) {
                        languageMatch = supportedLocale;
                        break;
                    }
                }
            }
        }

        return (languageMatch != null ? languageMatch :
                (defaultLocale != null ? defaultLocale :
                    requestLocale));
    }

    @Override
    public void setLocale(HttpServletRequest request,
                           @Nullable HttpServletResponse response,
                           @Nullable Locale locale) {
        throw new UnsupportedOperationException(
            "Cannot change HTTP Accept-Language header - " +
            "use a different locale resolution strategy");
        // Cannot change what client sends in header
        // Use SessionLocaleResolver or CookieLocaleResolver
        // if locale change is needed
    }
}
```

**Accept-Language header parsing:**
```
Accept-Language: fr-FR, fr;q=0.9, en-US;q=0.8, en;q=0.7

Parsed as:
  fr-FR  (quality=1.0)  → French (France)
  fr     (quality=0.9)  → French (any country)
  en-US  (quality=0.8)  → English (United States)
  en     (quality=0.7)  → English (any country)

Matching against supportedLocales=[fr, en, de]:
  fr-FR → exact match? No. Language match for fr? YES → match "fr"
  Returns: Locale("fr")
```

---

### SessionLocaleResolver — Session-Based Persistence

```java
public class SessionLocaleResolver
    implements LocaleContextResolver {

    // Session attribute name for storing locale
    public static final String LOCALE_SESSION_ATTRIBUTE_NAME =
        SessionLocaleResolver.class.getName() + ".LOCALE";

    // Session attribute name for storing time zone
    public static final String TIME_ZONE_SESSION_ATTRIBUTE_NAME =
        SessionLocaleResolver.class.getName() + ".TIME_ZONE";

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        Locale locale = (Locale) WebUtils.getSessionAttribute(
            request,
            LOCALE_SESSION_ATTRIBUTE_NAME);
        // = request.getSession().getAttribute(LOCALE_SESSION_ATTRIBUTE_NAME)

        if (locale == null) {
            locale = determineDefaultLocale(request);
            // Falls back to defaultLocale if set
            // OR request.getLocale() from Accept-Language
        }
        return locale;
    }

    @Override
    public void setLocale(HttpServletRequest request,
                           @Nullable HttpServletResponse response,
                           @Nullable Locale locale) {
        // Store locale in session
        HttpSession session =
            request.getSession();

        if (locale == null) {
            session.removeAttribute(
                LOCALE_SESSION_ATTRIBUTE_NAME);
        } else {
            session.setAttribute(
                LOCALE_SESSION_ATTRIBUTE_NAME, locale);
        }
    }
}
```

**Session locale lifecycle:**
```
First request:
  → No session attribute
  → Falls back to defaultLocale or Accept-Language

Locale change (via LocaleChangeInterceptor, ?lang=fr):
  → localeResolver.setLocale(req, res, Locale.FRENCH)
  → session.setAttribute(LOCALE_SESSION_ATTR, Locale.FRENCH)

Subsequent requests (same session):
  → session.getAttribute(LOCALE_SESSION_ATTR) → Locale.FRENCH
  → All messages rendered in French
  → Persists until session expires or explicit reset

Session expiry:
  → Locale lost
  → Falls back to default
```

---

### CookieLocaleResolver — Cross-Session Persistence

```java
public class CookieLocaleResolver
    extends CookieGenerator implements LocaleContextResolver {

    public static final String DEFAULT_COOKIE_NAME =
        CookieLocaleResolver.class.getName() + ".LOCALE";

    // Cookie configuration
    // Inherited from CookieGenerator:
    private String cookieName = DEFAULT_COOKIE_NAME;
    private int cookieMaxAge = -1;  // Session cookie by default
    // Set to seconds for persistent cookie:
    // cookieMaxAge = 365 * 24 * 60 * 60 (1 year)

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        // Check cookie
        String cookieValue = null;
        Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                if (getCookieName().equals(cookie.getName())) {
                    cookieValue = cookie.getValue();
                    break;
                }
            }
        }

        if (cookieValue != null) {
            return parseLocaleValue(cookieValue);
            // "fr_FR" or "fr-FR" → Locale("fr", "FR")
        }

        return determineDefaultLocale(request);
    }

    @Override
    public void setLocale(HttpServletRequest request,
                           @Nullable HttpServletResponse response,
                           @Nullable Locale locale) {
        if (locale != null) {
            // Add cookie to response
            addCookie(response,
                toLocaleValue(locale));
            // "fr_FR" cookie value
        } else {
            // Remove cookie
            removeCookie(response);
        }
    }
}
```

**Cookie locale vs Session locale:**
```
SessionLocaleResolver:
  Persists: Until session expires
  Scope:    Single browser session (tab/window set)
  Storage:  Server-side (HttpSession)

CookieLocaleResolver:
  Persists: Until cookie expires (configurable: session or years)
  Scope:    Browser (survives close/reopen if persistent cookie)
  Storage:  Client-side (browser cookie)

RECOMMENDATION:
  Mobile apps/SPA: CookieLocaleResolver (stateless)
  Traditional web: SessionLocaleResolver
  Public content: AcceptHeaderLocaleResolver
```

---

### FixedLocaleResolver — For Single-Language Applications

```java
public class FixedLocaleResolver
    extends AbstractLocaleContextResolver {

    public FixedLocaleResolver() {
        setDefaultLocale(Locale.getDefault());
    }

    public FixedLocaleResolver(Locale defaultLocale) {
        setDefaultLocale(defaultLocale);
    }

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        Locale locale = getDefaultLocale();
        return (locale != null ? locale : Locale.getDefault());
        // Always returns the same locale
        // Ignores Accept-Language header
    }

    @Override
    public void setLocale(HttpServletRequest request,
                           @Nullable HttpServletResponse response,
                           @Nullable Locale locale) {
        throw new UnsupportedOperationException(
            "Cannot change fixed locale - " +
            "use a different locale resolution strategy");
    }
}
```

---

### LocaleContextHolder — Thread-Scoped Locale Access

```java
// FrameworkServlet.processRequest() sets up the LocaleContext:
protected void processRequest(HttpServletRequest request,
                               HttpServletResponse response) {
    // Save previous state (nested dispatch support)
    LocaleContext previousLocaleContext =
        LocaleContextHolder.getLocaleContext();

    // Build locale context from LocaleResolver
    LocaleContext localeContext =
        buildLocaleContext(request);
    // → localeResolver.resolveLocaleContext(request)

    // Set on ThreadLocal
    initContextHolders(request, localeContext, requestAttributes);
    // → LocaleContextHolder.setLocaleContext(localeContext)

    try {
        doService(request, response);
    } finally {
        // ALWAYS restore previous context (for nested dispatches)
        resetContextHolders(request,
            previousLocaleContext, previousAttributes);
    }
}

// Access from anywhere in request processing:
Locale locale = LocaleContextHolder.getLocale();
TimeZone tz = LocaleContextHolder.getTimeZone();
// Available in: services, components, view templates
// (indirectly via MessageSource + LocaleContextHolder)

// Thread safety:
// LocaleContextHolder uses InheritableThreadLocal by default
// Child threads DO inherit the locale context
// (unlike RequestContextHolder which uses ThreadLocal)
```

---

### MessageSource — Complete Architecture

```java
// MessageSource interface
public interface MessageSource {

    // Resolve message with args (for parameterized messages)
    @Nullable
    String getMessage(String code,
                      @Nullable Object[] args,
                      @Nullable String defaultMessage,
                      Locale locale);

    // Resolve required message (throws NoSuchMessageException if missing)
    String getMessage(String code,
                      @Nullable Object[] args,
                      Locale locale)
        throws NoSuchMessageException;

    // Resolve from MessageSourceResolvable (used by validation)
    String getMessage(MessageSourceResolvable resolvable,
                      Locale locale)
        throws NoSuchMessageException;
}
```

---

### ResourceBundleMessageSource — Java .properties Files

```java
@Bean
public MessageSource messageSource() {
    ResourceBundleMessageSource source =
        new ResourceBundleMessageSource();

    // Base name(s) for message files
    source.setBasenames(
        "messages",
        "validation",
        "errors"
    );
    // Looks for:
    //   messages.properties (default)
    //   messages_fr.properties (French)
    //   messages_fr_FR.properties (French/France)
    //   messages_de.properties (German)
    //   etc.
    // MULTIPLE base names: searched in order, first match wins

    // Encoding
    source.setDefaultEncoding("UTF-8");
    // CRITICAL: Java .properties default encoding is ISO-8859-1
    // Without this: UTF-8 characters corrupted
    // Alternative: Use Unicode escape sequences \u00e9 for é

    // Fallback to parent locale
    source.setFallbackToSystemLocale(true);
    // true: if messages_fr.properties missing → try messages.properties
    // false: if messages_fr.properties missing → NoSuchMessageException
    // Default: true

    // Use message format
    source.setUseCodeAsDefaultMessage(false);
    // false: NoSuchMessageException if code not found
    // true: returns the code itself as the message
    // Development: true (shows missing keys clearly)
    // Production: false (proper error handling)

    source.setCacheSeconds(-1);
    // -1: cache forever (production)
    // 0: no caching (dev — reload on every access)

    return source;
}
```

**Properties file structure:**
```properties
# messages.properties (default/English)
product.name.label=Product Name
product.price.label=Price
product.list.title=Product Catalog
product.list.empty=No products found
welcome.message=Welcome, {0}!
items.count=You have {0} item(s) in your cart.
```

```properties
# messages_fr.properties (French)
product.name.label=Nom du produit
product.price.label=Prix
product.list.title=Catalogue de produits
product.list.empty=Aucun produit trouvé
welcome.message=Bienvenue, {0}!
items.count=Vous avez {0} article(s) dans votre panier.
```

```properties
# messages_de.properties (German)
product.name.label=Produktname
product.price.label=Preis
product.list.title=Produktkatalog
product.list.empty=Keine Produkte gefunden
welcome.message=Willkommen, {0}!
items.count=Sie haben {0} Artikel in Ihrem Warenkorb.
```

---

### ReloadableResourceBundleMessageSource — Dynamic Reloading

```java
@Bean
public MessageSource messageSource() {
    ReloadableResourceBundleMessageSource source =
        new ReloadableResourceBundleMessageSource();

    // Can load from classpath AND filesystem
    source.setBasenames(
        "classpath:messages",
        // Look in classpath
        "file:/var/app/messages"
        // Look in filesystem (allows runtime updates)
    );

    source.setDefaultEncoding("UTF-8");

    // Check for changes every 60 seconds
    source.setCacheSeconds(60);
    // After 60 seconds: re-reads .properties files
    // If file changed: uses new content
    // Allows message updates without restart

    // ResourceBundleMessageSource vs ReloadableResourceBundleMessageSource:
    // ResourceBundleMessageSource:
    //   + Faster: uses Java ResourceBundle mechanism
    //   - Cannot reload without restart
    //   - classpath only (not filesystem)
    // ReloadableResourceBundleMessageSource:
    //   + Reloadable: setCacheSeconds(n) enables reloading
    //   + Filesystem locations supported
    //   - Slightly slower due to file polling
    //   Use in production when dynamic message updates needed

    return source;
}
```

---

### Message Code Resolution — The Fallback Chain

```java
// MessageSource resolves codes with a fallback chain:
// For Locale("fr", "FR"):
// 1. messages_fr_FR.properties → look for code
// 2. messages_fr.properties → look for code
// 3. messages.properties → look for code
// 4. NoSuchMessageException (if useCodeAsDefaultMessage=false)
//    OR return code itself (if useCodeAsDefaultMessage=true)

// PARAMETERIZED MESSAGES:
// messages.properties: welcome.message=Welcome, {0}! You have {1} messages.
// Java call:
String message = messageSource.getMessage(
    "welcome.message",
    new Object[]{"Alice", 5},
    Locale.ENGLISH
);
// → "Welcome, Alice! You have 5 messages."

// MessageFormat is used internally:
// {0}, {1}, {2} are positional arguments
// {0,number,currency} → formatted as currency
// {0,date,short} → formatted as short date
// {0,time} → formatted as time

// VALIDATION MESSAGE CODE HIERARCHY:
// When @NotNull validation fails for field "name" in Product class:
// Codes generated (in order of specificity):
// "NotNull.product.name"       ← most specific
// "NotNull.name"
// "NotNull.java.lang.String"
// "NotNull"                    ← least specific
// MessageSource tries each code in order
// First match wins
```

---

### Validation Messages — MessageSource Integration

```java
// messages.properties (validation messages):
# JSR-303 validation codes (generated by DefaultMessageCodesResolver)
NotNull=This field is required
NotNull.product.name=Product name is required
Size.product.name=Product name must be between {2} and {1} characters
Min.product.price=Price must be at least {1}
Email.user.email=Please enter a valid email address

# Pattern: {annotation}.{objectName}.{fieldName}
# Arguments in validation messages are index-based
# For @Size(min=2, max=100): {0}=field, {1}=max, {2}=min
# (actual argument order depends on constraint)

# Custom messages in ValidationMessages.properties
# (for Bean Validation / Hibernate Validator):
javax.validation.constraints.NotNull.message=Field cannot be null
```

```java
// Custom MessageCodesResolver:
@Bean
public MessageCodesResolver messageCodesResolver() {
    DefaultMessageCodesResolver resolver =
        new DefaultMessageCodesResolver();
    // Prefix format: controls code generation order
    resolver.setMessageCodeFormatter(
        MessageCodeFormatter.PREFIX_ERROR_CODE);
    // "PREFIX_ERROR_CODE": annotation.fieldName.objectName
    // "POSTFIX_ERROR_CODE": objectName.fieldName.annotation (default)
    return resolver;
}
```

---

### LocaleChangeInterceptor — Runtime Locale Switching

```java
public class LocaleChangeInterceptor
    implements HandlerInterceptor {

    public static final String DEFAULT_PARAM_NAME = "locale";

    private String paramName = DEFAULT_PARAM_NAME;

    // Restrict to specific HTTP methods
    private String[] httpMethods;

    // Ignore invalid locale values
    private boolean ignoreInvalidLocale = false;

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler)
            throws ServletException {

        // Check HTTP method restriction
        String requestMethod = request.getMethod();
        if (this.httpMethods != null) {
            boolean match = false;
            for (String method : this.httpMethods) {
                if (method.equalsIgnoreCase(requestMethod)) {
                    match = true;
                    break;
                }
            }
            if (!match) return true; // Method not allowed for locale change
        }

        // Check for locale parameter
        String newLocale = request.getParameter(
            getParamName());
        // Default param: "locale"
        // Custom: setParamName("lang")

        if (newLocale != null) {
            // Parse locale string
            Locale locale = parseLocaleValue(newLocale);

            // Change locale using LocaleResolver
            LocaleResolver localeResolver =
                RequestContextUtils.getLocaleResolver(request);

            if (localeResolver == null) {
                throw new IllegalStateException(
                    "No LocaleResolver found in DispatcherServlet");
            }

            try {
                localeResolver.setLocale(request, response, locale);
                // For AcceptHeaderLocaleResolver:
                // → UnsupportedOperationException
                // MUST use SessionLocaleResolver or CookieLocaleResolver
            } catch (UnsupportedOperationException ex) {
                if (!ignoreInvalidLocale) {
                    throw ex;
                }
            }
        }

        return true; // Always continue processing
    }
}
```

**LocaleChangeInterceptor + LocaleResolver pairing rules:**
```
AcceptHeaderLocaleResolver + LocaleChangeInterceptor:
  → UnsupportedOperationException (setLocale not supported)
  → IllegalStateException in interceptor
  WRONG COMBINATION

SessionLocaleResolver + LocaleChangeInterceptor:
  → Works correctly
  → Locale stored in session
  → Persists for session duration
  CORRECT COMBINATION

CookieLocaleResolver + LocaleChangeInterceptor:
  → Works correctly
  → Locale stored in cookie
  → Persists across sessions (if persistent cookie)
  CORRECT COMBINATION

FixedLocaleResolver + LocaleChangeInterceptor:
  → UnsupportedOperationException
  WRONG COMBINATION
```

---

### Complete Configuration

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // ── LOCALE RESOLVER ───────────────────────────────────────────
    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver =
            new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        // Default: English if no locale in session
        resolver.setDefaultTimeZone(TimeZone.getTimeZone("UTC"));
        return resolver;
    }

    // ── MESSAGE SOURCE ────────────────────────────────────────────
    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource source =
            new ReloadableResourceBundleMessageSource();
        source.setBasenames(
            "classpath:messages/messages",
            "classpath:messages/validation",
            "classpath:messages/errors"
        );
        source.setDefaultEncoding("UTF-8");
        source.setFallbackToSystemLocale(false);
        // false: only fall back to base (messages.properties)
        // not to JVM system locale
        source.setCacheSeconds(3600);
        return source;
    }

    // ── LOCALE CHANGE INTERCEPTOR ─────────────────────────────────
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        LocaleChangeInterceptor lci =
            new LocaleChangeInterceptor();
        lci.setParamName("lang");
        // ?lang=fr → switch to French
        // ?lang=de → switch to German
        // ?lang=en → switch to English
        lci.setIgnoreInvalidLocale(true);
        // Invalid locale values silently ignored
        registry.addInterceptor(lci);
    }

    // ── VALIDATOR WITH MESSAGE SOURCE ─────────────────────────────
    @Bean
    public LocalValidatorFactoryBean validator() {
        LocalValidatorFactoryBean validator =
            new LocalValidatorFactoryBean();
        validator.setValidationMessageSource(messageSource());
        // JSR-303 validation errors resolved via MessageSource
        return validator;
    }

    @Override
    public Validator getValidator() {
        return validator();
    }
}
```

---

### MessageSource in Controllers

```java
@RestController
@RequestMapping("/api")
public class LocalizedController {

    @Autowired
    private MessageSource messageSource;

    // Locale injected as method parameter
    @GetMapping("/greeting")
    public Map<String, String> greeting(Locale locale) {
        // Locale resolved from LocaleContextHolder
        // (set by FrameworkServlet from LocaleResolver)
        String greeting = messageSource.getMessage(
            "welcome.message",
            new Object[]{"World"},
            locale
        );
        return Map.of("greeting", greeting);
    }

    // Using MessageSourceAccessor for cleaner API
    @Autowired
    private MessageSourceAccessor messageSourceAccessor;
    // Wrapper around MessageSource that uses current locale

    @GetMapping("/products/{id}")
    public ResponseEntity<Product> getProduct(
            @PathVariable Long id,
            Locale locale) {
        return productService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.status(HttpStatus.NOT_FOUND)
                .header("X-Error-Message",
                    messageSource.getMessage(
                        "product.not.found",
                        new Object[]{id},
                        locale))
                .build());
    }
}
```

---

### MessageSourceAccessor — Simpler API

```java
// MessageSourceAccessor wraps MessageSource with current locale
@Bean
public MessageSourceAccessor messageSourceAccessor(
        MessageSource messageSource) {
    return new MessageSourceAccessor(messageSource);
    // Uses LocaleContextHolder.getLocale() for all lookups
    // No need to pass Locale to every getMessage() call
}

// Usage in services:
@Service
public class NotificationService {

    @Autowired
    private MessageSourceAccessor messageAccessor;

    public String buildWelcomeMessage(String username) {
        return messageAccessor.getMessage(
            "welcome.message",
            new Object[]{username}
        );
        // Automatically uses LocaleContextHolder.getLocale()
    }

    public String buildErrorMessage(String errorCode,
                                     Object[] args) {
        return messageAccessor.getMessage(errorCode, args);
    }
}
```

---

## 2️⃣ CODE EXAMPLES

### Complete i18n Setup — Spring Boot

```java
// Spring Boot: minimal configuration needed
// Auto-configures MessageSource from messages.properties

// application.properties:
// spring.messages.basename=messages,validation
// spring.messages.encoding=UTF-8
// spring.messages.cache-duration=3600s
// spring.messages.fallback-to-system-locale=false

// Only need to add:
@Configuration
public class I18nConfig {

    @Bean
    public LocaleResolver localeResolver() {
        SessionLocaleResolver resolver =
            new SessionLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        return resolver;
    }

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor lci =
            new LocaleChangeInterceptor();
        lci.setParamName("lang");
        return lci;
    }

    // Register the interceptor via WebMvcConfigurer
}

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Autowired
    private LocaleChangeInterceptor localeChangeInterceptor;

    @Override
    public void addInterceptors(
            InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor);
    }
}
```

---

### Thymeleaf i18n Integration

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <!-- #{key} → MessageSource.getMessage(key, locale) -->
    <title th:text="#{product.list.title}">Products</title>
</head>
<body>

<!-- Simple message -->
<h1 th:text="#{page.heading}">Heading</h1>

<!-- Message with parameters -->
<p th:text="#{welcome.message(${#authentication.name})}">
    Welcome!
</p>
<!-- welcome.message=Welcome, {0}! -->
<!-- → "Welcome, Alice!" -->

<!-- Message with multiple parameters -->
<p th:text="#{items.count(${cart.size})}">
    0 items
</p>

<!-- Conditional locale-aware text -->
<span th:text="${product.available}
               ? #{product.in.stock}
               : #{product.out.of.stock}">
    Status
</span>

<!-- Locale change links -->
<nav>
    <a th:href="@{${#request.requestURI}(lang='en')}">English</a>
    <a th:href="@{${#request.requestURI}(lang='fr')}">Français</a>
    <a th:href="@{${#request.requestURI}(lang='de')}">Deutsch</a>
</nav>

<!-- Current locale display -->
<span th:text="${#locale.language}">en</span>
<!-- or -->
<span th:text="${#locale}">en_US</span>

<!-- Date formatting locale-aware -->
<td th:text="${#temporals.format(product.createdAt, #{date.format})}">
    Date
</td>
<!-- messages.properties: date.format=MM/dd/yyyy -->
<!-- messages_de.properties: date.format=dd.MM.yyyy -->

</body>
</html>
```

---

### JSP i18n Integration

```jsp
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags" %>
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>

<!DOCTYPE html>
<html>
<head>
    <!-- Spring message tag - uses Spring's MessageSource -->
    <title><spring:message code="product.list.title"/></title>
</head>
<body>

<!-- Simple message -->
<h1><spring:message code="page.heading"/></h1>

<!-- Message with arguments -->
<p>
    <spring:message code="welcome.message"
                    arguments="${username}"/>
</p>

<!-- Message with multiple args -->
<p>
    <spring:message code="items.count"
                    arguments="${cartSize}"/>
</p>

<!-- Store message in variable -->
<spring:message code="delete.confirm"
                var="confirmMsg"/>
<button onclick="confirm('${confirmMsg}')">Delete</button>

<!-- JSTL fmt tags (if using JstlView - uses Spring MessageSource) -->
<fmt:message key="product.list.title"/>
<fmt:setLocale value="${sessionScope.locale}"/>

<!-- Current locale -->
<fmt:formatDate value="${now}"
                type="both"
                dateStyle="medium"
                timeStyle="short"/>
<!-- Renders date/time according to current locale -->

<!-- Locale change links -->
<a href="?lang=en">English</a>
<a href="?lang=fr">Français</a>

</body>
</html>
```

---

### Custom LocaleResolver — Database-Backed

```java
// Store user's locale preference in database
@Component("localeResolver")
public class UserPreferenceLocaleResolver
    implements LocaleContextResolver {

    @Autowired
    private UserPreferenceService preferenceService;

    private static final String SESSION_KEY =
        "RESOLVED_LOCALE";

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        return resolveLocaleContext(request).getLocale();
    }

    @Override
    public LocaleContext resolveLocaleContext(
            HttpServletRequest request) {

        // 1. Check session cache first (performance)
        HttpSession session = request.getSession(false);
        if (session != null) {
            Locale cached = (Locale)
                session.getAttribute(SESSION_KEY);
            if (cached != null) {
                return new SimpleLocaleContext(cached);
            }
        }

        // 2. Check if user is authenticated
        Principal principal = request.getUserPrincipal();
        if (principal != null) {
            // Load from database
            Locale userLocale =
                preferenceService.getUserLocale(
                    principal.getName());
            if (userLocale != null) {
                // Cache in session
                request.getSession().setAttribute(
                    SESSION_KEY, userLocale);
                return new SimpleLocaleContext(userLocale);
            }
        }

        // 3. Fall back to Accept-Language
        return new SimpleLocaleContext(
            request.getLocale());
    }

    @Override
    public void setLocale(HttpServletRequest request,
                           @Nullable HttpServletResponse response,
                           @Nullable Locale locale) {
        Principal principal = request.getUserPrincipal();
        if (principal != null && locale != null) {
            // Persist to database
            preferenceService.saveUserLocale(
                principal.getName(), locale);
            // Update session cache
            request.getSession().setAttribute(
                SESSION_KEY, locale);
        }
    }

    @Override
    public void setLocaleContext(
            HttpServletRequest request,
            @Nullable HttpServletResponse response,
            @Nullable LocaleContext localeContext) {
        setLocale(request, response,
            localeContext != null ?
                localeContext.getLocale() : null);
    }
}
```

---

### Validation Messages Locale Integration

```java
// Bean validation with locale-aware error messages
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<User> createUser(
            @Valid @RequestBody UserRequest request,
            Locale locale) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(userService.create(request));
    }
}

// @ExceptionHandler for validation errors with locale
@RestControllerAdvice
public class ValidationExceptionHandler {

    @Autowired
    private MessageSource messageSource;

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handle(
            MethodArgumentNotValidException ex,
            Locale locale) {

        Map<String, String> fieldErrors =
            new LinkedHashMap<>();

        ex.getBindingResult().getFieldErrors().forEach(error -> {
            // error.getCodes() = ["NotNull.user.email",
            //                      "NotNull.email",
            //                      "NotNull.java.lang.String",
            //                      "NotNull"]
            // MessageSource.getMessage(error, locale)
            // → Tries each code, first match wins
            String message = messageSource.getMessage(
                error, locale);
            fieldErrors.put(error.getField(), message);
        });

        return ResponseEntity.badRequest()
            .body(Map.of(
                "errors", fieldErrors,
                "locale", locale.getLanguage()
            ));
    }
}

// messages_fr.properties:
# Validation messages in French
NotNull=Ce champ est obligatoire
NotNull.user.email=L''adresse e-mail est obligatoire
Size.product.name=Le nom doit contenir entre {2} et {1} caractères
Email=Veuillez saisir une adresse e-mail valide
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
`AcceptHeaderLocaleResolver` is configured as `localeResolver` bean. `LocaleChangeInterceptor` with `paramName="lang"` is registered. A request arrives with `?lang=fr`. What happens?

A) Locale changes to French — `LocaleChangeInterceptor` updates session
B) `UnsupportedOperationException` thrown — `AcceptHeaderLocaleResolver.setLocale()` throws
C) Request ignored — `lang` parameter silently ignored
D) `IllegalArgumentException` — incompatible resolver and interceptor

**Answer: B**
`LocaleChangeInterceptor.preHandle()` finds `lang=fr` parameter. Calls `localeResolver.setLocale(request, response, Locale.FRENCH)`. `AcceptHeaderLocaleResolver.setLocale()` throws `UnsupportedOperationException` — the Accept-Language header is sent by the client and cannot be changed by the server. The exception propagates from the interceptor. Must use `SessionLocaleResolver` or `CookieLocaleResolver` with `LocaleChangeInterceptor`.

---

**Q2 — Select All That Apply**
`ResourceBundleMessageSource` is configured with `setFallbackToSystemLocale(false)`. A request uses `Locale.FRENCH`. Only `messages.properties` and `messages_de.properties` exist (no French file). Which behaviours are TRUE?

A) `NoSuchMessageException` thrown for all message lookups in French
B) Messages fall back to `messages.properties` (base file)
C) Messages fall back to JVM system locale properties file
D) `setFallbackToSystemLocale(false)` means: don't fall back to JVM system locale, still fall back to base file
E) `setUseCodeAsDefaultMessage(true)` would return the message code as the message

**Answer: B, D, E**
A is wrong — fallback to base `messages.properties` ALWAYS happens regardless of `setFallbackToSystemLocale`. `setFallbackToSystemLocale(false)` specifically controls whether to check the JVM's system locale properties BEFORE the base file. With `false`: French → `messages_fr.properties` (missing) → `messages.properties` (found). C is wrong — JVM system locale fallback is DISABLED. D correctly describes the behaviour. E is correct — `useCodeAsDefaultMessage=true` returns the code itself if no message found.

---

**Q3 — Ordering**
For a request with `?lang=de`, order the locale processing steps:

- `LocaleContextHolder.getLocale()` returns German
- `localeResolver.setLocale()` called with `Locale.GERMAN`
- `messageSource.getMessage("key", args, locale)` uses German
- `LocaleChangeInterceptor.preHandle()` reads `lang=de`
- `FrameworkServlet` sets `LocaleContextHolder` from resolver

**Correct Order:**
1. `LocaleChangeInterceptor.preHandle()` reads `lang=de`
2. `localeResolver.setLocale()` called with `Locale.GERMAN`
3. `FrameworkServlet` sets `LocaleContextHolder` from resolver
   (Actually: LocaleContextHolder is set BEFORE interceptors run — but setLocale updates the session/cookie which affects NEXT request. The flow for THIS request is: FrameworkServlet sets context from CURRENT session value FIRST, then interceptor changes for NEXT request)
4. `LocaleContextHolder.getLocale()` returns German (next request)
5. `messageSource.getMessage("key", args, locale)` uses German

**Corrected Order (exact Spring behaviour):**
1. `FrameworkServlet.processRequest()` sets `LocaleContextHolder` from current session (previous locale)
2. `LocaleChangeInterceptor.preHandle()` reads `lang=de`
3. `localeResolver.setLocale()` stores German in session
4. Handler executes — `LocaleContextHolder.getLocale()` still returns PREVIOUS locale for THIS request (set in step 1)
5. `messageSource.getMessage()` uses PREVIOUS locale for THIS request
6. NEXT REQUEST: steps 1-4 use German

---

**Q4 — MCQ**
`@NotNull` validation fails for field `name` in object `product`. `DefaultMessageCodesResolver` generates multiple codes. In what order are they tried in MessageSource?

A) `NotNull` → `NotNull.name` → `NotNull.product.name` → `NotNull.java.lang.String`
B) `NotNull.product.name` → `NotNull.name` → `NotNull.java.lang.String` → `NotNull`
C) `NotNull.java.lang.String` → `NotNull.name` → `NotNull.product.name` → `NotNull`
D) Order is non-deterministic — implementation detail

**Answer: B**
`DefaultMessageCodesResolver` generates codes from most specific to least specific:
1. `NotNull.product.name` (annotation + object + field)
2. `NotNull.name` (annotation + field)
3. `NotNull.java.lang.String` (annotation + type)
4. `NotNull` (annotation only)

`MessageSource.getMessage(MessageSourceResolvable, locale)` tries codes in this order. The MOST SPECIFIC matching message wins. This allows overriding generic messages with field-specific ones.

---

**Q5 — True/False**
`LocaleContextHolder` uses `InheritableThreadLocal` by default, so child threads spawned in async handler methods have access to the current locale.

**Answer: True**
`LocaleContextHolder` uses `InheritableThreadLocal` (controlled by `inheritable` flag, default `true`). Child threads (spawned from the request thread) inherit the `LocaleContext`. This is different from `RequestContextHolder` which uses plain `ThreadLocal` — child threads do NOT inherit the request context. This matters for async operations that need locale context (e.g., sending localized emails from async threads).

---

**Q6 — MCQ**
`ReloadableResourceBundleMessageSource` with `setCacheSeconds(60)` is configured. A message is changed in `messages_fr.properties` at runtime. When does the application serve the new message?

A) Immediately — `setCacheSeconds(0)` would serve immediately
B) After the JVM process is restarted
C) After up to 60 seconds — cache TTL expires and file is re-read
D) After `ApplicationContext.refresh()` is called

**Answer: C**
`setCacheSeconds(60)` means: cache resolved messages for 60 seconds. After 60 seconds: next access checks if the file has been modified (via `lastModified()` timestamp). If modified: reloads the file and updates the cache. Between reloads: serves stale cached values. The maximum delay between file change and new message serving is 60 seconds.

---

**Q7 — Select All That Apply**
Which statements about `CookieLocaleResolver` vs `SessionLocaleResolver` are TRUE?

A) `CookieLocaleResolver` stores locale on the CLIENT; `SessionLocaleResolver` stores on the SERVER
B) `CookieLocaleResolver` with `cookieMaxAge=-1` is equivalent to `SessionLocaleResolver` persistence
C) `CookieLocaleResolver` locale survives browser close if `cookieMaxAge > 0`
D) `SessionLocaleResolver` locale is visible to other applications sharing the same session
E) Both require `LocaleChangeInterceptor` to change the locale

**Answer: A, B, C**
A is correct — cookie is client-side, session is server-side. B is correct — `cookieMaxAge=-1` creates a session cookie (expires on browser close) equivalent to session. C is correct — persistent cookie (`cookieMaxAge > 0`) survives browser restart. D is wrong — `SessionLocaleResolver` uses a namespaced key including the class name, not likely to collide with other apps. E is partially true but misleading — locale can also be changed programmatically by calling `localeResolver.setLocale()` directly.

---

**Q8 — Scenario**
`messages.properties` contains `items.count=You have {0} item(s)`. Controller calls `messageSource.getMessage("items.count", new Object[]{5}, Locale.ENGLISH)`. What is returned?

A) `"You have {0} item(s)"` — MessageFormat not applied by default
B) `"You have 5 item(s)"` — `{0}` replaced with `5`
C) `"You have 5.0 item(s)"` — `{0}` always formats as floating point
D) `NumberFormatException` — `{0}` requires explicit format specification

**Answer: B**
Spring's `ResourceBundleMessageSource` uses `java.text.MessageFormat` for messages with arguments. `MessageFormat.format("You have {0} item(s)", 5)` → `"You have 5 item(s)"`. `{0}` without format specifier uses `toString()` of the argument. For `Integer 5` → `"5"`. Result: `"You have 5 item(s)"`.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `AcceptHeaderLocaleResolver` + `LocaleChangeInterceptor` = Exception**
The most common i18n misconfiguration. `LocaleChangeInterceptor` calls `setLocale()` on whatever resolver is configured. `AcceptHeaderLocaleResolver.setLocale()` throws `UnsupportedOperationException`. This exception propagates as a 500 error — confusing because locale switching "should work." Always pair `LocaleChangeInterceptor` with `SessionLocaleResolver` or `CookieLocaleResolver`.

**Trap 2 — Locale change takes effect on NEXT request, not current**
`LocaleChangeInterceptor` runs in `preHandle()`. It calls `localeResolver.setLocale()` which stores the new locale in session/cookie. BUT — `FrameworkServlet.processRequest()` already called `LocaleContextHolder.setLocaleContext()` BEFORE interceptors ran, using the PREVIOUS locale from session. The current request's `LocaleContextHolder.getLocale()` returns the OLD locale. Messages in the CURRENT request are still in the old language. New locale takes effect on the NEXT request.

**Trap 3 — `setDefaultEncoding("UTF-8")` is mandatory**
Java `.properties` files have a default encoding of ISO-8859-1. Without `setDefaultEncoding("UTF-8")`, non-ASCII characters (French accents, German umlauts, Chinese characters) are corrupted. This is a silent bug — the application starts without error, but international characters display as garbage. Always set `setDefaultEncoding("UTF-8")` on `MessageSource` beans.

**Trap 4 — Message code fallback order**
When MessageSource cannot find a code for the requested locale, it falls back: `fr_FR` → `fr` → `messages.properties`. `setFallbackToSystemLocale(true)` (default) adds the JVM system locale as an intermediate step: `fr_FR` → `fr` → JVM locale properties → `messages.properties`. If JVM locale is Japanese and system has `messages_ja.properties`, a French user might get Japanese messages. Set `setFallbackToSystemLocale(false)` for predictable behaviour.

**Trap 5 — MessageFormat escaping**
`{0}` in messages is `MessageFormat` syntax. If your message contains `{` or `}` as literal characters (e.g., JSON examples in messages), they need escaping: `{{` for literal `{`. Also: single quotes must be escaped as `''` (two single quotes). `You're welcome` must be written `You''re welcome`. This causes confusing message rendering bugs.

**Trap 6 — Bean name "messageSource" is mandatory**
`ApplicationContext` looks for a bean named EXACTLY `"messageSource"` for i18n. If you name it `"myMessageSource"`, Spring won't find it for automatic locale resolution. The bean name must be `"messageSource"`. This is unlike most Spring beans where name doesn't matter.

**Trap 7 — Validation message vs application message — different files**
Bean Validation (`@NotNull`, `@Size`) resolves messages from `ValidationMessages.properties` (Bean Validation standard) AND from Spring's `MessageSource` (when configured via `LocalValidatorFactoryBean.setValidationMessageSource()`). If not configured, validation uses its own default messages (Hibernate Validator's built-ins). To get localized validation messages via Spring's MessageSource, explicitly wire it: `validator.setValidationMessageSource(messageSource())`.

---

## 5️⃣ SUMMARY SHEET

```
I18N COMPONENT HIERARCHY
─────────────────────────────────────────────────────
LocaleResolver → resolves Locale from HTTP request
  → FrameworkServlet stores in LocaleContextHolder (ThreadLocal)
  → View layer reads via LocaleContextHolder.getLocale()

LocaleChangeInterceptor (preHandle) → calls LocaleResolver.setLocale()
  → Stores new locale in session/cookie
  → New locale active from NEXT request

MessageSource → resolves message code to text
  → fallback chain: fr_FR → fr → base → exception/code

LOCALE RESOLVER COMPARISON
─────────────────────────────────────────────────────
AcceptHeaderLocaleResolver:
  Source: Accept-Language HTTP header
  setLocale(): THROWS UnsupportedOperationException
  Use with: NO LocaleChangeInterceptor (read-only)
  Persistence: None — per-request from header

SessionLocaleResolver:
  Source: HttpSession attribute
  setLocale(): Stores in session
  Use with: LocaleChangeInterceptor
  Persistence: Session lifetime
  Default: configurable (defaultLocale)

CookieLocaleResolver:
  Source: HTTP Cookie
  setLocale(): Writes cookie to response
  Use with: LocaleChangeInterceptor
  Persistence: cookieMaxAge (-1=session, positive=permanent)
  Survives browser close if maxAge > 0

FixedLocaleResolver:
  Source: Always configured locale
  setLocale(): THROWS UnsupportedOperationException
  Use: Single-language apps

LOCALE CHANGE INTERCEPTOR RULES
─────────────────────────────────────────────────────
Compatible resolvers: Session, Cookie
Incompatible: AcceptHeader, Fixed → UnsupportedOperationException
paramName default: "locale" (customise with setParamName)
setIgnoreInvalidLocale(true): bad locale values silently ignored
Locale change effective: NEXT request (not current)

MESSAGESOURCE CONFIGURATION
─────────────────────────────────────────────────────
Bean MUST be named: "messageSource" (exact name)
setDefaultEncoding("UTF-8"): REQUIRED for non-ASCII characters
setFallbackToSystemLocale(false): predictable fallback to base file
setUseCodeAsDefaultMessage(true): returns code if not found (dev)

Fallback chain:
  messages_fr_FR.properties → messages_fr.properties
  → (if fallbackToSystemLocale=true: JVM locale)
  → messages.properties → NoSuchMessageException (or code)

ResourceBundleMessageSource:
  + Fast: uses Java ResourceBundle
  - No reload without restart
  - Classpath only

ReloadableResourceBundleMessageSource:
  + Reloadable: setCacheSeconds(n)
  + Filesystem locations
  - Slightly slower

PARAMETERIZED MESSAGES (MessageFormat)
─────────────────────────────────────────────────────
welcome=Hello, {0}!
→ getMessage("welcome", new Object[]{"Alice"}, locale)
→ "Hello, Alice!"

{0,number,currency} → locale-sensitive currency format
{0,date,short}      → locale-sensitive date format
Escape single quotes: '' (two single quotes)
Escape braces: {{ and }} for literal { }

VALIDATION MESSAGE CODE HIERARCHY
─────────────────────────────────────────────────────
For @NotNull on Product.name:
1. NotNull.product.name   (most specific → checked first)
2. NotNull.name
3. NotNull.java.lang.String
4. NotNull               (least specific → checked last)

Wire to MessageSource: LocalValidatorFactoryBean.setValidationMessageSource()
Without wiring: Hibernate Validator's own messages used (not localized)

LOCALCONTEXTHOLDER THREADING
─────────────────────────────────────────────────────
InheritableThreadLocal: child threads inherit locale (default)
  → Async methods spawned from request thread have locale
RequestContextHolder: regular ThreadLocal
  → Child threads do NOT have request context

THYMELEAF I18N
─────────────────────────────────────────────────────
#{key}              → messageSource.getMessage(key, locale)
#{key(${arg})}      → with parameter
${#locale}          → current Locale
${#locale.language} → language code ("fr")

JSP I18N
─────────────────────────────────────────────────────
<spring:message code="key"/>               → MessageSource
<spring:message code="key" arguments="${arg}"/>
JstlView required for fmt:message → Spring MessageSource
Without JstlView: fmt:message uses JSTL resource bundle only

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "AcceptHeaderLocaleResolver + LocaleChangeInterceptor = UnsupportedOperationException"
• "Locale change in preHandle takes effect NEXT request — not current request"
• "MessageSource bean must be named 'messageSource' — exact name required"
• "setDefaultEncoding('UTF-8') mandatory — default is ISO-8859-1"
• "setFallbackToSystemLocale(false) — predictable; true could serve wrong language"
• "Validation message codes: NotNull.product.name → NotNull.name → NotNull (most to least specific)"
• "LocalValidatorFactoryBean.setValidationMessageSource() required for localized validation"
• "LocaleContextHolder uses InheritableThreadLocal — child threads inherit locale"
• "CookieLocaleResolver cookieMaxAge=-1 = session cookie; positive = persistent"
• "ReloadableResourceBundleMessageSource: setCacheSeconds(n) enables runtime reload"
```

---
