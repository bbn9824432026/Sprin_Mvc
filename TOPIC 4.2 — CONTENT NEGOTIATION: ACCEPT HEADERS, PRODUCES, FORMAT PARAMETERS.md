# TOPIC 4.2 — CONTENT NEGOTIATION: ACCEPT HEADERS, PRODUCES, FORMAT PARAMETERS

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What Content Negotiation Really Is

Content negotiation is the process by which Spring MVC determines **what format to send the response in** — JSON, XML, HTML, CSV, or anything else. The client expresses what it can accept; the server determines what it can produce; Spring MVC finds the best match. This sounds simple but the internals involve a sophisticated negotiation algorithm with multiple strategies, fallbacks, priority rules, and edge cases that are heavily tested.

---

### The Two-Phase Content Negotiation Model

```
PHASE 1: WHAT DOES THE CLIENT WANT?
(ContentNegotiationManager determines acceptable media types)
        │
        ├── Strategy 1: Header-based (Accept header)
        │     Accept: application/json, text/html;q=0.9, */*;q=0.8
        │
        ├── Strategy 2: Parameter-based (?format=json)
        │     /products?format=json
        │
        ├── Strategy 3: Path extension-based (deprecated)
        │     /products.json (DEPRECATED in Spring 5.3)
        │
        └── Returns: ordered List<MediaType> of acceptable types

PHASE 2: WHAT CAN THE SERVER PRODUCE?
(HandlerAdapter determines producible media types)
        │
        ├── Source 1: @RequestMapping(produces=...) annotation
        │     Restricts what this endpoint CAN produce
        │
        ├── Source 2: HttpMessageConverter supported types
        │     What converters are registered + what they support
        │
        └── Returns: List<MediaType> of producible types

MATCHING:
  For each acceptable type (in client preference order):
    For each producible type:
      If compatible → MATCH → select HttpMessageConverter
  No match → HttpMediaTypeNotAcceptableException → 406
```

---

### ContentNegotiationManager — The Central Coordinator

```java
public class ContentNegotiationManager
    implements ContentNegotiationStrategy, MediaTypeFileExtensionResolver {

    // Ordered list of strategies — first match wins
    private final List<ContentNegotiationStrategy> strategies =
        new ArrayList<>();

    @Override
    public List<MediaType> resolveMediaTypes(
            NativeWebRequest request)
            throws HttpMediaTypeNotAcceptableException {

        for (ContentNegotiationStrategy strategy :
                this.strategies) {
            List<MediaType> mediaTypes =
                strategy.resolveMediaTypes(request);

            if (mediaTypes.equals(
                    ContentNegotiationStrategy.MEDIA_TYPE_ALL_LIST)) {
                // Strategy returned */* (no preference)
                // Try next strategy
                continue;
            }

            return mediaTypes;
            // FIRST strategy that returns specific types WINS
        }

        // All strategies returned */* (no preference)
        return ContentNegotiationStrategy.MEDIA_TYPE_ALL_LIST;
    }
}
```

---

### ContentNegotiationStrategy Implementations

#### Strategy 1: HeaderContentNegotiationStrategy (Always Active)

```java
public class HeaderContentNegotiationStrategy
    implements ContentNegotiationStrategy {

    @Override
    public List<MediaType> resolveMediaTypes(
            NativeWebRequest request)
            throws HttpMediaTypeNotAcceptableException {

        // Read Accept header from request
        String[] headerValues = request.getHeaderValues(
            HttpHeaders.ACCEPT);

        if (headerValues == null) {
            // No Accept header → return */* (accept anything)
            return MEDIA_TYPE_ALL_LIST;
        }

        // Parse the Accept header value(s)
        List<MediaType> result = new ArrayList<>();
        for (String headerValue : headerValues) {
            result.addAll(
                MediaType.parseMediaTypes(headerValue));
        }
        // Sort by quality factor (q parameter)
        MediaType.sortByQualityValue(result);
        return result;
    }
}

// Accept header parsing examples:
// "application/json"
//   → [application/json] (q=1.0 implied)

// "application/json, text/html"
//   → [application/json (q=1.0), text/html (q=1.0)]
//   → order preserved as-is for equal quality

// "text/html, application/json;q=0.9, */*;q=0.8"
//   → [text/html (q=1.0), application/json (q=0.9), */* (q=0.8)]
//   → sorted by q value descending

// "*/*" (browser default)
//   → [*/*] → compatible with everything → server picks default
```

#### Strategy 2: ParameterContentNegotiationStrategy

```java
public class ParameterContentNegotiationStrategy
    extends MappingMediaTypeFileExtensionResolver
    implements ContentNegotiationStrategy {

    // Map: format parameter value → MediaType
    // "json" → application/json
    // "xml"  → application/xml
    // "html" → text/html

    private String parameterName = "format";  // Default param name

    @Override
    public List<MediaType> resolveMediaTypes(
            NativeWebRequest request)
            throws HttpMediaTypeNotAcceptableException {

        String key = request.getParameter(
            getParameterName());
        // e.g., ?format=json → key="json"

        if (StringUtils.hasText(key)) {
            MediaType mediaType =
                lookupMediaType(key);
            // "json" → application/json
            // "xml"  → application/xml
            // Unregistered value → MediaTypeNotAcceptableException

            if (mediaType != null) {
                return Collections.singletonList(mediaType);
            }
        }

        return MEDIA_TYPE_ALL_LIST; // No format param → */*
    }
}
```

#### Strategy 3: PathExtensionContentNegotiationStrategy (DEPRECATED)

```java
// DEPRECATED since Spring 5.3 — do not use in new code
// /products.json → application/json
// /products.xml  → application/xml

// Security issues:
// File extension spoofing, ReDoS attacks, content-type sniffing exploits
// Spring 5.3: sufixPatternMatch disabled by default
// Spring 6: removed entirely

// If legacy code uses this:
configurer.favorPathExtension(false); // Explicitly disable
```

#### Strategy 4: FixedContentNegotiationStrategy

```java
// Always returns the same media type
// Useful for single-purpose endpoints
public class FixedContentNegotiationStrategy
    implements ContentNegotiationStrategy {

    private final List<MediaType> contentTypes;

    @Override
    public List<MediaType> resolveMediaTypes(
            NativeWebRequest request) {
        return this.contentTypes;
        // Always returns fixed types
    }
}
```

---

### ContentNegotiationManager Configuration

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureContentNegotiation(
            ContentNegotiationConfigurer configurer) {

        configurer
            // Strategy order: parameter FIRST, then header
            // (override default: header first)
            .favorParameter(true)
            .parameterName("format")   // ?format=json

            // Header strategy (always active, just order matters)
            .ignoreAcceptHeader(false) // default true when favorParameter enabled
            // true: Accept header completely ignored
            // false: Accept header used as fallback after parameter strategy

            // Default content type if nothing matches
            .defaultContentType(MediaType.APPLICATION_JSON)

            // Register format → mediaType mappings
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML)
            .mediaType("csv", new MediaType("text", "csv"))
            .mediaType("html", MediaType.TEXT_HTML)

            // Default content type per request path pattern (Spring 5.3+)
            .defaultContentTypeStrategy(
                new PathExtensionContentNegotiationStrategy());
    }
}
```

**Strategy priority matrix:**

```
When favorParameter=true:
  1. ParameterContentNegotiationStrategy (?format=json)
  2. HeaderContentNegotiationStrategy (Accept header)
  [if ignoreAcceptHeader=true: step 2 skipped]

When favorParameter=false (default):
  1. HeaderContentNegotiationStrategy (Accept header)
  [parameter strategy not registered]

Default:
  Only HeaderContentNegotiationStrategy active
  Accept: */* (no header) → uses producer's declared type
```

---

### The produces Condition — Filtering Handler Selection

The `produces` attribute in `@RequestMapping` participates in **handler selection** — it is part of `RequestMappingInfo` that is evaluated during URL matching:

```java
// ProducesRequestCondition.getMatchingCondition()
@Override
@Nullable
public ProducesRequestCondition getMatchingCondition(
        HttpServletRequest request) {

    if (CorsUtils.isPreFlightRequest(request)) {
        return PRE_FLIGHT_MATCH;
    }

    if (isEmpty()) {
        return this;
        // No produces condition — matches everything
    }

    List<MediaType> acceptedMediaTypes = getAcceptedMediaTypes(request);
    // From ContentNegotiationManager

    List<ProduceMediaTypeExpression> result = new ArrayList<>();
    for (ProduceMediaTypeExpression expression : this.expressions) {
        if (expression.match(acceptedMediaTypes)) {
            result.add(expression);
        }
    }

    if (!result.isEmpty()) {
        return new ProducesRequestCondition(result, this.contentNegotiationManager);
    }
    // No match → this handler cannot serve this request
    // DispatcherServlet tries next handler mapping
    return null;
}
```

**`produces` vs content negotiation in body writing — TWO separate roles:**

```
Role 1 (Handler Selection):
  @GetMapping(value="/products", produces="application/json")
  → ProducesRequestCondition checked during getHandler()
  → If Accept header incompatible with "application/json" → null returned
  → handleNoMatch() → HttpMediaTypeNotAcceptableException → 406

Role 2 (Body Writing):
  Even without produces annotation, content negotiation happens in:
  RequestResponseBodyMethodProcessor.writeWithMessageConverters()
  → ContentNegotiationManager.resolveMediaTypes(request) → acceptable types
  → HttpMessageConverter.canWrite(type, mediaType) → producible types
  → Compatible intersection → selected converter
```

---

### HttpMessageConverter — The Content Negotiation Executor

```java
public interface HttpMessageConverter<T> {

    // Can this converter read/write the given class?
    boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);
    boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);

    // What media types does this converter support?
    List<MediaType> getSupportedMediaTypes();
    List<MediaType> getSupportedMediaTypes(Class<?> clazz);

    // Read from HTTP input (deserialise)
    T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
        throws IOException, HttpMessageNotReadableException;

    // Write to HTTP output (serialise)
    void write(T t, @Nullable MediaType contentType,
               HttpOutputMessage outputMessage)
        throws IOException, HttpMessageNotWritableException;
}
```

**Default converter chain and what each handles:**

```
Converter                              Reads/Writes         MediaType
─────────────────────────────────────────────────────────────────────────
ByteArrayHttpMessageConverter          byte[]               application/octet-stream, */*
StringHttpMessageConverter             String               text/plain, */*
ResourceHttpMessageConverter           Resource             */*
ResourceRegionHttpMessageConverter     ResourceRegion       application/octet-stream
SourceHttpMessageConverter             Source               application/xml, text/xml
AllEncompassingFormHttpMessageConverter MultiValueMap       application/x-www-form-urlencoded
                                                            multipart/form-data (write only)
MappingJackson2HttpMessageConverter    Object               application/json, application/*+json
MappingJackson2XmlHttpMessageConverter Object               application/xml, text/xml, application/*+xml
Jaxb2RootElementHttpMessageConverter   @XmlRootElement     application/xml, text/xml
```

---

### writeWithMessageConverters() — The Full Algorithm

```java
// AbstractMessageConverterMethodProcessor.writeWithMessageConverters()
protected <T> void writeWithMessageConverters(
        @Nullable T value,
        MethodParameter returnType,
        ServletServerHttpRequest inputMessage,
        ServletServerHttpResponse outputMessage)
        throws IOException, HttpMediaTypeNotAcceptableException,
               HttpMessageNotWritableException {

    Class<?> valueType = getValueType(value, returnType);
    Type targetType = getGenericType(returnType);

    // STEP 1: Determine acceptable media types from request
    List<MediaType> acceptableTypes;
    try {
        acceptableTypes =
            getAcceptableMediaTypes(
                inputMessage.getServletRequest());
        // Via ContentNegotiationManager
    } catch (HttpMediaTypeNotAcceptableException ex) {
        int code = ex.getRawStatusCode();
        if (value == null || code == 406) {
            throw ex;
        }
        acceptableTypes = Collections.emptyList();
    }

    // STEP 2: Determine producible media types
    List<MediaType> producibleTypes =
        getProducibleMediaTypes(
            inputMessage.getServletRequest(),
            valueType,
            targetType);

    if (value != null && producibleTypes.isEmpty()) {
        throw new HttpMessageNotWritableException(
            "No converter found for return value of type: " +
            valueType);
    }

    // STEP 3: Find compatible media types
    List<MediaType> compatibleMediaTypes = new ArrayList<>();
    for (MediaType acceptable : acceptableTypes) {
        for (MediaType producible : producibleTypes) {
            if (acceptable.isCompatibleWith(producible)) {
                compatibleMediaTypes.add(
                    getMostSpecificMediaType(
                        acceptable, producible));
            }
        }
    }

    if (compatibleMediaTypes.isEmpty()) {
        if (!acceptableTypes.isEmpty()) {
            throw new HttpMediaTypeNotAcceptableException(
                producibleTypes);
            // → HTTP 406
        }
        return;
    }

    // STEP 4: Sort by specificity and quality
    MediaType.sortBySpecificityAndQuality(compatibleMediaTypes);

    // STEP 5: Select first concrete type
    MediaType selectedContentType = null;
    for (MediaType mediaType : compatibleMediaTypes) {
        if (mediaType.isConcrete()) {
            selectedContentType = mediaType;
            break;
        } else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
            selectedContentType = MediaType.APPLICATION_OCTET_STREAM;
            break;
        }
    }

    if (selectedContentType != null) {
        selectedContentType = selectedContentType.removeQualityValue();

        // STEP 6: ResponseBodyAdvice.beforeBodyWrite()
        for (HttpMessageConverter<?> converter :
                this.messageConverters) {
            GenericHttpMessageConverter genericConverter =
                (converter instanceof GenericHttpMessageConverter ghmc ?
                    ghmc : null);

            if (targetType != null && genericConverter != null) {
                if (genericConverter.canWrite(
                        targetType, valueType, selectedContentType)) {
                    body = getAdvice().beforeBodyWrite(
                        body, returnType, selectedContentType,
                        (Class) converter.getClass(),
                        inputMessage, outputMessage);

                    if (body != null) {
                        // STEP 7: Set Content-Type header
                        outputMessage.getHeaders().setContentType(
                            selectedContentType);
                        // STEP 8: Write body
                        genericConverter.write(body, targetType,
                            selectedContentType, outputMessage);
                    }
                    return;
                }
            } else if (targetType == null || converter.canWrite(
                    valueType, selectedContentType)) {
                // Non-generic converter path (similar)
            }
        }
    }

    if (body != null) {
        Set<MediaType> producibleMediaTypes =
            (Set<MediaType>) inputMessage.getServletRequest()
                .getAttribute(HandlerMapping
                    .PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
        // Throw if explicitly declared and nothing matched
        if (!isCompatibleWith(acceptableTypes,
                producibleTypes)) {
            throw new HttpMediaTypeNotAcceptableException(
                producibleTypes);
        }
    }
}
```

---

### Media Type Compatibility Rules

```
MediaType.isCompatibleWith() — the matching rules:

*/*        ↔ everything    → always compatible
*/*        ↔ text/html     → compatible
text/*     ↔ text/html     → compatible (wildcard subtype)
text/*     ↔ application/json → NOT compatible (different type)
text/html  ↔ text/html     → compatible (exact match)
text/html  ↔ text/plain    → NOT compatible (different subtype)

application/*+json ↔ application/json → compatible (structured syntax)
application/*+json ↔ application/vnd.api+json → compatible

Quality factor (q) affects SELECTION priority, not compatibility:
Accept: text/html;q=0.9, application/json;q=1.0
→ Both compatible if server can produce either
→ application/json selected (higher q value)

Specificity in sortBySpecificityAndQuality:
application/json > application/* > */*
text/html > text/* > */*
More specific type sorted higher → selected first
```

---

### @RequestMapping(produces) — Interaction with Converters

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    // NO produces annotation → converter selection based solely on Accept
    @GetMapping
    public List<Product> getAllNegotiated() {
        return productService.findAll();
        // Accept: application/json → MappingJackson2HttpMessageConverter
        // Accept: application/xml  → MappingJackson2XmlHttpMessageConverter
        // Accept: */*             → first capable converter (Jackson JSON)
        // Accept: text/csv        → HttpMediaTypeNotAcceptableException → 406
    }

    // WITH produces → RESTRICTS both handler selection AND converter selection
    @GetMapping(produces = "application/json")
    public List<Product> getAllJson() {
        return productService.findAll();
        // Accept: application/json → MappingJackson2HttpMessageConverter ✓
        // Accept: */*             → compatible with application/json ✓
        // Accept: application/xml → HANDLER NOT SELECTED → 406
        // (handleNoMatch detects produces mismatch before reaching converter)
    }

    // Multiple produces → multiple options
    @GetMapping(produces = {"application/json", "application/xml"})
    public List<Product> getAllMulti() {
        return productService.findAll();
        // Accept: application/json → JSON
        // Accept: application/xml  → XML
        // Accept: text/html        → 406
    }

    // DIFFERENTIATION by produces — both map to same URL
    @GetMapping(value = "/export",
                produces = "application/json")
    public List<Product> exportJson() { ... }

    @GetMapping(value = "/export",
                produces = "text/csv")
    public String exportCsv() { ... }
    // Accept: application/json → first handler
    // Accept: text/csv         → second handler
    // Accept: application/xml  → 406 (neither handles it)
}
```

---

### ContentNegotiatingViewResolver — Integration Point

```java
// ContentNegotiatingViewResolver uses ContentNegotiationManager
// to select which View to render based on client Accept header

// For view-based controllers (non-@ResponseBody):
// The same negotiation concepts apply but at VIEW level

@Bean
public ContentNegotiatingViewResolver cnvr(
        ContentNegotiationManager manager) {
    ContentNegotiatingViewResolver r =
        new ContentNegotiatingViewResolver();
    r.setContentNegotiationManager(manager);
    r.setOrder(Ordered.HIGHEST_PRECEDENCE);

    // Fallback views when no resolver matches
    r.setDefaultViews(List.of(
        new MappingJackson2JsonView()  // For Accept: application/json
    ));
    return r;
}

// How it works:
// 1. Determine acceptable media types (via ContentNegotiationManager)
// 2. Ask ALL other ViewResolvers for candidate Views
// 3. For each candidate: check View.getContentType()
// 4. Select View whose content type is compatible with client's Accept
// 5. Return selected View (or null if none found)

// Example:
// Controller returns "products" view name
// Accept: application/json
// Candidates:
//   InternalResourceViewResolver: null content type → skipped
//   MappingJackson2JsonView: content type = application/json → SELECTED
//   ThymeleafView: content type = text/html → not compatible
```

---

### Producing Different Formats from Same Controller

```java
@RestController
@RequestMapping("/api/products")
public class MultiFormatController {

    @Autowired
    private ProductService productService;

    // Content negotiation selects format:
    // Accept: application/json → JSON via Jackson
    // Accept: application/xml  → XML via Jackson XML or JAXB
    // Accept: text/csv         → CSV via custom converter
    @GetMapping(produces = {
        MediaType.APPLICATION_JSON_VALUE,
        MediaType.APPLICATION_XML_VALUE,
        "text/csv"
    })
    public ProductListResponse getAll() {
        return new ProductListResponse(
            productService.findAll());
    }

    // Parameter-based format selection (when favorParameter=true)
    // GET /api/products?format=json
    // GET /api/products?format=xml
    // GET /api/products?format=csv
    // (same method, same URL — format param selects converter)
}

// ProductListResponse — designed for multi-format output
@XmlRootElement(name = "products")  // JAXB for XML
public class ProductListResponse {
    @XmlElement(name = "product")    // JAXB element mapping
    private List<Product> items;

    // Jackson uses this for JSON (same class, different serialisation)
    public ProductListResponse(List<Product> items) {
        this.items = items;
    }

    public List<Product> getItems() { return items; }
}

// Custom CSV converter registered:
@Override
public void extendMessageConverters(
        List<HttpMessageConverter<?>> converters) {
    converters.add(new ProductCsvHttpMessageConverter());
}
```

---

### Troubleshooting 406 Not Acceptable

```
406 Not Acceptable root causes:

1. No converter can produce the requested media type:
   Controller produces Product, Jackson on classpath
   Client: Accept: text/csv
   No CSV converter → 406

2. produces annotation excludes the requested type:
   @GetMapping(produces="application/json")
   Client: Accept: application/xml
   ProducesRequestCondition fails → 406

3. ignoreAcceptHeader=true and no format parameter:
   ContentNegotiationManager ignores Accept header
   No default type configured → returns */* → fallback needed

4. configureMessageConverters() used (removed all defaults):
   Only custom converter registered
   Client: Accept: application/json
   No Jackson converter → 406

DEBUGGING STEPS:
  1. Check ContentNegotiationManager strategy configuration
  2. Check HttpMessageConverter chain (extendMessageConverters vs configure)
  3. Check @RequestMapping produces annotation scope
  4. Check if AcceptHeader is being overridden by proxy
  5. Enable DEBUG logging for org.springframework.web
```

---

## 2️⃣ CODE EXAMPLES

### Complete Content Negotiation Configuration

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // Configure content negotiation strategy
    @Override
    public void configureContentNegotiation(
            ContentNegotiationConfigurer configurer) {

        configurer
            // BOTH parameter AND header strategies active
            // Parameter checked FIRST
            .favorParameter(true)
            .parameterName("format")

            // Don't completely ignore Accept header
            .ignoreAcceptHeader(false)

            // Default when no preference determinable
            .defaultContentType(
                MediaType.APPLICATION_JSON,
                MediaType.TEXT_HTML     // fallback if JSON not acceptable
            )

            // Format parameter mappings
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML)
            .mediaType("csv", new MediaType("text", "csv"))
            .mediaType("pdf", new MediaType("application", "pdf"))
            .mediaType("html", MediaType.TEXT_HTML);
    }

    // Add custom message converter
    @Override
    public void extendMessageConverters(
            List<HttpMessageConverter<?>> converters) {

        // Add at position BEFORE Jackson XML to give it priority
        // over JAXB for XML output
        int jacksonIndex = -1;
        for (int i = 0; i < converters.size(); i++) {
            if (converters.get(i) instanceof
                    MappingJackson2HttpMessageConverter) {
                jacksonIndex = i;
                break;
            }
        }

        // Custom CSV converter
        converters.add(jacksonIndex + 1,
            new CsvHttpMessageConverter());

        // Custom PDF converter
        converters.add(new PdfHttpMessageConverter());

        // Modify Jackson ObjectMapper settings
        converters.stream()
            .filter(c -> c instanceof
                MappingJackson2HttpMessageConverter)
            .map(c -> (MappingJackson2HttpMessageConverter) c)
            .findFirst()
            .ifPresent(c -> c.getObjectMapper()
                .configure(SerializationFeature
                    .WRITE_DATES_AS_TIMESTAMPS, false));
    }
}
```

---

### Multi-Format REST Endpoint

```java
@RestController
@RequestMapping("/api/reports")
public class ReportController {

    @Autowired
    private ReportService reportService;

    // Single method handles multiple formats:
    // GET /api/reports/sales
    // GET /api/reports/sales?format=json
    // GET /api/reports/sales?format=xml
    // GET /api/reports/sales?format=csv
    // GET /api/reports/sales (with Accept: application/pdf)
    @GetMapping(
        value = "/sales",
        produces = {
            MediaType.APPLICATION_JSON_VALUE,
            MediaType.APPLICATION_XML_VALUE,
            "text/csv",
            "application/pdf"
        }
    )
    public SalesReport getSalesReport(
            @RequestParam @DateTimeFormat(iso=DateTimeFormat.ISO.DATE)
                LocalDate from,
            @RequestParam @DateTimeFormat(iso=DateTimeFormat.ISO.DATE)
                LocalDate to) {

        return reportService.generateSales(from, to);
        // SalesReport serialised differently per format:
        // JSON: {"total": 1000, "orders": [...]}
        // XML:  <salesReport><total>1000</total>...</salesReport>
        // CSV:  Custom CsvHttpMessageConverter processes it
        // PDF:  Custom PdfHttpMessageConverter processes it
    }
}

// SalesReport annotated for multiple serialisation targets
@XmlRootElement(name = "salesReport")
@XmlAccessorType(XmlAccessType.FIELD)
public class SalesReport {

    @JsonProperty("total")
    @XmlElement
    private BigDecimal total;

    @JsonProperty("orders")
    @XmlElementWrapper(name = "orders")
    @XmlElement(name = "order")
    private List<OrderSummary> orders;

    @JsonProperty("generated")
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")
    @XmlElement
    private LocalDateTime generated;

    // getters/setters
}
```

---

### Custom HttpMessageConverter for CSV

```java
// Full CSV converter implementation
public class CsvHttpMessageConverter
    extends AbstractHttpMessageConverter<Object> {

    private static final MediaType TEXT_CSV =
        new MediaType("text", "csv",
            StandardCharsets.UTF_8);

    public CsvHttpMessageConverter() {
        super(TEXT_CSV);
    }

    @Override
    protected boolean supports(Class<?> clazz) {
        // Supports any object that can be iterated
        return Iterable.class.isAssignableFrom(clazz) ||
               clazz.isArray() ||
               clazz.isAnnotationPresent(CsvExportable.class);
    }

    @Override
    protected void writeInternal(
            Object object,
            HttpOutputMessage outputMessage)
            throws IOException, HttpMessageNotWritableException {

        outputMessage.getHeaders()
            .setContentType(TEXT_CSV);
        outputMessage.getHeaders()
            .set("Content-Disposition",
                "attachment; filename=export.csv");

        Writer writer = new OutputStreamWriter(
            outputMessage.getBody(),
            StandardCharsets.UTF_8);

        // Write BOM for Excel compatibility
        writer.write("\uFEFF");

        // Write header row
        CsvExportable annotation = object.getClass()
            .getAnnotation(CsvExportable.class);
        if (annotation != null) {
            writer.write(
                String.join(",", annotation.headers()));
            writer.write("\n");
        }

        // Write data rows
        if (object instanceof Iterable<?> iterable) {
            for (Object item : iterable) {
                writer.write(toCsvRow(item));
                writer.write("\n");
            }
        }

        writer.flush();
    }

    @Override
    protected Object readInternal(
            Class<?> clazz,
            HttpInputMessage inputMessage)
            throws IOException {
        // CSV import implementation
        return CsvParser.parse(inputMessage.getBody(), clazz);
    }

    private String toCsvRow(Object item) {
        // Use reflection or CsvRow annotation
        // to extract field values as CSV
        return CsvRowMapper.map(item);
    }
}
```

---

### ContentNegotiationManager Testing and Inspection

```java
@RestController
@RequestMapping("/debug")
public class NegotiationDebugController {

    @Autowired
    private ContentNegotiationManager contentNegotiationManager;

    @GetMapping("/negotiation")
    public Map<String, Object> debugNegotiation(
            WebRequest webRequest,
            HttpServletRequest request) throws Exception {

        // Show what the ContentNegotiationManager resolves
        List<MediaType> resolved =
            contentNegotiationManager
                .resolveMediaTypes(webRequest);

        // Show what converters are available
        Map<String, Object> debug = new LinkedHashMap<>();
        debug.put("acceptedTypes",
            resolved.stream()
                .map(MediaType::toString)
                .collect(Collectors.toList()));
        debug.put("acceptHeader",
            request.getHeader("Accept"));
        debug.put("formatParam",
            request.getParameter("format"));

        return debug;
    }

    // Demonstrate handler selection by produces
    @GetMapping(value = "/select",
                produces = MediaType.APPLICATION_JSON_VALUE)
    public Map<String, String> jsonHandler() {
        return Map.of("format", "json",
                      "handler", "jsonHandler");
    }

    @GetMapping(value = "/select",
                produces = MediaType.APPLICATION_XML_VALUE)
    @ResponseBody
    public Map<String, String> xmlHandler() {
        return Map.of("format", "xml",
                      "handler", "xmlHandler");
    }
    // GET /debug/select Accept: application/json → jsonHandler
    // GET /debug/select Accept: application/xml  → xmlHandler
    // GET /debug/select Accept: text/html        → 406
}
```

---

### Edge Case — Accept Header */* Behaviour

```java
@RestController
public class WildcardAcceptController {

    // Browser sends Accept: text/html,application/json;q=0.9,*/*;q=0.8

    @GetMapping("/data")
    public Product getData() {
        return productService.findFirst();
    }

    // What happens:
    // ContentNegotiationManager resolves:
    //   [text/html (q=1.0), application/json (q=0.9), */* (q=0.8)]
    //
    // getProducibleMediaTypes() for Product:
    //   [application/json, application/xml] (from registered converters)
    //
    // Compatibility check:
    //   text/html vs application/json → NOT compatible
    //   text/html vs application/xml  → NOT compatible
    //   application/json vs application/json → COMPATIBLE (q=0.9)
    //   application/json vs application/xml  → NOT compatible
    //   */* vs application/json → COMPATIBLE (q=0.8)
    //   */* vs application/xml  → COMPATIBLE (q=0.8)
    //
    // Compatible pairs sorted by specificity+quality:
    //   (application/json, q=0.9) comes before (*/* → application/xml, q=0.8)
    //
    // SELECTED: application/json
    // → Jackson serialises Product

    // MORAL: */* in Accept header does NOT mean "any format"
    // It means "any format is acceptable" but MORE SPECIFIC types
    // with higher q values take priority
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
`ContentNegotiationManager` has `favorParameter=true` and `ignoreAcceptHeader=false`. A request arrives with `?format=json` AND `Accept: application/xml`. Which media type is determined?

A) `application/xml` — Accept header has higher authority
B) `application/json` — parameter strategy checked FIRST, wins
C) `HttpMediaTypeNotAcceptableException` — conflicting preferences
D) `application/json;q=0.5` — weighted average of both

**Answer: B**
`ContentNegotiationManager.resolveMediaTypes()` iterates strategies in order. With `favorParameter=true`, `ParameterContentNegotiationStrategy` is first. It finds `format=json` → returns `[application/json]` — not `*/*`. Since a specific type was returned, iteration STOPS. `HeaderContentNegotiationStrategy` never consulted. Result: `application/json`.

---

**Q2 — Select All That Apply**
Which behaviours are TRUE about `@RequestMapping(produces="application/json")`?

A) It causes the controller method to only be invoked when `Accept: application/json`
B) It causes the response to always have `Content-Type: application/json` regardless of Accept
C) `Accept: */*` is compatible with `application/json` — handler IS selected
D) It prevents `HttpMessageConverter` selection — Jackson is hardcoded
E) A handler without `produces` is less specific than one with `produces` for same request

**Answer: A, C, E**
A: handler selection uses `ProducesRequestCondition` — if Accept incompatible, handler returns null. B is wrong — Content-Type is set by the converter after negotiation, not forced by produces annotation. C: `*/*` is compatible with `application/json` — handler IS selected. D is wrong — converter selection still happens via content negotiation, produces just filters which handlers are eligible. E: correct — produces condition makes handler MORE specific (higher specificity ranking).

---

**Q3 — Code Prediction**
```java
@RestController
public class TestController {

    @GetMapping(value="/data",
                produces="application/json")
    public Map<String, String> asJson() {
        return Map.of("type", "json");
    }

    @GetMapping(value="/data",
                produces="application/xml")
    public Map<String, String> asXml() {
        return Map.of("type", "xml");
    }
}
```
`GET /data` with `Accept: */*` — what happens?

A) `IllegalStateException` — ambiguous mapping
B) `asJson()` called — first registered handler
C) `asXml()` called — XML has higher specificity
D) HTTP 406 — `*/*` doesn't match specific produces

**Answer: B**
`*/*` is compatible with BOTH `application/json` AND `application/xml`. Both handlers match. Specificity comparison: both have produces condition, equal specificity by that metric. Spring selects based on registration order — `asJson()` was registered first. Result: `asJson()` called, response is `{"type":"json"}` with `Content-Type: application/json`.

---

**Q4 — MCQ**
`writeWithMessageConverters()` receives acceptable types `[text/html, */*]` and producible types `[application/json]`. What is the negotiated result?

A) `HttpMediaTypeNotAcceptableException` — `text/html` not in producible
B) `application/json` selected — `*/*` is compatible with `application/json`
C) `text/html` selected — it has higher priority (q=1.0 implied)
D) `application/octet-stream` selected — default fallback

**Answer: B**
Compatibility check: `text/html` vs `application/json` → NOT compatible. `*/*` vs `application/json` → COMPATIBLE. `application/json` is in compatible types. `sortBySpecificityAndQuality`: `application/json` (concrete, q=1.0 from `*/*`) — selected. Response: `application/json`.

---

**Q5 — True/False**
When `configureContentNegotiation().ignoreAcceptHeader(true)` is set, requests with `Accept: application/xml` will never receive XML responses from `@ResponseBody` methods.

**Answer: False**
`ignoreAcceptHeader(true)` means the `HeaderContentNegotiationStrategy` is NOT registered. The Accept header is ignored for strategy resolution. BUT the `produces` annotation on `@RequestMapping` still uses the Accept header for HANDLER SELECTION (`ProducesRequestCondition`). Additionally, if `favorParameter=true` and client sends `?format=xml`, XML can still be returned. `ignoreAcceptHeader` only affects the `ContentNegotiationManager` strategy, not `produces` filtering.

---

**Q6 — Select All That Apply**
`MediaType.isCompatibleWith()` returns `true` for which pairs?

A) `application/json` and `application/json`
B) `*/*` and `text/html`
C) `text/*` and `text/plain`
D) `application/*` and `application/xml`
E) `text/*` and `application/json`

**Answer: A, B, C, D**
E: `text/*` and `application/json` — `text/*` means any `text` subtype. `application/json` has type `application`, not `text`. NOT compatible. A-D all follow wildcard compatibility rules correctly.

---

**Q7 — MCQ**
`StringHttpMessageConverter` is registered with `getSupportedMediaTypes()` returning `[text/plain, */*]`. A request with `Accept: application/json` arrives for a `String` return value. What converter is selected?

A) `StringHttpMessageConverter` — it supports `*/*` which matches `application/json`
B) `MappingJackson2HttpMessageConverter` — JSON specific, higher priority
C) `HttpMediaTypeNotAcceptableException` — no concrete match
D) `StringHttpMessageConverter` — String type takes priority over converter specificity

**Answer: B**
`writeWithMessageConverters()` finds compatible pairs:
- `StringHttpMessageConverter.canWrite(String.class, application/json)` → `*/*` is compatible with `application/json` → candidate
- `MappingJackson2HttpMessageConverter.canWrite(String.class, application/json)` → `application/json` is compatible → candidate

After `sortBySpecificityAndQuality`: `application/json` (concrete, from Jackson) is MORE SPECIFIC than `application/json` from `*/*` wildcard of StringHttpMessageConverter. Jackson's converter selected. String written as JSON string: `"some text"`.

---

**Q8 — Scenario**
`favorParameter=true`, `parameterName="format"`, format mappings: `json→application/json`. Request: `GET /api/data?format=pdf` where `pdf` is NOT in the format mappings. What happens?

A) `*/*` returned — unknown format treated as no preference
B) `HttpMediaTypeNotAcceptableException` thrown with 406
C) Parameter strategy ignored, Accept header consulted
D) `IllegalArgumentException` — invalid format value

**Answer: B**
`ParameterContentNegotiationStrategy.resolveMediaTypes()`: `key="pdf"`. `lookupMediaType("pdf")` → `pdf` not in mappings → returns `null`. When the parameter exists but maps to no known type: `MediaTypeNotAcceptableException` thrown (the parameter had a value, just an unrecognised one). This is distinct from the parameter being absent (which returns `*/*`).

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `ignoreAcceptHeader=true` does NOT bypass `produces` filtering**
Setting `ignoreAcceptHeader=true` removes the `HeaderContentNegotiationStrategy` from the `ContentNegotiationManager`. This affects `writeWithMessageConverters()` body writing. But `ProducesRequestCondition` in handler mapping STILL reads the Accept header to filter handler selection. An endpoint with `produces="application/json"` will STILL reject requests with `Accept: text/html` even with `ignoreAcceptHeader=true`. These are two separate Accept header usages.

**Trap 2 — `produces` is TWO separate filters**
`produces` annotation participates in (1) handler selection (`ProducesRequestCondition` during URL matching) AND (2) influences producible types during body writing. Removing `produces` does NOT remove content negotiation — it just makes the handler accept ANY Accept header (no pre-filter). Content negotiation still runs during body writing.

**Trap 3 — `favorParameter=true` with unknown format value → 406**
If parameter exists but value is unregistered (e.g., `?format=xlsx` with no xlsx mapping), `ParameterContentNegotiationStrategy` throws `HttpMediaTypeNotAcceptableException`. This is DIFFERENT from parameter absent (returns `*/*`). Exam questions test whether developers know that an UNRECOGNISED format param causes 406 vs absent param causes fallback.

**Trap 4 — `*/*` means "any" but specific types win**
Browser Accept header: `text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8`. ALL types match `*/*` at q=0.8. But `text/html` at q=1.0 is checked FIRST. If server can produce `text/html` (e.g., has HTML view AND `MappingJackson2JsonView`), HTML is selected. Many REST API developers are surprised that browser requests to REST APIs get HTML instead of JSON if a view resolver is configured.

**Trap 5 — `sortBySpecificityAndQuality` — specificity beats quality**
`Accept: application/json;q=0.5, */*;q=1.0`. One might think `*/*;q=1.0` should win. But `sortBySpecificityAndQuality` sorts by SPECIFICITY first, THEN quality. `application/json` (concrete, specific) sorts BEFORE `*/*` (wildcard) even with lower quality. More specific types always take precedence over wildcards regardless of q value.

**Trap 6 — `configureMessageConverters()` vs `extendMessageConverters()`**
`configureMessageConverters()` REPLACES all default converters — Jackson is gone. 406 for all JSON requests. `extendMessageConverters()` ADDS to defaults — Jackson stays. This is the most common content negotiation configuration mistake — using configure instead of extend.

**Trap 7 — Path extension strategy is deprecated/removed**
`/products.json` style URL content negotiation is DEPRECATED in Spring 5.3 and REMOVED in Spring 6.0. Applications relying on suffix patterns for content negotiation break silently when upgrading. Use parameter strategy (`?format=json`) or Accept header instead.

---

## 5️⃣ SUMMARY SHEET

```
CONTENT NEGOTIATION — TWO-PHASE MODEL
─────────────────────────────────────────────────────
Phase 1: What does client want?
  ContentNegotiationManager.resolveMediaTypes()
  → ParameterContentNegotiationStrategy (?format=json)
  → HeaderContentNegotiationStrategy (Accept header)
  → First non-*/* result wins

Phase 2: What can server produce?
  getProducibleMediaTypes()
  → @RequestMapping(produces=...) if specified
  → All registered HttpMessageConverter supported types

MATCH: compatible intersection → selected converter

CONTENTNEGOTIATIONMANAGER STRATEGIES
─────────────────────────────────────────────────────
Default:      HeaderContentNegotiationStrategy only
favorParameter=true: ParameterStrategy FIRST, then Header
parameterName="format": customise param name
ignoreAcceptHeader=true: removes Header strategy from ContentNegotiationManager
  CAUTION: does NOT bypass produces-based handler filtering

MEDIA TYPE COMPATIBILITY RULES
─────────────────────────────────────────────────────
*/*     ↔ anything      → compatible (wildcard)
text/*  ↔ text/html     → compatible (subtype wildcard)
text/*  ↔ application/json → NOT compatible (different type)
text/html ↔ text/html   → compatible (exact)
application/*+json ↔ application/json → compatible (suffix match)

SPECIFICITY ORDER (for sortBySpecificityAndQuality):
  Concrete type > subtype wildcard > */*
  Higher q value > lower q value (for same specificity)
  SPECIFICITY beats QUALITY in sorting

PRODUCES ANNOTATION — TWO ROLES
─────────────────────────────────────────────────────
Role 1: Handler selection (ProducesRequestCondition)
  If Accept not compatible with produces → handler returns null → 406

Role 2: Restricts producible types during body writing
  getProducibleMediaTypes() returns only declared types
  Instead of all converter-supported types

Accept: */* compatible with any produces value → handler always selected
Accept: application/json compatible with produces="application/json" → selected
Accept: application/xml with produces="application/json" → 406

HTTPMESSAGECONVERTER DEFAULT CHAIN
─────────────────────────────────────────────────────
ByteArrayHttpMessageConverter   → byte[]           → */*
StringHttpMessageConverter      → String           → text/plain, */*
ResourceHttpMessageConverter    → Resource         → */*
SourceHttpMessageConverter      → Source           → application/xml, text/xml
MappingJackson2HttpMessageConverter → Object       → application/json, application/*+json
MappingJackson2XmlHttpMessageConverter → Object    → application/xml, text/xml

COMMON 406 CAUSES
─────────────────────────────────────────────────────
1. No converter handles requested media type
2. produces annotation excludes Accept type
3. configureMessageConverters() removed Jackson (use extendMessageConverters)
4. Unknown ?format= parameter value (not in mediaType mappings)
5. ignoreAcceptHeader=true + no format param + no default content type

DEPRECATED/REMOVED
─────────────────────────────────────────────────────
Path extension: /products.json → deprecated Spring 5.3, removed Spring 6.0
Use: ?format=json or Accept header instead
favorPathExtension(false) to explicitly disable

PARAMETER STRATEGY EDGE CASES
─────────────────────────────────────────────────────
?format=json (registered) → [application/json] → parameter wins
?format=pdf (NOT registered) → HttpMediaTypeNotAcceptableException → 406
No ?format parameter → [*/*] → fall through to Header strategy

WRITEWITHMESSAGECONVERTERS ALGORITHM
─────────────────────────────────────────────────────
1. Get acceptable types (ContentNegotiationManager)
2. Get producible types (@produces or converter capabilities)
3. Find compatible intersections
4. sortBySpecificityAndQuality
5. Select first concrete compatible type
6. ResponseBodyAdvice.beforeBodyWrite()
7. converter.write() → Content-Type header set

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "favorParameter=true: ?format=json checked BEFORE Accept header"
• "ignoreAcceptHeader ignores header in ContentNegotiationManager — NOT in produces filtering"
• "produces has two roles: handler selection + restricts producible types"
• "Specificity beats quality in media type sorting: application/json beats */* even at lower q"
• "configureMessageConverters() replaces ALL defaults — use extendMessageConverters() to add"
• "Unknown ?format= value → 406; absent ?format= → */* → fallback to Accept"
• "Path extension /products.json: deprecated Spring 5.3, removed Spring 6"
• "Accept: */* compatible with ALL produces values — handler always selected"
• "Two handlers same URL, different produces: content negotiation selects correct one"
• "StringHttpMessageConverter supports */* but specific converters win on specificity"
```

---
