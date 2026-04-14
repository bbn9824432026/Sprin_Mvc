# TOPIC 5.2 — CORS CONFIGURATION: @CrossOrigin, GLOBAL CORS, PRE-FLIGHT REQUESTS

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What CORS Really Is — The Browser Security Model

CORS (Cross-Origin Resource Sharing) is a **browser security mechanism** that restricts JavaScript code from making HTTP requests to a different origin than the page it came from. Understanding CORS deeply means understanding the browser's Same-Origin Policy, why pre-flight requests exist, exactly what headers travel in each direction, and how Spring MVC processes them.

---

### Same-Origin Policy — The Root Cause

```
SAME ORIGIN:
  Page at: https://app.example.com:443/dashboard
  Request to: https://app.example.com:443/api/products
  → SAME origin (scheme + host + port all match)
  → Browser allows the request directly

CROSS-ORIGIN:
  Page at: https://app.example.com/dashboard
  Request to: https://api.example.com/products
  → DIFFERENT host → CROSS-ORIGIN
  → Browser applies CORS restrictions

  Page at: http://app.example.com/dashboard
  Request to: https://app.example.com/api/products
  → DIFFERENT scheme (http vs https) → CROSS-ORIGIN

  Page at: https://app.example.com:3000/dashboard
  Request to: https://app.example.com:443/api/products
  → DIFFERENT port → CROSS-ORIGIN

WHAT CORS IS NOT:
  → Not a Spring security feature — it's a browser feature
  → Server still receives ALL requests (even without CORS headers)
  → Server's CORS headers tell the BROWSER what's allowed
  → Curl, Postman, server-to-server: not affected by CORS
  → Only browser JavaScript cross-origin requests are restricted
```

---

### Simple Requests vs Pre-flight Requests

**Simple requests** (browser sends directly without pre-flight):
```
Conditions for a "simple" request:
  Method: GET, POST, or HEAD only
  Content-Type: application/x-www-form-urlencoded,
                multipart/form-data, or text/plain
  No custom headers (Authorization, X-Custom, etc.)
  No credentials in complex ways

Flow:
  Browser → Server: GET /api/products
                    Origin: https://app.example.com

  Server → Browser: 200 OK
                    Access-Control-Allow-Origin: https://app.example.com
                    [response body]

  Browser: checks Access-Control-Allow-Origin
  → Matches origin? → JavaScript receives response
  → Doesn't match? → JavaScript blocked (response still reached server)
```

**Pre-flight requests** (browser sends OPTIONS first):
```
Conditions that REQUIRE pre-flight:
  Method: PUT, DELETE, PATCH, or custom methods
  Content-Type: application/json (common trigger)
  Custom headers: Authorization, X-API-Key, X-Custom, etc.
  Any header not in "safe" list

Flow:
  Step 1: Browser sends pre-flight OPTIONS request
  Browser → Server: OPTIONS /api/products/42
                    Origin: https://app.example.com
                    Access-Control-Request-Method: DELETE
                    Access-Control-Request-Headers: Authorization, Content-Type

  Step 2: Server responds to pre-flight
  Server → Browser: 200 OK (or 204)
                    Access-Control-Allow-Origin: https://app.example.com
                    Access-Control-Allow-Methods: GET, POST, PUT, DELETE
                    Access-Control-Allow-Headers: Authorization, Content-Type
                    Access-Control-Max-Age: 3600
                    [NO response body]

  Step 3: Browser evaluates pre-flight response
  → All requested method/headers allowed? → Send actual request

  Step 4: Actual request
  Browser → Server: DELETE /api/products/42
                    Origin: https://app.example.com
                    Authorization: Bearer token

  Step 5: Server responds to actual request
  Server → Browser: 200 OK
                    Access-Control-Allow-Origin: https://app.example.com
                    [response body]
```

---

### CORS Headers — Complete Reference

**Request headers (sent by browser):**
```
Origin: https://app.example.com
  → Present on ALL cross-origin requests (simple AND pre-flight)
  → Present on same-origin requests only in some cases

Access-Control-Request-Method: DELETE
  → Pre-flight only: what method the actual request will use

Access-Control-Request-Headers: Authorization, Content-Type
  → Pre-flight only: what custom headers the actual request will send
```

**Response headers (sent by server):**
```
Access-Control-Allow-Origin: https://app.example.com
  → Required on ALL CORS responses (simple AND actual request)
  → Value: specific origin OR "*" (wildcard)
  → "*" cannot be used with Allow-Credentials: true

Access-Control-Allow-Methods: GET, POST, PUT, DELETE, PATCH, OPTIONS
  → Required on pre-flight response
  → What HTTP methods are permitted

Access-Control-Allow-Headers: Authorization, Content-Type, X-Custom
  → Required on pre-flight when request has custom headers
  → What request headers are permitted

Access-Control-Allow-Credentials: true
  → Optional: allows browser to include cookies/auth headers
  → Cannot be combined with Allow-Origin: *
  → Requires Allow-Origin to be a specific origin

Access-Control-Expose-Headers: X-Total-Count, X-Rate-Limit
  → Optional: which response headers JavaScript can read
  → By default, only "safe" headers are readable

Access-Control-Max-Age: 3600
  → Optional: seconds to cache pre-flight response
  → Browser won't send pre-flight again for this duration
  → Default browser max: 600 seconds (Chrome), 86400 (Firefox)
```

---

### Spring MVC CORS Processing — The Complete Pipeline

```
Request arrives at DispatcherServlet
        │
        ▼
AbstractHandlerMapping.getHandler()
        │
        ├── Not a CORS request? → normal processing
        │
        └── IS a CORS request (has Origin header)?
              │
              ├── Is it a pre-flight (OPTIONS + ACRS-Method)?
              │     → getCorsHandlerExecutionChain()
              │     → CorsProcessor.processPreFlightRequest()
              │     → Returns PreFlightHandler
              │     → Sets CORS response headers
              │     → Returns 200/204 with no body
              │
              └── Is it an actual CORS request?
                    → getCorsHandlerExecutionChain()
                    → Wraps chain with CorsInterceptor
                    → CorsInterceptor runs in preHandle()
                    → Sets CORS response headers on actual response
                    → If origin not allowed → 403 Forbidden
```

---

### CorsConfiguration — The Data Object

```java
public class CorsConfiguration {

    // Allowed origins (exact match)
    @Nullable
    private List<String> allowedOrigins;
    // null = not configured (different from empty list)
    // ["*"] = any origin allowed
    // ["https://app.example.com"] = specific origin

    // Allowed origin patterns (regex-like)
    @Nullable
    private List<String> allowedOriginPatterns;
    // "https://*.example.com" = any subdomain
    // "https://{app,api}.example.com" = specific subdomains

    // Allowed HTTP methods
    @Nullable
    private List<String> allowedMethods;
    // null = defaults to GET, HEAD
    // ["*"] = all methods

    // Allowed request headers
    @Nullable
    private List<String> allowedHeaders;
    // null = only safe headers
    // ["*"] = all headers allowed

    // Headers exposed to JavaScript
    @Nullable
    private List<String> exposedHeaders;

    // Allow credentials (cookies, auth headers)
    @Nullable
    private Boolean allowCredentials;
    // Cannot be true when allowedOrigins=["*"]

    // Pre-flight cache duration
    @Nullable
    private Long maxAge;
    // Seconds to cache pre-flight response

    // Combine two CorsConfiguration objects
    @Nullable
    public CorsConfiguration combine(
            @Nullable CorsConfiguration other) {
        // Handler-level + global level combined
        // Handler-level overrides global for same properties
    }

    // Validate: credentials + wildcard origin = exception
    public void validateAllowCredentials() {
        if (Boolean.TRUE.equals(this.allowCredentials) &&
                this.allowedOrigins != null &&
                this.allowedOrigins.contains(ALL)) {
            throw new IllegalArgumentException(
                "When allowCredentials is true, " +
                "allowedOrigins cannot contain '*'");
        }
    }
}
```

---

### @CrossOrigin — Method and Class Level

```java
// @CrossOrigin annotation:
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CrossOrigin {

    // Allowed origins — same as CorsConfiguration.allowedOrigins
    @AliasFor("origins")
    String[] value() default {};

    @AliasFor("value")
    String[] origins() default {};

    // Allowed origin patterns
    String[] originPatterns() default {};

    // Allowed headers — default: all (*)
    String[] allowedHeaders() default {};

    // Exposed headers
    String[] exposedHeaders() default {};

    // Allowed methods — default: same as handler's @RequestMapping methods
    RequestMethod[] methods() default {};

    // Allow credentials — default: not set (false)
    String allowCredentials() default "";
    // Must be "true" or "false" string, or empty

    // Max age for pre-flight cache
    long maxAge() default -1;
    // -1 means not set → use default (1800 seconds)
}
```

**@CrossOrigin at different levels:**

```java
// CLASS LEVEL — applies to ALL methods in the controller
@RestController
@RequestMapping("/api/products")
@CrossOrigin(
    origins = "https://app.example.com",
    allowedHeaders = {"Authorization", "Content-Type"},
    methods = {RequestMethod.GET, RequestMethod.POST},
    maxAge = 3600
)
public class ProductController {

    // Inherits class-level CORS config
    @GetMapping
    public List<Product> getAll() { ... }

    // METHOD LEVEL — OVERRIDES class-level for this method
    @DeleteMapping("/{id}")
    @CrossOrigin(
        origins = {"https://app.example.com",
                   "https://admin.example.com"}
        // This method allows additional origin
        // OTHER properties come from class level (combined)
    )
    public void delete(@PathVariable Long id) { ... }

    // NO annotation — inherits class-level CORS config
    @PostMapping
    public Product create(@RequestBody Product product) { ... }
}

// COMBINING: method-level MERGED with class-level
// For delete():
// origins: method-level wins → {app.example.com, admin.example.com}
// allowedHeaders: class-level → {Authorization, Content-Type}
// methods: class-level → {GET, POST} + DELETE (method's own mapping)
```

---

### RequestMappingHandlerMapping CORS Processing

```java
// AbstractHandlerMapping.getHandler() — CORS integration point
@Override
@Nullable
public final HandlerExecutionChain getHandler(
        HttpServletRequest request) throws Exception {

    Object handler = getHandlerInternal(request);
    // ... standard handler lookup

    // CORS HANDLING:
    if (hasCorsConfigurationSource(handler) ||
            CorsUtils.isPreFlightRequest(request)) {

        // Get CORS configuration for this handler
        CorsConfiguration config =
            getCorsConfiguration(handler, request);

        // Combine with global CORS configuration
        if (getCorsConfigurationSource() != null) {
            CorsConfiguration globalConfig =
                getCorsConfigurationSource()
                    .getCorsConfiguration(request);
            config = (globalConfig != null ?
                globalConfig.combine(config) : config);
        }

        if (config != null) {
            config.validateAllowCredentials();
        }

        // Build CORS-aware execution chain
        executionChain =
            getCorsHandlerExecutionChain(
                request, executionChain, config);
    }

    return executionChain;
}

// getCorsHandlerExecutionChain():
protected HandlerExecutionChain getCorsHandlerExecutionChain(
        HttpServletRequest request,
        HandlerExecutionChain chain,
        @Nullable CorsConfiguration config) {

    if (CorsUtils.isPreFlightRequest(request)) {
        // PRE-FLIGHT: replace handler with CorsProcessor
        HandlerInterceptor[] interceptors =
            chain.getInterceptors();
        chain = new HandlerExecutionChain(
            new PreFlightHandler(config), interceptors);
        // PreFlightHandler processes the OPTIONS request
        // and sets all Access-Control-Allow-* headers
    } else {
        // ACTUAL CORS REQUEST: add interceptor
        chain.addInterceptor(0,
            new CorsInterceptor(config));
        // CorsInterceptor.preHandle() validates origin
        // and sets Access-Control-Allow-Origin header
    }

    return chain;
}
```

---

### DefaultCorsProcessor — The Header Writer

```java
public class DefaultCorsProcessor implements CorsProcessor {

    @Override
    public boolean processRequest(
            @Nullable CorsConfiguration config,
            HttpServletRequest request,
            HttpServletResponse response)
            throws IOException {

        Collection<String> varyHeaders =
            response.getHeaders(HttpHeaders.VARY);
        if (!varyHeaders.contains(HttpHeaders.ORIGIN)) {
            response.addHeader(HttpHeaders.VARY,
                HttpHeaders.ORIGIN);
        }
        // Vary: Origin tells caches to treat responses
        // with different Origin headers as different

        String requestOrigin =
            request.getHeader(HttpHeaders.ORIGIN);

        String allowOrigin =
            checkOrigin(config, requestOrigin);
        // Validates requestOrigin against config.allowedOrigins
        // Returns null if not allowed

        if (allowOrigin == null) {
            // Origin not allowed
            logger.debug("Reject: '" + requestOrigin +
                "' origin is not allowed");
            rejectRequest(response);
            return false;
            // response.sendError(HttpServletResponse.SC_FORBIDDEN)
            // → HTTP 403 Forbidden
        }

        if (CorsUtils.isPreFlightRequest(request)) {
            handlePreflightResponse(config, response,
                allowOrigin);
            return false;
            // false = "request was handled by CORS processor"
            // actual handler should NOT process it
        }

        handleSimpleResponse(config, response, allowOrigin);
        return true;
        // true = "continue to actual handler"
    }

    private void handlePreflightResponse(
            CorsConfiguration config,
            HttpServletResponse response,
            String allowOrigin) {

        response.addHeader(
            HttpHeaders.ACCESS_CONTROL_ALLOW_ORIGIN,
            allowOrigin);

        if (config.getAllowCredentials() != null &&
                config.getAllowCredentials()) {
            response.addHeader(
                HttpHeaders.ACCESS_CONTROL_ALLOW_CREDENTIALS,
                "true");
        }

        List<HttpMethod> allowedMethods =
            config.resolveAllowedMethods(request);
        response.addHeader(
            HttpHeaders.ACCESS_CONTROL_ALLOW_METHODS,
            StringUtils.collectionToCommaDelimitedString(
                allowedMethods));

        List<String> allowedHeaders =
            config.resolveAllowedHeaders(request);
        if (!CollectionUtils.isEmpty(allowedHeaders)) {
            response.addHeader(
                HttpHeaders.ACCESS_CONTROL_ALLOW_HEADERS,
                StringUtils.collectionToCommaDelimitedString(
                    allowedHeaders));
        }

        if (config.getMaxAge() != null) {
            response.addHeader(
                HttpHeaders.ACCESS_CONTROL_MAX_AGE,
                config.getMaxAge().toString());
        }

        response.setStatus(HttpServletResponse.SC_NO_CONTENT);
        // 204 No Content for pre-flight
        // (some implementations use 200)
    }
}
```

---

### Global CORS Configuration — WebMvcConfigurer

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {

        // GLOBAL: applies to ALL endpoints matching the pattern

        registry.addMapping("/api/**")
                // URL pattern — applies to all /api/** endpoints
                .allowedOrigins(
                    "https://app.example.com",
                    "https://admin.example.com")
                // EXACT origins allowed
                // Cannot use with allowCredentials=true and "*"

                .allowedOriginPatterns(
                    "https://*.example.com")
                // Pattern-based (alternative to allowedOrigins)
                // Allows all subdomains of example.com
                // CAN be used with credentials=true (unlike "*")

                .allowedMethods(
                    "GET", "POST", "PUT",
                    "DELETE", "PATCH", "OPTIONS")

                .allowedHeaders(
                    "Authorization",
                    "Content-Type",
                    "X-Requested-With",
                    "X-CSRF-Token")

                .exposedHeaders(
                    "X-Total-Count",
                    "X-Rate-Limit-Remaining")
                // These response headers visible to JavaScript

                .allowCredentials(true)
                // Allows cookies, Authorization header
                // Requires specific origins (not "*")

                .maxAge(3600L);
                // Cache pre-flight for 1 hour

        // DIFFERENT config for public API
        registry.addMapping("/public/api/**")
                .allowedOrigins("*")
                // Any origin allowed
                // BUT: cannot use allowCredentials(true) with "*"
                .allowedMethods("GET", "HEAD")
                .maxAge(86400L);
    }
}
```

---

### @CrossOrigin vs Global CORS — Combination Rules

```java
// Combination algorithm in AbstractHandlerMapping:

// handler-level CorsConfiguration (from @CrossOrigin):
CorsConfiguration handlerConfig = getHandlerCorsConfig();
// {allowedOrigins: ["https://handler.example.com"],
//  allowedMethods: ["GET"],
//  maxAge: 1800}

// global CorsConfiguration (from addCorsMappings):
CorsConfiguration globalConfig = getGlobalCorsConfig();
// {allowedOrigins: ["https://global.example.com"],
//  allowedMethods: ["GET", "POST"],
//  allowCredentials: true,
//  maxAge: 3600}

// Combined result (CorsConfiguration.combine()):
CorsConfiguration combined = globalConfig.combine(handlerConfig);
// Combination rules:
// allowedOrigins: HANDLER wins → ["https://handler.example.com"]
// allowedMethods: HANDLER wins → ["GET"]
//   (handler specified methods → replaces global)
// allowCredentials: GLOBAL used (handler didn't specify) → true
// maxAge: HANDLER wins → 1800
//   (handler specified maxAge → replaces global)
//
// RULE: if property explicitly set in handler annotation → handler wins
//       if property not set in handler → global value used

// CRITICAL: defaults of @CrossOrigin:
// origins default: [] (empty = not set, NOT wildcard)
// allowedHeaders default: [] (empty = not set, meaning "*" effectively)
// methods default: [] (empty = uses handler's @RequestMapping methods)
// allowCredentials default: "" (empty string = not set)
// maxAge default: -1 (not set)
```

---

### CORS with Spring Security — The Integration Point

```java
// CRITICAL: Spring Security's FilterChain runs BEFORE
// DispatcherServlet. If CORS pre-flight OPTIONS request
// reaches Spring Security filter, it may be REJECTED
// before Spring MVC's CORS processing runs.

// WRONG: Spring Security blocks pre-flight requests
@Bean
public SecurityFilterChain securityFilterChain(
        HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        );
    // PRE-FLIGHT OPTIONS /api/products:
    // → Spring Security filter: no authentication → REJECT → 401
    // → Spring MVC CORS processing NEVER REACHED
    // → Browser: CORS error
    return http.build();
}

// CORRECT: Permit CORS pre-flight in Spring Security
@Bean
public SecurityFilterChain securityFilterChain(
        HttpSecurity http) throws Exception {
    http
        .cors(Customizer.withDefaults())
        // This adds CorsFilter to Spring Security filter chain
        // CorsFilter runs BEFORE authentication checks
        // Pre-flight OPTIONS requests handled before auth
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated()
        );
    return http.build();
}

// Spring Security's CorsFilter reads from CorsConfigurationSource:
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(
        List.of("https://app.example.com"));
    config.setAllowedMethods(
        List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(
        List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source =
        new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    source.registerCorsConfiguration("/public/**", config);
    return source;
}
```

---

### CorsFilter — Servlet Filter Level CORS

```java
// CorsFilter operates at Servlet Filter level
// BEFORE DispatcherServlet — handles CORS for ALL requests
// including those not handled by Spring MVC

@Bean
public CorsFilter corsFilter() {
    UrlBasedCorsConfigurationSource source =
        new UrlBasedCorsConfigurationSource();

    CorsConfiguration config = new CorsConfiguration();
    config.applyPermitDefaultValues();
    // Sets reasonable defaults:
    // allowedOrigins: *
    // allowedMethods: GET, HEAD, POST
    // allowedHeaders: *
    // maxAge: 1800

    // Override defaults:
    config.setAllowedOrigins(
        List.of("https://app.example.com"));
    config.setAllowedMethods(
        List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
    config.addAllowedHeader("*");
    config.setAllowCredentials(true);

    source.registerCorsConfiguration("/**", config);
    return new CorsFilter(source);
}

// When to use CorsFilter vs Spring MVC CORS:
// CorsFilter: when you also have non-Spring MVC endpoints
//             when using Spring Security (must be before auth)
//             for maximum control over filter ordering
// Spring MVC CORS: for Spring MVC-only applications
//                  simpler configuration
//                  @CrossOrigin per-controller/method support
```

---

### allowedOriginPatterns — Wildcard Origins with Credentials

```java
// PROBLEM: allowedOrigins("*") + allowCredentials(true) = INVALID
// SOLUTION: allowedOriginPatterns + allowCredentials(true) = VALID

registry.addMapping("/api/**")
        .allowedOriginPatterns(
            "https://*.example.com",
            // Matches: https://app.example.com
            //          https://api.example.com
            //          https://any.example.com
            // NOT:     https://notexample.com
            //          http://app.example.com (http not https)

            "http://localhost:[*]",
            // Matches: http://localhost:3000
            //          http://localhost:8080
            // Useful for development

            "https://{app,admin}.example.com"
            // Matches: https://app.example.com
            //          https://admin.example.com
            // NOT: https://other.example.com
        )
        .allowCredentials(true)
        // allowedOriginPatterns allows this combination
        // allowedOrigins("*") would NOT allow this

        .allowedMethods("*")
        .allowedHeaders("*");

// HOW IT WORKS:
// On each request:
// → Extract Origin header value
// → Match against each pattern
// → If match: respond with Access-Control-Allow-Origin: <actual-origin>
//             (NOT the pattern — the specific origin)
// → This is correct for credentials mode (specific origin, not *)
```

---

## 2️⃣ CODE EXAMPLES

### Production CORS Configuration

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Value("${cors.allowed-origins}")
    private String[] allowedOrigins;

    @Value("${cors.allowed-origin-patterns:}")
    private String[] allowedOriginPatterns;

    @Override
    public void addCorsMappings(CorsRegistry registry) {

        // API endpoints — authenticated, credentials
        registry.addMapping("/api/**")
                .allowedOriginPatterns(allowedOriginPatterns)
                .allowedMethods("GET", "POST", "PUT",
                                "DELETE", "PATCH", "OPTIONS")
                .allowedHeaders(
                    "Authorization",
                    "Content-Type",
                    "X-Requested-With",
                    "Accept",
                    "X-CSRF-Token")
                .exposedHeaders(
                    "X-Total-Count",
                    "X-Rate-Limit",
                    "X-Rate-Limit-Remaining",
                    "Location")
                .allowCredentials(true)
                .maxAge(3600L);

        // Public API — no credentials, wildcard origins
        registry.addMapping("/public/**")
                .allowedOrigins("*")
                .allowedMethods("GET", "HEAD", "OPTIONS")
                .allowedHeaders("Accept", "Content-Type")
                .maxAge(86400L);

        // WebSocket handshake
        registry.addMapping("/ws/**")
                .allowedOriginPatterns(allowedOriginPatterns)
                .allowedMethods("GET")
                .allowCredentials(true)
                .maxAge(3600L);
    }
}

// application.properties:
// cors.allowed-origin-patterns=https://*.example.com,http://localhost:[*]
// cors.allowed-origins=https://app.example.com,https://admin.example.com
```

---

### @CrossOrigin — Fine-Grained Per-Endpoint Control

```java
@RestController
@RequestMapping("/api/v1")
// Class-level: applies to all methods
@CrossOrigin(
    originPatterns = "https://*.example.com",
    allowedHeaders = {"Authorization", "Content-Type"},
    exposedHeaders = {"X-Total-Count"},
    maxAge = 3600
)
public class ProductApiController {

    // Inherits class CORS config
    @GetMapping("/products")
    public Page<Product> getProducts(Pageable pageable) {
        return productService.findAll(pageable);
    }

    // More permissive for this specific endpoint
    @GetMapping("/products/public")
    @CrossOrigin(
        origins = "*",  // Override: allow any origin
        allowCredentials = "false",
        methods = {RequestMethod.GET}
    )
    public List<Product> getPublicProducts() {
        return productService.findPublic();
    }

    // Additional origin for admin operations
    @DeleteMapping("/products/{id}")
    @CrossOrigin(
        originPatterns = {
            "https://*.example.com",
            "https://admin.partner.com"
        },
        methods = {RequestMethod.DELETE},
        allowCredentials = "true"
    )
    public void delete(@PathVariable Long id) {
        productService.delete(id);
    }
}
```

---

### CORS with Spring Security — Complete Integration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(
            HttpSecurity http) throws Exception {
        http
            // CORS must be configured BEFORE other filters
            .cors(cors -> cors
                .configurationSource(corsConfigurationSource()))
            // Disable CSRF for REST APIs (stateless)
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session
                .sessionCreationPolicy(
                    SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.OPTIONS, "/**")
                    .permitAll()
                    // Explicitly permit pre-flight
                    // (redundant with cors() above, but explicit)
                .requestMatchers("/api/public/**")
                    .permitAll()
                .anyRequest()
                    .authenticated()
            );
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();

        config.setAllowedOriginPatterns(
            List.of("https://*.example.com",
                    "http://localhost:[*]"));
        config.setAllowedMethods(
            List.of("GET", "POST", "PUT", "DELETE",
                    "PATCH", "OPTIONS", "HEAD"));
        config.setAllowedHeaders(List.of("*"));
        config.setExposedHeaders(
            List.of("Authorization",
                    "X-Total-Count",
                    "Location"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source =
            new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

---

### Testing CORS Configuration with MockMvc

```java
@WebMvcTest(ProductController.class)
class ProductControllerCorsTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ProductService productService;

    @Test
    void shouldAllowCorsFromAllowedOrigin() throws Exception {
        mockMvc.perform(get("/api/products")
                .header("Origin",
                    "https://app.example.com"))
            .andExpect(status().isOk())
            .andExpect(header().string(
                "Access-Control-Allow-Origin",
                "https://app.example.com"));
    }

    @Test
    void shouldRejectCorsFromUnknownOrigin() throws Exception {
        mockMvc.perform(get("/api/products")
                .header("Origin",
                    "https://evil.attacker.com"))
            .andExpect(status().isForbidden());
    }

    @Test
    void shouldHandlePreflightRequest() throws Exception {
        mockMvc.perform(options("/api/products")
                .header("Origin",
                    "https://app.example.com")
                .header("Access-Control-Request-Method",
                    "DELETE")
                .header("Access-Control-Request-Headers",
                    "Authorization, Content-Type"))
            .andExpect(status().isNoContent())
            // 204 No Content for pre-flight
            .andExpect(header().string(
                "Access-Control-Allow-Origin",
                "https://app.example.com"))
            .andExpect(header().string(
                "Access-Control-Allow-Methods",
                containsString("DELETE")))
            .andExpect(header().string(
                "Access-Control-Allow-Headers",
                containsString("Authorization")))
            .andExpect(header().exists(
                "Access-Control-Max-Age"));
    }

    @Test
    void shouldExposeCustomResponseHeaders() throws Exception {
        given(productService.findAll(any()))
            .willReturn(Page.empty());

        mockMvc.perform(get("/api/products")
                .header("Origin",
                    "https://app.example.com"))
            .andExpect(header().exists(
                "Access-Control-Expose-Headers"))
            .andExpect(header().string(
                "Access-Control-Expose-Headers",
                containsString("X-Total-Count")));
    }
}
```

---

### Edge Case — Vary Header and Caching

```java
// Spring automatically adds "Vary: Origin" to CORS responses
// This is critical for proxy caching correctness

// Without Vary: Origin:
// Proxy caches: GET /api/products (from app.example.com) → response
// Next request: GET /api/products (from evil.com)
// Proxy serves CACHED response (with Allow-Origin: app.example.com)
// evil.com gets response with wrong CORS header → browser blocks
// OR: evil.com gets response with app.example.com ACAO header
//     Browser may allow it thinking server approved

// With Vary: Origin:
// Proxy knows: same URL with different Origin = different response
// Caches separately per Origin
// Correct CORS headers always served

// Spring adds this automatically in DefaultCorsProcessor:
response.addHeader(HttpHeaders.VARY, HttpHeaders.ORIGIN);
response.addHeader(HttpHeaders.VARY,
    HttpHeaders.ACCESS_CONTROL_REQUEST_METHOD);
response.addHeader(HttpHeaders.VARY,
    HttpHeaders.ACCESS_CONTROL_REQUEST_HEADERS);
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
A browser JavaScript makes `DELETE /api/products/42` with `Content-Type: application/json` from `https://app.example.com`. The server has CORS configured to allow this origin. What is the exact sequence of HTTP requests?

A) Single DELETE request directly
B) OPTIONS pre-flight → server 200 → DELETE actual request → server 200
C) GET request first → server redirects → DELETE request
D) Browser blocks the request immediately without sending anything

**Answer: B**
`DELETE` with `Content-Type: application/json` triggers pre-flight because:
- `DELETE` is not a simple method (only GET, POST, HEAD are simple)
- `Content-Type: application/json` is not a simple content type (only `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain` are simple)
Browser sends OPTIONS first. Server responds with CORS allow headers. Browser then sends actual DELETE.

---

**Q2 — Select All That Apply**
Which combinations of CORS configuration are INVALID in Spring MVC?

A) `allowedOrigins("*")` + `allowCredentials(true)`
B) `allowedOriginPatterns("https://*.example.com")` + `allowCredentials(true)`
C) `allowedOrigins("https://app.example.com")` + `allowCredentials(true)`
D) `allowedMethods("*")` + `allowCredentials(true)`
E) `allowedOrigins("*")` + `allowCredentials(false)`

**Answer: A**
A is INVALID — Spring will throw `IllegalArgumentException` because wildcard origin (`*`) cannot be combined with `allowCredentials=true` per the CORS spec (browsers reject this combination). B is valid — `allowedOriginPatterns` is the solution for wildcard patterns with credentials. C is valid — specific origin with credentials is the standard configuration. D is valid — wildcard methods is fine with credentials. E is valid — wildcard origin without credentials is fine.

---

**Q3 — MCQ**
A `@CrossOrigin` annotation on a method specifies `origins = "https://handler.example.com"`. Global CORS via `addCorsMappings()` specifies `allowedOrigins("https://global.example.com")` with `allowCredentials(true)`. What is the effective CORS config for this handler method?

A) Only global config applies — method annotation overrides only for non-configured properties
B) Only handler config applies — handler always completely overrides global
C) `origins` from handler wins; `allowCredentials` from global applies (unset in handler)
D) Config with most restrictive settings wins for each property

**Answer: C**
`CorsConfiguration.combine()` merges: handler-level wins for properties it explicitly sets; global fills in for properties not set in handler. `@CrossOrigin` sets `origins` explicitly → handler's `origins` wins. `@CrossOrigin` doesn't set `allowCredentials` (default is `""` = not set) → global's `allowCredentials=true` applies. Combined result: `origins=["https://handler.example.com"]`, `allowCredentials=true`.

---

**Q4 — Select All That Apply**
Which are TRUE about pre-flight CORS requests?

A) Pre-flight requests include the request body of the intended actual request
B) A 204 (No Content) response to pre-flight is correct
C) Pre-flight results can be cached by the browser based on `Access-Control-Max-Age`
D) Pre-flight requests are automatically sent by browsers — application code cannot prevent them
E) Spring MVC handles pre-flight by replacing the handler with `PreFlightHandler`

**Answer: B, C, D, E**
A is wrong — pre-flight requests have NO body. They only carry `Access-Control-Request-Method` and `Access-Control-Request-Headers` headers to describe the intended request. B is correct — 204 No Content is the proper response to pre-flight (no body needed). C is correct — `Access-Control-Max-Age` tells browser how long to cache the pre-flight result. D is correct — pre-flight is a browser mechanism, completely automatic. E is correct — `getCorsHandlerExecutionChain()` replaces the handler with `PreFlightHandler` for OPTIONS pre-flight requests.

---

**Q5 — True/False**
`Access-Control-Allow-Origin: *` and `Access-Control-Allow-Credentials: true` in a server response is a valid CORS configuration that browsers accept.

**Answer: False**
This combination violates the CORS specification. Browsers REJECT responses that have both `Access-Control-Allow-Origin: *` AND `Access-Control-Allow-Credentials: true`. The browser will throw a CORS error even if the server sends both headers. The correct approach: use a specific origin (`Access-Control-Allow-Origin: https://app.example.com`) when credentials are involved, or use `allowedOriginPatterns` in Spring which responds with the actual request origin instead of `*`.

---

**Q6 — MCQ**
`allowedOriginPatterns("https://*.example.com")` is configured with `allowCredentials(true)`. A request arrives from `https://app.example.com`. What value does the server send in `Access-Control-Allow-Origin`?

A) `https://*.example.com` (the pattern itself)
B) `*` (wildcard)
C) `https://app.example.com` (the actual request origin)
D) Nothing — `allowedOriginPatterns` doesn't set this header

**Answer: C**
Pattern matching works by comparing the actual request `Origin` against the pattern. If it matches, Spring responds with `Access-Control-Allow-Origin: https://app.example.com` — the ACTUAL origin from the request, not the pattern. This is why `allowedOriginPatterns` can be combined with `allowCredentials=true` — it never sends `*`, it always sends a specific origin.

---

**Q7 — MCQ**
Spring Security is configured with `http.authorizeHttpRequests(auth -> auth.anyRequest().authenticated())`. A browser sends a CORS pre-flight OPTIONS request to `/api/products`. Without explicit CORS configuration in Spring Security, what happens?

A) OPTIONS request passes through to Spring MVC — pre-flight processed normally
B) Spring Security blocks the OPTIONS request with 401 Unauthorized — pre-flight fails
C) Spring Security allows OPTIONS through automatically — it recognises pre-flight requests
D) Browser handles pre-flight internally — OPTIONS never reaches the server

**Answer: B**
Spring Security's `AuthorizationFilter` runs for ALL requests (including OPTIONS) when `anyRequest().authenticated()` is configured. The OPTIONS pre-flight request has no authentication token → Spring Security returns 401 → browser receives non-2xx response → CORS pre-flight "fails" → browser blocks the actual request. Fix: add `http.cors(Customizer.withDefaults())` which adds `CorsFilter` before authentication, or explicitly permit OPTIONS requests.

---

**Q8 — Select All That Apply**
Which response headers does `DefaultCorsProcessor` add to EVERY CORS response (not just pre-flight)?

A) `Access-Control-Allow-Origin`
B) `Access-Control-Allow-Methods`
C) `Access-Control-Allow-Headers`
D) `Access-Control-Allow-Credentials`
E) `Vary: Origin`

**Answer: A, D, E**
`Access-Control-Allow-Methods` (B) and `Access-Control-Allow-Headers` (C) are only added on PRE-FLIGHT responses (not on actual CORS requests). `Access-Control-Allow-Origin` (A) is added on both pre-flight AND actual CORS responses. `Access-Control-Allow-Credentials` (D) is added on both when `allowCredentials=true`. `Vary: Origin` (E) is added on ALL CORS responses to ensure correct proxy caching.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `allowedOrigins("*")` + `allowCredentials(true)` throws exception**
This is the most common CORS configuration mistake. It LOOKS valid but Spring throws `IllegalArgumentException: When allowCredentials is true, allowedOrigins cannot contain '*'`. The fix is `allowedOriginPatterns("*")` (which dynamically responds with the actual origin) or list specific origins. Many developers add `allowCredentials(true)` to an existing `allowedOrigins("*")` config and are confused by the runtime exception.

**Trap 2 — CORS does NOT protect servers from receiving requests**
CORS is a BROWSER mechanism. The server receives ALL requests — CORS headers just tell the browser whether JavaScript is allowed to READ the response. A CURL request or Postman request ignores CORS entirely. Security must be implemented in authentication/authorization, not CORS. Relying on CORS as a security layer (thinking it blocks server requests from unauthorized origins) is a critical security misconception.

**Trap 3 — Spring Security blocks pre-flight before Spring MVC CORS**
Spring Security filters run BEFORE `DispatcherServlet`. Without `http.cors()` configuration, Spring Security's authentication filter can reject OPTIONS pre-flight with 401 before Spring MVC's CORS processor ever runs. Developers configure CORS in Spring MVC but forget Spring Security — pre-flight fails. Always add `http.cors(Customizer.withDefaults())` or explicitly permit OPTIONS.

**Trap 4 — `@CrossOrigin` defaults are NOT "allow everything"**
`@CrossOrigin()` with no arguments does NOT mean "allow all origins." Default behavior:
- `origins`: empty → inherited from global or handler's `@RequestMapping` mapping
- `allowedHeaders`: empty → effectively `*` (all headers allowed)
- `methods`: empty → uses the handler's declared HTTP methods
- `allowCredentials`: empty string → NOT set (not enabled)
The annotation without configuration provides minimal CORS support.

**Trap 5 — Pre-flight response must be 2xx for the actual request to proceed**
If the pre-flight response is 401, 403, 404, or 500, the browser treats it as pre-flight failure and does NOT send the actual request. The entire CORS exchange fails. The pre-flight endpoint must return 200 or 204. Spring MVC's `PreFlightHandler` returns 204. If Spring Security rejects it with 401 first, the 204 never arrives.

**Trap 6 — `Vary: Origin` and proxy caching**
Without `Vary: Origin` in CORS responses, CDN/proxy caches can serve wrong CORS headers — serving cached response with one origin's `Allow-Origin` to a request from a different origin. Spring's `DefaultCorsProcessor` adds `Vary: Origin` automatically. Custom CORS implementations must do this manually.

**Trap 7 — `maxAge` has browser-imposed limits**
`Access-Control-Max-Age: 86400` (24 hours) in the server response doesn't guarantee 24-hour pre-flight caching. Chrome caps it at 2 hours (7200 seconds). Firefox caps it at 24 hours (86400 seconds). Safari has its own limits. Setting `maxAge` higher than browser limits has no effect. For performance, set to `3600` (1 hour) — respected by most browsers.

---

## 5️⃣ SUMMARY SHEET

```
CORS FUNDAMENTALS
─────────────────────────────────────────────────────
Same-Origin: scheme + host + port all match → no CORS
Cross-Origin: any differs → browser applies CORS
CORS = browser mechanism → server receives ALL requests
CORS headers = browser instructions, not security enforcement

SIMPLE vs PRE-FLIGHT
─────────────────────────────────────────────────────
Simple request (NO pre-flight):
  Methods: GET, HEAD, POST only
  Content-Type: form-urlencoded, multipart, text/plain only
  No custom headers
  → Direct request + Origin header

Pre-flight required when:
  Method: PUT, DELETE, PATCH, or custom
  Content-Type: application/json (common case)
  Custom headers: Authorization, X-Custom, etc.
  → OPTIONS request first → then actual request

PRE-FLIGHT REQUEST HEADERS
─────────────────────────────────────────────────────
Origin: https://app.example.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: Authorization, Content-Type

PRE-FLIGHT RESPONSE HEADERS (204 No Content)
─────────────────────────────────────────────────────
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 3600
Vary: Origin

ACTUAL CORS RESPONSE HEADERS
─────────────────────────────────────────────────────
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true (if configured)
Access-Control-Expose-Headers: X-Total-Count (if configured)
Vary: Origin

INVALID COMBINATION
─────────────────────────────────────────────────────
allowedOrigins("*") + allowCredentials(true) = IllegalArgumentException
FIX: allowedOriginPatterns("*") + allowCredentials(true) = VALID
     (responds with actual origin, not *)

@CROSSORIGIN vs GLOBAL CORS COMBINATION
─────────────────────────────────────────────────────
Handler annotation properties: win when explicitly set
Global properties: used when handler doesn't set them
Empty/default in @CrossOrigin: NOT considered "set" → global fills in

SPRING SECURITY INTEGRATION
─────────────────────────────────────────────────────
Problem: Security filters run BEFORE DispatcherServlet
  → Pre-flight OPTIONS → 401 before Spring MVC CORS → failure

Fix: http.cors(Customizer.withDefaults())
  → Adds CorsFilter BEFORE authentication filters
  → Pre-flight processed before auth checks

OR: permit OPTIONS explicitly:
  .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()

SPRING MVC CORS PROCESSING INTERNALS
─────────────────────────────────────────────────────
Pre-flight: handler replaced with PreFlightHandler
  → DefaultCorsProcessor.processPreFlightRequest()
  → Sets Allow-Origin, Allow-Methods, Allow-Headers, etc.
  → Returns 204 No Content

Actual CORS request: CorsInterceptor added to chain
  → Validates Origin header
  → Sets Access-Control-Allow-Origin on response
  → Origin not allowed → 403 Forbidden

Vary: Origin automatically added to ALL CORS responses

CORS CONFIGURATION APPROACHES (three options)
─────────────────────────────────────────────────────
1. @CrossOrigin on @Controller/@RequestMapping
   Finest granularity, per-endpoint

2. WebMvcConfigurer.addCorsMappings()
   Global configuration, URL pattern based

3. CorsFilter bean
   Servlet filter level, before Spring MVC
   Required with Spring Security

ALLOWEDORIGINPATTERNS
─────────────────────────────────────────────────────
"https://*.example.com" → any subdomain
"http://localhost:[*]"  → any localhost port
"https://{app,api}.example.com" → specific subdomains
Responds with actual origin (not pattern) → credentials safe

MAXAGE BROWSER LIMITS
─────────────────────────────────────────────────────
Chrome: max 2 hours (7200s)
Firefox: max 24 hours (86400s)
Recommended: 3600 (1 hour) — broadly respected

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "CORS is a browser mechanism — servers receive ALL requests regardless"
• "allowedOrigins(*) + credentials=true = IllegalArgumentException"
• "allowedOriginPatterns(*) + credentials=true = valid (responds with actual origin)"
• "Pre-flight: OPTIONS with ACRS-Method header → 204 + allow headers"
• "Spring Security blocks pre-flight — add http.cors() or permit OPTIONS"
• "@CrossOrigin explicit properties win; empty defaults use global config"
• "DefaultCorsProcessor adds Vary: Origin automatically to all CORS responses"
• "Pre-flight: PreFlightHandler replaces actual handler; actual CORS: CorsInterceptor added"
• "Simple requests: GET/HEAD/POST with standard content types — no pre-flight"
• "DELETE/PATCH/PUT + application/json = always triggers pre-flight"
```

---
