# TOPIC 3.2 — @RequestMapping: URL PATTERNS, METHODS, PARAMS, HEADERS

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What @RequestMapping Really Encapsulates

`@RequestMapping` is not just a URL mapping annotation. It is a **multi-dimensional matching specification**. Every `@RequestMapping` annotation defines up to six independent matching conditions, ALL of which must match simultaneously for the mapping to be selected. Understanding each condition independently, how they combine, and how they interact during the matching process is the core of this topic.

```
@RequestMapping matches a request when ALL of these match:

Dimension 1: URL Pattern         (path / value)
Dimension 2: HTTP Method         (method)
Dimension 3: Request Parameters  (params)
Dimension 4: Request Headers     (headers)
Dimension 5: Consumed Media Type (consumes)
Dimension 6: Produced Media Type (produces)

All six → RequestMappingInfo
ALL must match → handler selected
ANY fails → this mapping rejected, try next
```

---

### The @RequestMapping Annotation — Complete Attribute Inventory

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Mapping
@Reflective(ControllerMappingReflectiveProcessor.class)
public @interface RequestMapping {

    // Mapping name — used for URL building
    // Referenced via mvc:mapping-url tag in JSP
    String name() default "";

    // URL patterns — the primary matching dimension
    // Aliases: path() is an alias for value()
    @AliasFor("path")
    String[] value() default {};

    @AliasFor("value")
    String[] path() default {};

    // HTTP methods — GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS
    RequestMethod[] method() default {};
    // Empty = matches ALL methods

    // Request parameter conditions
    // "param"        → param must be present
    // "!param"       → param must NOT be present
    // "param=value"  → param must equal value
    // "param!=value" → param must not equal value
    String[] params() default {};

    // HTTP header conditions (same syntax as params)
    // "X-Header"         → header must be present
    // "!X-Header"        → header must NOT be present
    // "X-Header=value"   → header must equal value
    // "X-Header!=value"  → header must not equal value
    String[] headers() default {};

    // Request Content-Type condition
    // Only relevant for requests with a body (POST, PUT, PATCH)
    // "application/json" → request body must be JSON
    // "!application/json" → request body must NOT be JSON
    String[] consumes() default {};

    // Response Content-Type condition
    // Based on request's Accept header
    // "application/json" → client must accept JSON
    // "!application/json" → client must not require JSON
    String[] produces() default {};
}
```

---

### Composed Mapping Annotations — The Shortcuts

Spring 4.3 introduced composed annotations. These are `@RequestMapping` specialisations:

```java
@GetMapping     = @RequestMapping(method = RequestMethod.GET)
@PostMapping    = @RequestMapping(method = RequestMethod.POST)
@PutMapping     = @RequestMapping(method = RequestMethod.PUT)
@DeleteMapping  = @RequestMapping(method = RequestMethod.DELETE)
@PatchMapping   = @RequestMapping(method = RequestMethod.PATCH)
```

Each is a meta-annotation:

```java
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping(method = RequestMethod.GET)
public @interface GetMapping {
    @AliasFor(annotation = RequestMapping.class)
    String[] value() default {};

    @AliasFor(annotation = RequestMapping.class)
    String[] path() default {};

    @AliasFor(annotation = RequestMapping.class)
    String[] params() default {};

    @AliasFor(annotation = RequestMapping.class)
    String[] headers() default {};

    @AliasFor(annotation = RequestMapping.class)
    String[] consumes() default {};

    @AliasFor(annotation = RequestMapping.class)
    String[] produces() default {};
}
```

`@GetMapping` **does not support** the `method` attribute — it's hardcoded to `GET`. All other attributes are forwarded via `@AliasFor`.

---

### Dimension 1: URL Patterns — Deep Analysis

#### Pattern Types

```
Exact literal:      /products
                    Matches only /products exactly

Single wildcard:    /products/*
                    Matches /products/list
                    Matches /products/42
                    Does NOT match /products/list/detail (has sub-path)
                    Does NOT match /products/ (trailing slash — configurable)

Double wildcard:    /products/**
                    Matches /products/
                    Matches /products/list
                    Matches /products/list/detail
                    Matches /products/a/b/c/d

URI Template:       /products/{id}
                    Matches /products/42    → id=42
                    Matches /products/abc  → id=abc
                    Does NOT match /products/
                    Does NOT match /products/42/detail

Constrained URI:    /products/{id:\\d+}
                    Matches /products/42   → id=42
                    Does NOT match /products/abc

Extension:          /products.json        (exact — avoid in modern APIs)
                    (suffix pattern matching deprecated in Spring 5.3)
```

#### PathPattern vs AntPathMatcher

```
PathPattern (default since Spring 5.3):
  → Compiled pattern object — fast matching
  → {*variable} for capturing the rest of the path
  → {varName:[a-z]+} for regex-constrained segments
  → Strict separator matching (/ must be explicit)

AntPathMatcher (legacy):
  → String-based matching
  → Slower but more lenient
  → *.do style extensions (discouraged)
  → Configure: setPatternParser(null) in RequestMappingHandlerMapping
```

#### Multiple URL Patterns

```java
// A single mapping can match multiple URL patterns
@GetMapping({"/products", "/items", "/goods"})
public List<Product> getAll() { ... }
// Registers THREE separate mappings pointing to same HandlerMethod
// /products → this method
// /items → this method
// /goods → this method

// Array notation
@RequestMapping(
    path = {"/api/v1/products", "/api/v2/products"},
    method = RequestMethod.GET
)
public List<Product> getAllVersioned() { ... }
```

#### Class-Level + Method-Level Combining

```java
@Controller
@RequestMapping("/products")        // Class-level prefix
public class ProductController {

    @GetMapping                     // No additional path → /products
    public String list() { ... }

    @GetMapping("/{id}")            // → /products/{id}
    public String detail() { ... }

    @PostMapping                    // → /products (POST)
    public String create() { ... }

    @GetMapping("/search")          // → /products/search
    public String search() { ... }

    @RequestMapping(
        value = "/export",
        method = {GET, POST},       // → /products/export (GET or POST)
        produces = "text/csv"
    )
    public String export() { ... }
}
```

**Combining rules:**
- URL: class prefix + method path (concatenated with no separator if method path starts with `/`)
- Method: intersection (if both specify methods) OR method-level only (if class has none)
- Params, Headers: union (both sets must be satisfied)
- Consumes, Produces: method-level OVERRIDES class-level (not merged)

---

### Dimension 2: HTTP Methods — Deep Analysis

```java
// Explicit method specification
@RequestMapping(value="/products", method=RequestMethod.GET)
@RequestMapping(value="/products", method=RequestMethod.POST)
@RequestMapping(value="/products", method={GET, POST})  // multiple

// No method → matches ALL HTTP methods
@RequestMapping("/products")  // GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS

// HEAD is handled specially:
// @GetMapping implicitly also handles HEAD (without body)
// Spring calls doGet() and strips the body for HEAD
```

**HTTP Method to Handler:**

```
GET     → Retrieve resource
POST    → Create resource or submit data
PUT     → Replace entire resource
PATCH   → Partial update
DELETE  → Delete resource
HEAD    → GET without body (auto-handled for @GetMapping)
OPTIONS → Which methods supported? (auto-handled by Spring MVC)
TRACE   → disabled by default (dispatchTraceRequest=false)
```

**`handleNoMatch()` for wrong method:**
```
URL matches, method doesn't → HttpRequestMethodNotSupportedException
→ DefaultHandlerExceptionResolver → HTTP 405 Method Not Allowed
→ Response includes Allow header: "GET, HEAD"
```

---

### Dimension 3: Request Parameters — Complete Syntax

```java
// Syntax: String[] params
// Each string is ONE condition — all must be met

// 1. Parameter must be PRESENT (any value)
@GetMapping(value="/products", params="featured")
// GET /products?featured → MATCH
// GET /products?featured=true → MATCH
// GET /products?featured=false → MATCH (any value counts)
// GET /products → NO MATCH

// 2. Parameter must be ABSENT
@GetMapping(value="/products", params="!featured")
// GET /products → MATCH
// GET /products?featured → NO MATCH

// 3. Parameter must have SPECIFIC VALUE
@GetMapping(value="/products", params="type=premium")
// GET /products?type=premium → MATCH
// GET /products?type=standard → NO MATCH
// GET /products → NO MATCH

// 4. Parameter must NOT have specific value
@GetMapping(value="/products", params="type!=deleted")
// GET /products?type=active → MATCH
// GET /products?type=deleted → NO MATCH
// GET /products → MATCH (param absent = not equal)

// 5. Multiple conditions — ALL must match
@GetMapping(value="/products",
            params={"active=true", "!deleted", "type"})
// GET /products?active=true&type=premium → MATCH
// GET /products?active=true&type=premium&deleted=1 → NO MATCH (deleted present)
// GET /products?active=false&type=premium → NO MATCH (active!=true)
```

**When params mismatch is detected:**
```
URL matches, params condition fails
→ handleNoMatch() detects hasParamsMismatch()
→ UnsatisfiedServletRequestParameterException
→ DefaultHandlerExceptionResolver → HTTP 400 Bad Request
```

---

### Dimension 4: Headers — Complete Syntax

Same syntax as params but applied to HTTP request headers:

```java
// Header must be PRESENT
@GetMapping(value="/products", headers="X-Custom-Header")
// GET /products + X-Custom-Header: anything → MATCH
// GET /products (no header) → NO MATCH

// Header must be ABSENT
@GetMapping(value="/products", headers="!X-Custom-Header")

// Header must have SPECIFIC VALUE
@GetMapping(value="/products", headers="X-API-Version=2")
// GET /products + X-API-Version: 2 → MATCH
// GET /products + X-API-Version: 1 → NO MATCH

// Content-Type and Accept via headers (discouraged — use consumes/produces instead)
@GetMapping(value="/products",
            headers="Content-Type=application/json")
// WORKS but produces and consumes are better for content type conditions
```

**Important:** `Content-Type` header condition specified via `headers` behaves differently from `consumes`. The `consumes` attribute understands media type wildcards and negotiation. `headers="Content-Type=..."` is exact string matching.

---

### Dimension 5: Consumes — Request Body Media Type

```java
// Only relevant for requests with bodies (POST, PUT, PATCH)

// Must receive JSON
@PostMapping(value="/products",
             consumes="application/json")
// POST /products + Content-Type: application/json → MATCH
// POST /products + Content-Type: application/xml → NO MATCH (415)
// POST /products + no Content-Type → NO MATCH (415)

// Multiple accepted content types
@PostMapping(value="/products",
             consumes={"application/json",
                       "application/x-www-form-urlencoded"})

// Wildcard — any subtype of application
@PostMapping(value="/products",
             consumes="application/*")

// Negation — NOT this type
@PostMapping(value="/products",
             consumes="!text/plain")
// Accepts anything EXCEPT text/plain

// Using MediaType constants
@PostMapping(value="/products",
             consumes=MediaType.APPLICATION_JSON_VALUE)
```

**When consumes mismatch:**
```
URL matches, consumes condition fails
→ handleNoMatch() detects hasConsumesMismatch()
→ HttpMediaTypeNotSupportedException
→ DefaultHandlerExceptionResolver → HTTP 415 Unsupported Media Type
```

---

### Dimension 6: Produces — Response Accept Negotiation

```java
// Client must accept this content type (from Accept header)

// Only produce JSON
@GetMapping(value="/products",
            produces="application/json")
// GET /products + Accept: application/json → MATCH
// GET /products + Accept: */* → MATCH (wildcard accepts anything)
// GET /products + Accept: text/html → NO MATCH (406)

// Multiple producible types — Spring selects best match
@GetMapping(value="/products",
            produces={"application/json", "application/xml"})
// Accept: application/json → produces JSON
// Accept: application/xml → produces XML
// Accept: */* → first listed type (application/json) wins

// Produces with charset
@GetMapping(value="/text",
            produces="text/plain;charset=UTF-8")

// Negation — don't produce this type
@GetMapping(value="/products",
            produces="!text/html")

// MediaType constant
@GetMapping(value="/products",
            produces=MediaType.APPLICATION_JSON_VALUE)
```

**When produces mismatch:**
```
URL matches, produces condition fails
→ handleNoMatch() detects hasProducesMismatch()
→ HttpMediaTypeNotAcceptableException
→ DefaultHandlerExceptionResolver → HTTP 406 Not Acceptable
```

---

### Specificity and Conflict Resolution

When multiple mappings match a request, Spring selects the MOST SPECIFIC:

```
SPECIFICITY HIERARCHY (most to least specific):

1. Exact literal path > URI template > wildcard
   /products/featured > /products/{id} > /products/*

2. More HTTP methods specified > fewer
   GET > GET+POST

3. Params condition specified > no params
   params="type=premium" > no params condition

4. Headers condition specified > no headers
   headers="X-Version=2" > no headers condition

5. Consumes specified > no consumes
   consumes="application/json" > no consumes

6. Produces specified > no produces
   produces="application/json" > no produces

CONFLICT DETECTION:
Two mappings with IDENTICAL specificity for same request
→ IllegalStateException at STARTUP (for exact duplicates)
→ IllegalStateException at RUNTIME (for runtime-ambiguous equal-specificity)

VALID DIFFERENTIATION:
Same URL, different produces → content negotiation selects
@GetMapping(value="/products", produces="application/json")
@GetMapping(value="/products", produces="application/xml")
→ Both valid, no conflict — produces differentiates
```

---

### Class-Level vs Method-Level Combination Rules — Precise

```java
@Controller
@RequestMapping(
    value = "/api",
    method = {GET, POST},              // Class: GET or POST
    params = "version=2",              // Class: version param
    headers = "X-Client=mobile",       // Class: mobile client
    consumes = "application/json",     // Class: JSON input
    produces = "application/json"      // Class: JSON output
)
public class ApiController {

    @PostMapping(
        value = "/products",
        // method: NOT specified → USES CLASS-LEVEL {GET, POST}
        params = "category=electronics", // ADDED to class params
        headers = "X-Feature=search",    // ADDED to class headers
        consumes = "application/xml",    // OVERRIDES class consumes
        produces = "application/xml"     // OVERRIDES class produces
    )
    public Product create(@RequestBody Product product) { ... }
    // Effective mapping:
    // URL:      /api/products
    // Method:   GET or POST (from class)
    // Params:   version=2 AND category=electronics (BOTH required)
    // Headers:  X-Client=mobile AND X-Feature=search (BOTH required)
    // Consumes: application/xml (method OVERRIDES class)
    // Produces: application/xml (method OVERRIDES class)
}
```

**Combination rules summarised:**
```
URL:      class prefix PREPENDED to method path (concatenated)
Method:   if method has methods → method-level used
          if method has no methods → class-level used
Params:   BOTH class and method conditions must be satisfied (AND logic)
Headers:  BOTH class and method conditions must be satisfied (AND logic)
Consumes: method-level OVERRIDES class-level (if method specifies any)
Produces: method-level OVERRIDES class-level (if method specifies any)
```

---

### RequestMappingInfo Internal Construction

```java
// getMappingForMethod() creates RequestMappingInfo from @RequestMapping
private RequestMappingInfo createRequestMappingInfo(
        AnnotatedElement element) {

    RequestMapping requestMapping =
        AnnotatedElementUtils.findMergedAnnotation(
            element, RequestMapping.class);
    // findMergedAnnotation: discovers @RequestMapping even if
    // used as meta-annotation (e.g., @GetMapping)
    // Returns null if no @RequestMapping found

    RequestCondition<?> customCondition =
        (element instanceof Class ?
            getCustomTypeCondition((Class<?>) element) :
            getCustomMethodCondition((Method) element));

    return (requestMapping != null ?
        createRequestMappingInfo(requestMapping, customCondition) :
        null);
}

// Build the RequestMappingInfo
protected RequestMappingInfo createRequestMappingInfo(
        RequestMapping requestMapping,
        @Nullable RequestCondition<?> customCondition) {

    return RequestMappingInfo
        .paths(resolveEmbeddedValuesInPatterns(
            requestMapping.path()))
        // URL patterns — supports ${} placeholders from properties
        .methods(requestMapping.method())
        .params(requestMapping.params())
        .headers(requestMapping.headers())
        .consumes(requestMapping.consumes())
        .produces(requestMapping.produces())
        .mappingName(requestMapping.name())
        .customCondition(customCondition)
        .options(this.config)
        // config controls PathPattern vs AntPathMatcher
        .build();
}
```

**`resolveEmbeddedValuesInPatterns()` enables property placeholders in URLs:**

```java
@GetMapping("${api.prefix}/products")
// api.prefix=api/v1 in application.properties
// → registers mapping for /api/v1/products
// Resolved at startup — not per-request
```

---

### @RequestMapping on @Configuration — Registration

`@RequestMapping` can also appear on `@Configuration` class methods, but this is unusual. The normal mechanism:

```
@Configuration class scanned by ConfigurationClassPostProcessor
→ @Bean methods registered as BeanDefinition
→ @RequestMapping on @Configuration has no effect (not a @Controller)

@Controller class scanned by ClassPathBeanDefinitionScanner
→ Registered as BeanDefinition
→ Then RequestMappingHandlerMapping.isHandler() checks it
→ @RequestMapping methods discovered and registered
```

---

### URL Pattern Edge Cases — Tested Scenarios

```java
@RestController
@RequestMapping("/api")
public class EdgeCaseController {

    // Trailing slash — configurable behaviour
    // By default Spring 5: /products and /products/ are treated SAME
    // Spring 6: trailing slash match DISABLED by default
    @GetMapping("/products")
    public String products() { return "products"; }
    // Spring 5: GET /api/products AND GET /api/products/ → MATCH
    // Spring 6: GET /api/products/ → 404 (trailing slash strict)

    // {*variable} — captures ENTIRE remaining path (PathPattern only)
    @GetMapping("/files/{*path}")
    public String files(@PathVariable String path) { return path; }
    // GET /api/files/a/b/c.txt → path = "/a/b/c.txt"
    // Includes the leading slash

    // Matrix variables — requires semicolon in URL
    // Must enable via UrlPathHelper.setRemoveSemicolonContent(false)
    @GetMapping("/users/{userId}")
    public String userWithMatrix(
            @PathVariable String userId,
            @MatrixVariable String role) { return userId + ":" + role; }
    // GET /api/users/42;role=admin → userId=42, role=admin

    // Multiple variables in one segment
    @GetMapping("/locations/{latitude},{longitude}")
    public String location(
            @PathVariable double latitude,
            @PathVariable double longitude) { ... }
    // GET /api/locations/51.5,-0.1 → lat=51.5, lon=-0.1

    // Optional path segment via two mappings
    @GetMapping({"/catalog", "/catalog/{category}"})
    public String catalog(
            @PathVariable Optional<String> category) { ... }
    // GET /api/catalog → category = empty
    // GET /api/catalog/electronics → category = "electronics"
}
```

---

### Content Negotiation vs Consumes/Produces

```
CONSUMES affects INBOUND request handling:
  consumes="application/json"
  → Checks request Content-Type header
  → Controls WHICH mapping handles the request
  → No effect on response format

PRODUCES affects OUTBOUND response handling:
  produces="application/json"
  → Checks request Accept header
  → Controls WHICH mapping handles the request
  → ALSO sets Content-Type on response
  → Linked to HttpMessageConverter selection

Both can appear simultaneously:
  @PostMapping(value="/transform",
               consumes="application/xml",   // Receives XML
               produces="application/json")  // Returns JSON
  // Only handles POST with XML body
  // Returns JSON response
```

---

### @RequestMapping on Interface vs Implementation

```java
// Mapping on interface method
public interface ProductApi {
    @GetMapping("/products")
    List<Product> getAll();
}

// Implementation inherits the mapping IF it also has @Controller
@RestController
public class ProductController implements ProductApi {
    @Override
    public List<Product> getAll() {
        return productService.findAll();
    }
    // @GetMapping("/products") inherited from interface
    // RequestMappingHandlerMapping resolves this via
    // AnnotatedElementUtils which follows interface methods
}

// WARNING: In Spring 5.3+, class-level @RequestMapping on interface
// is deprecated as proxy behaviour can be inconsistent with interfaces
```

---

## 2️⃣ CODE EXAMPLES

### Complete Mapping Specification — All Dimensions

```java
@RestController
@RequestMapping(
    value = "/api/v2",
    produces = MediaType.APPLICATION_JSON_VALUE
)
public class CompleteController {

    // All six dimensions explicit
    @RequestMapping(
        value    = "/products/{id}",
        method   = RequestMethod.GET,
        params   = "!deleted",
        headers  = "X-API-Key",
        consumes = MediaType.ALL_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    public Product getProduct(
            @PathVariable Long id) {
        // Effective URL: /api/v2/products/{id}
        // Method: GET
        // Params: "deleted" must NOT be present
        // Headers: X-API-Key must be present (any value)
        // Consumes: any Content-Type (or none)
        // Produces: application/json (overrides class-level redundantly)
        return productService.findById(id)
            .orElseThrow(() ->
                new ResponseStatusException(
                    HttpStatus.NOT_FOUND));
    }

    // Differentiated by produces — content negotiation selects
    @GetMapping(
        value    = "/report",
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    public Map<String, Object> reportJson() {
        return reportService.generateMap();
    }

    @GetMapping(
        value    = "/report",
        produces = "text/csv"
    )
    public String reportCsv() {
        return reportService.generateCsv();
    }

    // Differentiated by params
    @GetMapping(value="/products", params="type=featured")
    public List<Product> getFeatured() {
        return productService.findFeatured();
    }

    @GetMapping(value="/products", params="type=new")
    public List<Product> getNew() {
        return productService.findNew();
    }

    @GetMapping(value="/products")  // Catch-all — most general
    public List<Product> getAll() {
        return productService.findAll();
    }
}
```

---

### Property Placeholder in URL Pattern

```java
// application.properties:
// api.base.path=/api/v1
// api.version=v1

@RestController
public class PropertyPathController {

    // URL resolved from properties at startup
    @GetMapping("${api.base.path}/products")
    public List<Product> getProducts() {
        return productService.findAll();
    }
    // Effective URL: /api/v1/products

    // Combine with class-level
    @Controller
    @RequestMapping("${api.base.path}")
    public class ProductController {

        @GetMapping("/products")  // → /api/v1/products
        public String list() { ... }
    }
}
```

---

### Handling Multiple HTTP Methods with Shared Logic

```java
@RestController
@RequestMapping("/api/documents")
public class DocumentController {

    // Handles both GET and HEAD
    // HEAD: same processing, but Spring strips response body
    @RequestMapping(
        value = "/{id}",
        method = {RequestMethod.GET, RequestMethod.HEAD}
    )
    public ResponseEntity<Document> getDocument(
            @PathVariable Long id) {
        Document doc = documentService.findById(id)
            .orElseThrow(() ->
                new ResponseStatusException(
                    HttpStatus.NOT_FOUND));

        return ResponseEntity.ok()
            .lastModified(doc.getLastModified())
            .eTag(doc.getEtag())
            .body(doc);
        // HEAD request: headers set, body stripped by Spring
    }

    // OPTIONS handled automatically by Spring
    // Returns Allow: GET, HEAD, POST, OPTIONS
    // But can be overridden:
    @RequestMapping(
        value = "/{id}",
        method = RequestMethod.OPTIONS
    )
    public ResponseEntity<Void> options() {
        return ResponseEntity.ok()
            .allow(HttpMethod.GET, HttpMethod.POST,
                   HttpMethod.PUT, HttpMethod.DELETE)
            .build();
    }
}
```

---

### Params and Headers for API Versioning

```java
@RestController
@RequestMapping("/api/products")
public class VersionedController {

    // Version via request parameter
    @GetMapping(params = "version=1")
    public List<ProductV1> getAllV1() {
        return productService.findAllAsV1();
    }
    // GET /api/products?version=1

    @GetMapping(params = "version=2")
    public List<ProductV2> getAllV2() {
        return productService.findAllAsV2();
    }
    // GET /api/products?version=2

    @GetMapping  // No version param — default
    public List<ProductV2> getAllDefault() {
        return productService.findAllAsV2();
    }
    // GET /api/products (most general — matched last)

    // Version via Accept header (recommended for REST)
    @GetMapping(
        headers = "Accept=application/vnd.api.v1+json",
        produces = "application/vnd.api.v1+json"
    )
    public List<ProductV1> getAllV1Header() {
        return productService.findAllAsV1();
    }

    @GetMapping(
        headers = "Accept=application/vnd.api.v2+json",
        produces = "application/vnd.api.v2+json"
    )
    public List<ProductV2> getAllV2Header() {
        return productService.findAllAsV2();
    }
}
```

---

### Complex URL Pattern Combinations

```java
@RestController
public class PatternDemoController {

    // {*path} — captures everything including slashes
    @GetMapping("/static/{*resourcePath}")
    public ResponseEntity<Resource> staticResource(
            @PathVariable String resourcePath) {
        // GET /static/css/main.css → resourcePath="/css/main.css"
        // GET /static/js/app/bundle.js → resourcePath="/js/app/bundle.js"
        Resource resource = resourceLoader.getResource(
            "classpath:static" + resourcePath);
        return ResponseEntity.ok(resource);
    }

    // Regex constraint — digits only
    @GetMapping("/products/{id:\\d+}")
    public Product numericId(@PathVariable Long id) {
        return productService.findById(id)
            .orElseThrow();
    }

    // Regex constraint — UUID format
    @GetMapping("/sessions/{sessionId:[a-f0-9\\-]{36}}")
    public Session sessionById(
            @PathVariable String sessionId) {
        return sessionService.findById(sessionId)
            .orElseThrow();
    }

    // Comma-separated variables in ONE path segment
    @GetMapping("/range/{min:\\d+},{max:\\d+}")
    public List<Product> byPriceRange(
            @PathVariable double min,
            @PathVariable double max) {
        // GET /range/10,100 → min=10, max=100
        return productService.findByPriceRange(min, max);
    }

    // Multiple optional paths via array
    @GetMapping({
        "/search",
        "/search/{category}",
        "/search/{category}/{subcategory}"
    })
    public List<Product> search(
            @PathVariable Optional<String> category,
            @PathVariable Optional<String> subcategory,
            @RequestParam(defaultValue = "") String q) {
        return productService.search(
            category.orElse(null),
            subcategory.orElse(null), q);
    }
}
```

---

### Edge Case — Ambiguous Mappings

```java
@RestController
public class AmbiguityDemo {

    // CASE 1: Identical URL + method → STARTUP error
    @GetMapping("/products")
    public List<Product> getAll1() { ... }

    @GetMapping("/products")  // DUPLICATE
    public List<Product> getAll2() { ... }
    // → IllegalStateException at startup:
    // "Ambiguous mapping. Cannot map 'ambiguityDemo' method
    //  getAll2() to {GET /products}: There is already
    //  'ambiguityDemo' bean method getAll1()"

    // CASE 2: {id} vs {name} — IDENTICAL patterns → STARTUP error
    @GetMapping("/products/{id}")
    public Product byId(@PathVariable Long id) { ... }

    @GetMapping("/products/{name}")  // SAME pattern as /{id}
    public Product byName(@PathVariable String name) { ... }
    // → Same startup error

    // CASE 3: Different produces → VALID, no error
    @GetMapping(value="/products", produces="application/json")
    public List<Product> asJson() { ... }

    @GetMapping(value="/products", produces="application/xml")
    public List<Product> asXml() { ... }
    // → Valid — produces differentiates them

    // CASE 4: Literal vs template — VALID, literal wins
    @GetMapping("/products/featured")
    public List<Product> featured() { ... }

    @GetMapping("/products/{id}")
    public Product byId(@PathVariable Long id) { ... }
    // → Valid — literal /products/featured more specific
    // GET /products/featured → featured()
    // GET /products/42 → byId()
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
A mapping is defined as:
```java
@PostMapping(value="/products",
             params="type=premium",
             headers="X-Region",
             consumes="application/json",
             produces="application/xml")
```
A `POST /products` arrives with: `Content-Type: application/json`, `Accept: application/xml`, `X-Region: EU`, and a query param `type=standard`. What HTTP status is returned?

A) 200 OK — close enough match
B) 400 Bad Request — params condition fails
C) 406 Not Acceptable — produces mismatch
D) 415 Unsupported Media Type — consumes mismatch

**Answer: B**
`params="type=premium"` requires `type=premium` exactly. The request has `type=standard`. `handleNoMatch()` detects `hasParamsMismatch()` → throws `UnsatisfiedServletRequestParameterException` → HTTP 400. The other conditions (consumes=json ✓, produces=xml ✓, X-Region ✓) match, but params fails first in the mismatch detection order.

---

**Q2 — Select All That Apply**
Which of the following correctly describe how class-level and method-level `@RequestMapping` attributes are combined?

A) URL paths are concatenated — class prefix + method path
B) HTTP methods are intersected — only methods in BOTH are allowed
C) `params` conditions from class and method are both required (AND logic)
D) `consumes` from method-level OVERRIDES class-level entirely
E) `headers` conditions from class and method are both required (AND logic)

**Answer: A, C, D, E**
B is wrong — if the method specifies methods, they are used; class-level methods are not intersected. If method specifies no methods, class-level is inherited. There is no intersection. A (concatenation), C (params AND), D (consumes overrides), E (headers AND) are all correct.

---

**Q3 — Code Prediction**
```java
@RestController
@RequestMapping("/api")
public class TestController {

    @GetMapping("/items/export")
    public String export() { return "export"; }

    @GetMapping("/items/{id}")
    public String byId(@PathVariable String id) {
        return "id=" + id;
    }

    @GetMapping("/items/*")
    public String wildcard() { return "wildcard"; }
}
```
Match each request to the correct handler:
1. `GET /api/items/export`
2. `GET /api/items/42`
3. `GET /api/items/42/details`

A) export, byId, wildcard
B) export, byId, 404
C) export, wildcard, 404
D) export, byId, byId

**Answer: B**
1. `/api/items/export`: literal `export` matches — `export()` wins over `{id}` (literal > template)
2. `/api/items/42`: both `{id}` (template) and `*` (wildcard) match — `{id}` is MORE SPECIFIC than `/*` → `byId()`
3. `/api/items/42/details`: `*` only matches single segment (no slashes) → no match → 404. `{id}` only matches single segment → no match.

---

**Q4 — MCQ**
`@GetMapping(produces = "application/json")` is on a method. A request arrives with `Accept: */*`. What happens?

A) HTTP 406 — `*/*` does not match `application/json`
B) The mapping MATCHES — `*/*` is compatible with `application/json`
C) Spring selects a different mapping without `produces` constraint
D) Spring returns `application/json` only if Jackson is on classpath

**Answer: B**
The `produces` condition uses `MediaType.isCompatibleWith()`, not exact equality. `*/*` means "I accept anything." `application/json` IS compatible with `*/*`. The mapping matches. Spring will produce `application/json` (the mapping's declared type). Accept: `*/*` satisfies any `produces` constraint.

---

**Q5 — True/False**
Two mappings `@GetMapping(value="/products", produces="application/json")` and `@GetMapping(value="/products", produces="application/xml")` on different methods in the same controller cause an `IllegalStateException` at startup.

**Answer: False**
Different `produces` conditions differentiate two otherwise identical mappings. `RequestMappingHandlerMapping` detects that these are NOT ambiguous because `ProducesRequestCondition` comparison shows they are different. Spring's ambiguity check compares specificity — same URL + method but DIFFERENT produces = valid differentiation. No startup error. Content negotiation selects the appropriate one per request.

---

**Q6 — MCQ**
A `@RequestMapping` method has `params = {"a", "b=2", "!c"}`. Which request matches?

A) `?a=1&b=2&c=3`
B) `?a=5&b=2`
C) `?b=2`
D) `?a=x&b=3`

**Answer: B**
`params={"a", "b=2", "!c"}` requires:
- `a` must be PRESENT (any value) ✓ `a=5` has `a`
- `b=2` must equal 2 ✓ `b=2` matches
- `!c` means `c` must be ABSENT ✓ no `c` in request

A fails: `c=3` is present (violates `!c`).
C fails: `a` is absent (violates `a`).
D fails: `b=3` doesn't equal 2 (violates `b=2`).

---

**Q7 — Select All That Apply**
Which features does `@GetMapping` support that `@RequestMapping(method=GET)` does NOT?

A) URL path patterns
B) `consumes` attribute
C) `produces` attribute
D) `name` attribute
E) None — `@GetMapping` is exactly equivalent to `@RequestMapping(method=GET)` with the same attributes

**Answer: E**
`@GetMapping` is a composed annotation that delegates ALL its attributes to `@RequestMapping` via `@AliasFor`. It provides no additional features beyond fixing the method to GET and providing a more concise syntax. All attributes (`value`, `path`, `params`, `headers`, `consumes`, `produces`, `name`) are available on both.

---

**Q8 — Scenario**
A controller has these two mappings:
```java
@GetMapping(value="/data", params="format=json", produces="application/json")
public String jsonFormat() { return "{...}"; }

@GetMapping(value="/data", produces="application/json")
public String defaultFormat() { return "{...}"; }
```
`GET /data?format=json` with `Accept: application/json` arrives. Which method is called?

A) `jsonFormat()` — more specific (has params condition)
B) `defaultFormat()` — registered first
C) `IllegalStateException` — ambiguous
D) Neither — 406 Not Acceptable

**Answer: A**
Both mappings match this request:
- `jsonFormat()`: URL ✓, GET ✓, params `format=json` ✓, produces `application/json` ✓
- `defaultFormat()`: URL ✓, GET ✓, no params condition (more general), produces ✓

Specificity comparison: `jsonFormat()` has a params condition specified — this is MORE SPECIFIC than having no params condition. Spring selects `jsonFormat()`. This is the intended use of params for disambiguation.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — Empty `method` attribute means ALL methods**
`@RequestMapping("/products")` with no `method` attribute matches GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS. Many assume "no method" means GET. It means ALL. A security trap: a method intended for GET only that uses `@RequestMapping("/sensitive")` can be called via POST, potentially bypassing CSRF protection.

**Trap 2 — params condition with absent parameter**
`params="type"` requires the `type` parameter to be PRESENT. `params="!type"` requires it to be ABSENT. `params="type=premium"` requires both presence AND value. A common trap: `params="type"` does NOT mean `type` must equal `"type"` — it means the parameter named `type` must exist with any value.

**Trap 3 — consumes vs headers="Content-Type=..."**
`consumes="application/json"` understands media type semantics — `application/*` wildcards work, charset parameters are handled. `headers="Content-Type=application/json"` is pure string matching — `Content-Type: application/json; charset=UTF-8` would NOT match because the header value differs. Always use `consumes` for Content-Type conditions.

**Trap 4 — produces negation is not intuitive**
`produces="!text/html"` means "client must NOT require text/html". If `Accept: text/html` → NO MATCH. If `Accept: application/json` → MATCH. If `Accept: */*` → tricky — `*/*` is compatible with `text/html`, so the negation condition might not behave as expected. The negation checks whether the NEGOTIATED type is the negated type.

**Trap 5 — Class-level method vs method-level method interaction**
If class has `method={GET, POST}` and method has `method=GET`, the effective mapping is GET (method-level overrides). If class has `method={GET, POST}` and method has NO method, the effective mapping is `{GET, POST}` (class-level inherited). If class has NO method and method has `method=POST`, effective is POST. The rule: method-level wins if specified, class-level inherited if not.

**Trap 6 — `{id}` and `{name}` are identical patterns**
URI template variable NAMES don't matter for pattern matching. `/products/{id}` and `/products/{name}` are the SAME URL pattern. Registering both → `IllegalStateException` at startup. Only the REGEX constraint differentiates templates: `/products/{id:\\d+}` vs `/products/{slug:[a-z-]+}` are genuinely different.

**Trap 7 — Trailing slash in Spring 6**
Spring 5 and before: `/products` and `/products/` treated as same URL (trailing slash match enabled). Spring 6: trailing slash match is DISABLED by default. `/products/` returns 404 if only `/products` is mapped. This breaks existing applications when upgrading. Re-enable with `configurer.setUseTrailingSlashMatch(true)` in `configurePathMatch()`.

---

## 5️⃣ SUMMARY SHEET

```
@REQUESTMAPPING — SIX MATCHING DIMENSIONS
─────────────────────────────────────────────────────
Dimension    Attribute   Fail = HTTP Status
─────────────────────────────────────────────────────
URL Pattern  value/path  404 (no URL match)
HTTP Method  method      405 Method Not Allowed
Params       params      400 Bad Request
Headers      headers     400 Bad Request (mapped to this)
Consumes     consumes    415 Unsupported Media Type
Produces     produces    406 Not Acceptable

ALL six must match → mapping selected
ANY fails → mapping rejected → try next mapping

URL PATTERN TYPES
─────────────────────────────────────────────────────
/products           Exact literal (most specific)
/products/{id}      URI template variable
/products/{id:\d+}  Regex-constrained template
/products/*         Single segment wildcard
/products/**        Any path wildcard (least specific)
/products/{*path}   Capture-all variable (PathPattern only)
{lat},{lng}         Two variables in one segment

SPECIFICITY ORDER
─────────────────────────────────────────────────────
/products/featured > /products/{id} > /products/* > /products/**
With method        > without method
With params        > without params
With headers       > without headers
With consumes      > without consumes
With produces      > without produces

CLASS + METHOD COMBINATION RULES
─────────────────────────────────────────────────────
URL:      concatenated (class prefix + method path)
Method:   method-level if specified, else class-level (NOT intersected)
Params:   BOTH required — AND logic
Headers:  BOTH required — AND logic
Consumes: method OVERRIDES class (if method specifies any)
Produces: method OVERRIDES class (if method specifies any)

PARAMS SYNTAX
─────────────────────────────────────────────────────
"param"        param must be present (any value)
"!param"       param must be absent
"param=value"  param must equal value
"param!=value" param must not equal value (or absent)
Multiple       ALL conditions must be met (AND)

HEADERS SYNTAX — SAME AS PARAMS
─────────────────────────────────────────────────────
"X-Header"         header must be present
"!X-Header"        header must be absent
"X-Header=value"   header must equal value
"X-Header!=value"  header must not equal value

CONSUMES vs PRODUCES
─────────────────────────────────────────────────────
consumes = checks request Content-Type
  → controls WHICH mapping handles the request
  → supports wildcards: application/*
  → 415 on mismatch

produces = checks request Accept header
  → controls WHICH mapping handles the request
  → ALSO sets Content-Type on response
  → supports wildcards: */* always matches
  → 406 on mismatch

PROPERTY PLACEHOLDER IN URL
─────────────────────────────────────────────────────
@GetMapping("${api.prefix}/products")
Resolved at startup via EmbeddedValueResolver
Value comes from application.properties / environment

COMPOSED ANNOTATIONS
─────────────────────────────────────────────────────
@GetMapping    = @RequestMapping(method=GET)
@PostMapping   = @RequestMapping(method=POST)
@PutMapping    = @RequestMapping(method=PUT)
@DeleteMapping = @RequestMapping(method=DELETE)
@PatchMapping  = @RequestMapping(method=PATCH)
All delegate attributes via @AliasFor — fully equivalent

TRAILING SLASH
─────────────────────────────────────────────────────
Spring 5: /products == /products/ (trailing slash matching on)
Spring 6: /products != /products/ (trailing slash matching OFF by default)

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "@RequestMapping with no method matches ALL HTTP methods"
• "{id} and {name} are identical URL patterns — ambiguous at startup"
• "Literal path beats URI template — /products/featured > /products/{id}"
• "consumes checks Content-Type (inbound); produces checks Accept (outbound)"
• "params/headers combine with AND logic; consumes/produces method-level overrides"
• "Spring 6 trailing slash match is disabled by default — breaking change from 5"
• "produces negation (!text/html) rarely behaves as intuitively expected"
• "params='type' means parameter present (any value), not equals string 'type'"
• "@GetMapping is @RequestMapping(method=GET) — no additional features"
• "Property placeholders in URL: @GetMapping('${prefix}/path') resolved at startup"
```

---
