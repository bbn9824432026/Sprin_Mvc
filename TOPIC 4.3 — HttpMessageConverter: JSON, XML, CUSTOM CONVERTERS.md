# TOPIC 4.3 — HttpMessageConverter: JSON, XML, CUSTOM CONVERTERS

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What HttpMessageConverter Really Does

`HttpMessageConverter` is the **serialisation/deserialisation bridge** between Java objects and HTTP message bodies. It operates in two directions: reading a request body (deserialising bytes → Java object) and writing a response body (serialising Java object → bytes). Every `@RequestBody`, `@ResponseBody`, `ResponseEntity`, and `HttpEntity` goes through this mechanism. Understanding it at depth means understanding: how converters are selected, what happens during Jackson serialisation, how JAXB XML works, how to implement custom converters, and how the converter chain interacts with content negotiation.

---

### The HttpMessageConverter Interface — Complete Contract

```java
public interface HttpMessageConverter<T> {

    // Can this converter READ the given class from the given media type?
    boolean canRead(Class<?> clazz,
                    @Nullable MediaType mediaType);

    // Can this converter WRITE the given class as the given media type?
    boolean canWrite(Class<?> clazz,
                     @Nullable MediaType mediaType);

    // What media types does this converter support?
    List<MediaType> getSupportedMediaTypes();

    // What media types does this converter support for this specific class?
    // Allows per-class specialization
    default List<MediaType> getSupportedMediaTypes(Class<?> clazz) {
        return (canRead(clazz, null) || canWrite(clazz, null) ?
            getSupportedMediaTypes() :
            Collections.emptyList());
    }

    // Deserialise — read from HTTP input message
    T read(Class<? extends T> clazz,
           HttpInputMessage inputMessage)
        throws IOException, HttpMessageNotReadableException;

    // Serialise — write to HTTP output message
    void write(T t, @Nullable MediaType contentType,
               HttpOutputMessage outputMessage)
        throws IOException, HttpMessageNotWritableException;
}

// GenericHttpMessageConverter — for generic types (List<Product>)
public interface GenericHttpMessageConverter<T>
    extends HttpMessageConverter<T> {

    boolean canRead(Type type,
                    @Nullable Class<?> contextClass,
                    @Nullable MediaType mediaType);

    T read(Type type,
           @Nullable Class<?> contextClass,
           HttpInputMessage inputMessage)
        throws IOException, HttpMessageNotReadableException;

    boolean canWrite(Type type,
                     @Nullable Class<?> contextClass,
                     @Nullable MediaType mediaType);

    void write(T t, Type type,
               @Nullable MediaType contentType,
               HttpOutputMessage outputMessage)
        throws IOException, HttpMessageNotWritableException;
}
```

---

### AbstractHttpMessageConverter — The Base Template

```java
public abstract class AbstractHttpMessageConverter<T>
    implements HttpMessageConverter<T> {

    private List<MediaType> supportedMediaTypes =
        Collections.emptyList();

    // Charset for text-based converters
    @Nullable
    protected Charset defaultCharset;

    protected AbstractHttpMessageConverter(
            MediaType... supportedMediaTypes) {
        setSupportedMediaTypes(Arrays.asList(supportedMediaTypes));
    }

    @Override
    public boolean canRead(Class<?> clazz,
                            @Nullable MediaType mediaType) {
        return supports(clazz) && canRead(mediaType);
    }

    protected boolean canRead(@Nullable MediaType mediaType) {
        if (mediaType == null) return true;
        for (MediaType supportedMediaType :
                getSupportedMediaTypes()) {
            if (supportedMediaType.isCompatibleWith(mediaType)) {
                return true;
            }
        }
        return false;
    }

    @Override
    public boolean canWrite(Class<?> clazz,
                             @Nullable MediaType mediaType) {
        return supports(clazz) && canWrite(mediaType);
    }

    // TEMPLATE METHOD — subclasses implement this
    protected abstract boolean supports(Class<?> clazz);

    // TEMPLATE METHODS for actual reading/writing
    protected abstract T readInternal(
        Class<? extends T> clazz,
        HttpInputMessage inputMessage)
        throws IOException, HttpMessageNotReadableException;

    protected abstract void writeInternal(
        T t, HttpOutputMessage outputMessage)
        throws IOException, HttpMessageNotWritableException;

    @Override
    public final void write(T t,
                            @Nullable MediaType contentType,
                            HttpOutputMessage outputMessage)
            throws IOException, HttpMessageNotWritableException {

        HttpHeaders headers = outputMessage.getHeaders();

        // Add Content-Type header if not set
        if (headers.getContentType() == null) {
            MediaType contentTypeToUse = contentType;
            if (contentType == null ||
                    !contentType.isConcrete()) {
                contentTypeToUse =
                    getDefaultContentType(t);
            }
            if (contentTypeToUse != null) {
                contentTypeToUse =
                    addDefaultCharset(contentTypeToUse,
                        this.defaultCharset);
                headers.setContentType(contentTypeToUse);
            }
        }

        // Add Content-Length if known
        if (headers.getContentLength() < 0) {
            Long contentLength = getContentLength(
                t, headers.getContentType());
            if (contentLength != null) {
                headers.setContentLength(contentLength);
            }
        }

        // Call subclass implementation
        writeInternal(t, outputMessage);
        outputMessage.getBody().flush();
    }
}
```

---

### MappingJackson2HttpMessageConverter — Deep Analysis

Jackson is the default JSON converter in Spring MVC. Understanding exactly how it integrates matters for performance tuning, custom serialisation, and debugging.

```java
public class MappingJackson2HttpMessageConverter
    extends AbstractJackson2HttpMessageConverter {

    // Constructor: registers application/json + application/*+json
    public MappingJackson2HttpMessageConverter() {
        this(Jackson2ObjectMapperBuilder.json().build());
    }

    public MappingJackson2HttpMessageConverter(
            ObjectMapper objectMapper) {
        super(objectMapper,
              MediaType.APPLICATION_JSON,
              new MediaType("application", "*+json"));
    }
}
```

#### AbstractJackson2HttpMessageConverter — The Real Work

```java
// readInternal() — deserialisation
@Override
public Object read(Type type, @Nullable Class<?> contextClass,
                    HttpInputMessage inputMessage)
        throws IOException, HttpMessageNotReadableException {

    JavaType javaType = getJavaType(type, contextClass);
    return readJavaType(javaType, inputMessage);
}

private Object readJavaType(JavaType javaType,
                              HttpInputMessage inputMessage) {

    // Determine charset from Content-Type
    MediaType contentType =
        inputMessage.getHeaders().getContentType();
    Charset charset = getCharset(contentType);

    // Check for empty body
    // (Content-Length: 0 or GET with no body)
    boolean isEmptyBody = isEmptyBody(inputMessage);

    ObjectMapper objectMapper = selectObjectMapper(
        javaType.getRawClass(), contentType);
    // objectMapperRegistry: allows per-MediaType ObjectMappers
    // If no specific mapper: use default objectMapper

    try {
        InputStream inputStream = StreamUtils.nonClosing(
            inputMessage.getBody());
        // nonClosing prevents accidental stream close

        if (inputStream.available() == 0 || isEmptyBody) {
            // Empty body:
            // If type is Optional → Optional.empty()
            // If type requires non-null → exception or null
            return getForEmptyBody(javaType);
        }

        // Create JsonParser from input
        // ObjectMapper reads JSON and maps to JavaType
        return objectMapper.readValue(inputStream, javaType);

    } catch (InvalidDefinitionException ex) {
        throw new HttpMessageConversionException(
            "Type definition error: " + ex.getType(), ex);
    } catch (JsonProcessingException ex) {
        throw new HttpMessageNotReadableException(
            "JSON parse error: " + ex.getOriginalMessage(),
            ex, inputMessage);
        // This becomes HTTP 400 Bad Request via
        // DefaultHandlerExceptionResolver
    }
}

// writeInternal() — serialisation
@Override
protected void writeInternal(Object object,
                               @Nullable Type type,
                               HttpOutputMessage outputMessage)
        throws IOException, HttpMessageNotWritableException {

    MediaType contentType =
        outputMessage.getHeaders().getContentType();

    // Charset from Content-Type header
    JsonEncoding encoding = getJsonEncoding(contentType);

    // Select ObjectMapper (possibly per content type)
    ObjectMapper objectMapper = selectObjectMapper(
        object.getClass(), contentType);

    // Create JsonGenerator
    JsonGenerator generator = objectMapper.getFactory()
        .createGenerator(outputMessage.getBody(), encoding);

    try {
        // Write JSON encoding hint (BOM for UTF-8 if needed)
        writePrefix(generator, object);

        // Determine actual type for generic serialisation
        Class<?> serializationView = null;
        FilterProvider filters = null;
        Object value = object;
        JavaType javaType = null;

        if (object instanceof MappingJacksonValue container) {
            // MappingJacksonValue for serialisation views + filters
            value = container.getValue();
            serializationView = container.getSerializationView();
            filters = container.getFilters();
            javaType = container.getJavaType();
        }

        if (javaType != null && javaType.isContainerType()) {
            // Generic container: List<Product>, Map<String, Product>
            ObjectWriter objectWriter =
                (serializationView != null ?
                    objectMapper.writerWithView(serializationView) :
                    objectMapper.writer());
            if (filters != null) {
                objectWriter =
                    objectWriter.with(filters);
            }
            objectWriter.forType(javaType)
                .writeValue(generator, value);
        } else {
            // Standard serialisation
            ObjectWriter objectWriter =
                (serializationView != null ?
                    objectMapper.writerWithView(serializationView) :
                    objectMapper.writer());
            if (filters != null) {
                objectWriter =
                    objectWriter.with(filters);
            }
            objectWriter.writeValue(generator, value);
        }

        writeSuffix(generator, object);
        generator.flush();

    } catch (InvalidDefinitionException ex) {
        throw new HttpMessageConversionException(
            "Type definition error: " + ex.getType(), ex);
    } catch (JsonProcessingException ex) {
        throw new HttpMessageNotWritableException(
            "Could not write JSON: " + ex.getOriginalMessage(),
            ex);
    }
}
```

---

### Jackson ObjectMapper — Configuration and Singleton

```java
// The ObjectMapper is a SINGLETON in Spring MVC
// Shared across all requests
// Must be THREAD-SAFE configuration (it is — ObjectMapper is thread-safe)

// Spring Boot auto-configures via JacksonAutoConfiguration:
@Bean
@ConditionalOnMissingBean
public ObjectMapper jacksonObjectMapper(
        Jackson2ObjectMapperBuilder builder) {
    return builder.build();
}

// Jackson2ObjectMapperBuilder common configuration:
@Bean
public Jackson2ObjectMapperBuilderCustomizer customizer() {
    return builder -> {
        // Serialisation features
        builder.featuresToDisable(
            SerializationFeature.WRITE_DATES_AS_TIMESTAMPS,
            SerializationFeature.FAIL_ON_EMPTY_BEANS
        );
        builder.featuresToEnable(
            SerializationFeature.INDENT_OUTPUT  // Dev only
        );

        // Deserialisation features
        builder.featuresToDisable(
            DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES
        );

        // Null handling
        builder.serializationInclusion(
            JsonInclude.Include.NON_NULL);

        // Date format
        builder.simpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSZ");
        // OR:
        builder.modules(new JavaTimeModule());

        // Property naming
        builder.propertyNamingStrategy(
            PropertyNamingStrategies.SNAKE_CASE);
        // Product.productName → "product_name" in JSON
    };
}
```

---

### Jackson Annotations — The Essential Set

```java
// Class-level annotations:
@JsonIgnoreProperties(ignoreUnknown = true)
// Ignore unknown JSON fields on deserialisation
// Equivalent to FAIL_ON_UNKNOWN_PROPERTIES=false for this class

@JsonIgnoreProperties({"password", "secretKey"})
// Always exclude these fields from serialisation

@JsonInclude(JsonInclude.Include.NON_NULL)
// Don't serialise null fields
// Options: ALWAYS, NON_NULL, NON_ABSENT, NON_EMPTY, NON_DEFAULT

@JsonPropertyOrder({"id", "name", "price", "createdAt"})
// Control serialisation order

@JsonTypeName("product")
@JsonTypeInfo(use=JsonTypeInfo.Id.NAME, include=As.PROPERTY,
              property="type")
// Polymorphic type handling

// Field/method annotations:
@JsonProperty("product_name")  // Map to different JSON key
private String name;

@JsonIgnore  // Exclude from serialisation AND deserialisation
private String internalCode;

@JsonGetter("displayName")  // Custom serialiser method
public String getComputedName() { ... }

@JsonSetter("display_name")  // Custom deserialiser method
public void setName(String name) { ... }

@JsonAlias({"name", "title", "label"})  // Multiple names on deserialise
private String displayName;

@JsonFormat(pattern="yyyy-MM-dd", timezone="UTC")
private LocalDate releaseDate;

@JsonSerialize(using = MoneySerializer.class)
@JsonDeserialize(using = MoneyDeserializer.class)
private Money price;

// Conditional serialisation:
@JsonInclude(value=JsonInclude.Include.CUSTOM,
             valueFilter=EmptyListFilter.class)
private List<Tag> tags;
```

---

### Jackson Serialisation Views — @JsonView

```java
// Define view interfaces (marker interfaces)
public class Views {
    public interface Public {}
    public interface Internal extends Public {}
    public interface Admin extends Internal {}
}

// Apply to domain object
public class User {
    @JsonView(Views.Public.class)
    private Long id;

    @JsonView(Views.Public.class)
    private String username;

    @JsonView(Views.Internal.class)
    private String email;

    @JsonView(Views.Admin.class)
    private String passwordHash;

    @JsonView(Views.Admin.class)
    private String internalNotes;
}

// Apply in controller — view determines which fields are included
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    @JsonView(Views.Public.class)
    public User getPublic(@PathVariable Long id) {
        return userService.findById(id);
        // Only id, username serialised
    }

    @GetMapping("/{id}/profile")
    @JsonView(Views.Internal.class)
    public User getInternal(@PathVariable Long id) {
        return userService.findById(id);
        // id, username, email serialised (Internal extends Public)
    }

    @GetMapping("/{id}/admin")
    @JsonView(Views.Admin.class)
    public User getAdmin(@PathVariable Long id) {
        return userService.findById(id);
        // All fields serialised
    }
}

// Jackson implementation in MappingJackson2HttpMessageConverter:
// When @JsonView found on handler method:
// → objectWriter = objectMapper.writerWithView(viewClass)
// → Jackson only includes fields where @JsonView annotation
//   is assignable from viewClass
```

---

### MappingJackson2XmlHttpMessageConverter — XML via Jackson

```java
// Jackson XML uses same annotation model as JSON
// But produces XML output via jackson-dataformat-xml

// Maven dependency:
// <dependency>
//   <groupId>com.fasterxml.jackson.dataformat</groupId>
//   <artifactId>jackson-dataformat-xml</artifactId>
// </dependency>

// Domain object with XML annotations:
@XmlRootElement(name = "product")  // JAXB (also works with Jackson XML)
@JsonRootName("product")           // Jackson JSON root name
public class Product {

    @JacksonXmlProperty(isAttribute=true)  // → <product id="42">
    private Long id;

    @JacksonXmlProperty(localName="productName") // → <productName>Widget</productName>
    private String name;

    @JacksonXmlElementWrapper(localName="tags")  // Wrapper element
    @JacksonXmlProperty(localName="tag")         // Item element name
    private List<String> tags;

    @JsonProperty  // JSON uses this; XML uses JacksonXmlProperty
    private BigDecimal price;
}

// Produces:
// JSON: {"id":1,"name":"Widget","tags":["electronics"],"price":9.99}
// XML:
// <product id="1">
//   <productName>Widget</productName>
//   <tags>
//     <tag>electronics</tag>
//   </tags>
//   <price>9.99</price>
// </product>
```

---

### Jaxb2RootElementHttpMessageConverter — JAXB XML

```java
// Uses Jakarta XML Binding (JAXB) for XML serialisation
// Requires @XmlRootElement on the class

@XmlRootElement(name = "product")
@XmlAccessorType(XmlAccessType.FIELD)
public class Product {

    @XmlAttribute
    private Long id;

    @XmlElement
    private String name;

    @XmlTransient  // Equivalent to @JsonIgnore
    private String internalCode;

    @XmlElement(name = "price")
    @XmlJavaTypeAdapter(MoneyAdapter.class)  // Custom type conversion
    private Money price;

    @XmlElementWrapper(name = "tags")
    @XmlElement(name = "tag")
    private List<String> tags;
}

// JAXB context is created and CACHED per class
// Thread-safe for reading; write operations use marshaller per thread
// JAXBContext creation is expensive — caching is critical

// Jaxb2RootElementHttpMessageConverter:
// canRead: is @XmlRootElement present? AND mediaType is XML?
// canWrite: is @XmlRootElement present? AND mediaType is XML?

// Jackson XML vs JAXB:
// Jackson XML: better performance, uses XmlMapper
// JAXB: standard Java API, more annotation options, slower startup
// Both produce XML but with different defaults
// Converter order matters: first in chain wins
```

---

### StringHttpMessageConverter — Charset Trap

```java
// Default charset is ISO-8859-1 — critical trap
public class StringHttpMessageConverter
    extends AbstractHttpMessageConverter<String> {

    public static final Charset DEFAULT_CHARSET =
        StandardCharsets.ISO_8859_1;
    // NOT UTF-8

    public StringHttpMessageConverter() {
        this(DEFAULT_CHARSET);
        // Uses ISO-8859-1 by default
    }

    public StringHttpMessageConverter(Charset defaultCharset) {
        super(defaultCharset,
              MediaType.TEXT_PLAIN,
              MediaType.ALL);
        this.availableCharsets =
            new ArrayList<>(
                Charset.availableCharsets().values());
    }

    @Override
    protected String readInternal(Class<? extends String> clazz,
                                   HttpInputMessage inputMessage)
            throws IOException {
        Charset charset = getContentTypeCharset(
            inputMessage.getHeaders().getContentType());
        // Reads using charset from Content-Type header
        // Falls back to defaultCharset if none specified
        return StreamUtils.copyToString(
            inputMessage.getBody(), charset);
    }
}

// Fix: replace with UTF-8 charset
@Override
public void extendMessageConverters(
        List<HttpMessageConverter<?>> converters) {
    converters.removeIf(c ->
        c instanceof StringHttpMessageConverter);
    // Insert at position 0 to maintain priority
    converters.add(0,
        new StringHttpMessageConverter(
            StandardCharsets.UTF_8));
}
```

---

### ByteArrayHttpMessageConverter

```java
// Simplest converter — byte arrays
public class ByteArrayHttpMessageConverter
    extends AbstractHttpMessageConverter<byte[]> {

    public ByteArrayHttpMessageConverter() {
        super(MediaType.APPLICATION_OCTET_STREAM,
              MediaType.ALL);
        // Supports ANY media type via MediaType.ALL
    }

    @Override
    protected boolean supports(Class<?> clazz) {
        return byte[].class == clazz;
    }

    @Override
    public byte[] readInternal(Class<? extends byte[]> clazz,
                                HttpInputMessage inputMessage)
            throws IOException {
        long contentLength =
            inputMessage.getHeaders().getContentLength();
        ByteArrayOutputStream bos = new ByteArrayOutputStream(
            contentLength >= 0 ? (int) contentLength : 4096);
        StreamUtils.copy(inputMessage.getBody(), bos);
        return bos.toByteArray();
    }

    @Override
    protected void writeInternal(byte[] bytes,
                                   HttpOutputMessage outputMessage)
            throws IOException {
        // Just copy bytes to output
        StreamUtils.copy(bytes, outputMessage.getBody());
    }
}

// Usage in controller:
@GetMapping("/download/{id}")
public ResponseEntity<byte[]> download(@PathVariable Long id) {
    byte[] content = fileService.getContent(id);
    return ResponseEntity.ok()
        .contentType(MediaType.APPLICATION_PDF)
        .header("Content-Disposition",
            "attachment; filename=file.pdf")
        .body(content);
    // ByteArrayHttpMessageConverter.canWrite(byte[], application/pdf)?
    // → supports(byte[]) = true
    // → canWrite(application/pdf): application/pdf compatible with */*? YES
    // → writes bytes to output
}
```

---

### ResourceHttpMessageConverter — Resource Type

```java
// Handles org.springframework.core.io.Resource implementations
// InputStreamResource, FileSystemResource, ClassPathResource, etc.

@GetMapping("/files/{name}")
public ResponseEntity<Resource> downloadFile(
        @PathVariable String name) {
    Path filePath = storageService.load(name);
    Resource resource = new UrlResource(filePath.toUri());

    if (!resource.exists() || !resource.isReadable()) {
        throw new ResourceNotFoundException(name);
    }

    return ResponseEntity.ok()
        .header("Content-Disposition",
            "attachment; filename=\"" + name + "\"")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(resource);
    // ResourceHttpMessageConverter handles Resource types
    // Streams file content to response
    // Does NOT load entire file into memory (streaming)
    // ContentLength set from resource.contentLength()
}
```

---

### AbstractJackson2HttpMessageConverter — Per-MediaType ObjectMapper

```java
// Advanced: register different ObjectMappers per content type
@Bean
public MappingJackson2HttpMessageConverter jsonConverter() {
    MappingJackson2HttpMessageConverter converter =
        new MappingJackson2HttpMessageConverter();

    // Default ObjectMapper for application/json
    converter.setObjectMapper(defaultObjectMapper());

    // Register different ObjectMapper for
    // application/vnd.api+json (JSON:API format)
    ObjectMapper jsonApiMapper = new ObjectMapper();
    jsonApiMapper.registerModule(
        new JsonApiModule());

    converter.registerObjectMappersForType(
        Object.class,
        map -> map.put(
            new MediaType("application", "vnd.api+json"),
            jsonApiMapper)
    );

    return converter;
}
```

---

## 2️⃣ CODE EXAMPLES

### Complete Custom HttpMessageConverter

```java
// Custom converter for YAML format
@Component
public class YamlHttpMessageConverter
    extends AbstractHttpMessageConverter<Object> {

    private static final MediaType MEDIA_TYPE_YAML =
        new MediaType("application", "yaml",
            StandardCharsets.UTF_8);
    private static final MediaType MEDIA_TYPE_YAML_TEXT =
        new MediaType("text", "yaml",
            StandardCharsets.UTF_8);

    private final ObjectMapper yamlMapper;

    public YamlHttpMessageConverter() {
        super(MEDIA_TYPE_YAML, MEDIA_TYPE_YAML_TEXT);
        // YAMLFactory from jackson-dataformat-yaml
        this.yamlMapper = new ObjectMapper(
            new YAMLFactory()
                .disable(YAMLGenerator.Feature.WRITE_DOC_START_MARKER)
                .enable(YAMLGenerator.Feature.MINIMIZE_QUOTES)
        );
        this.yamlMapper.findAndRegisterModules();
    }

    @Override
    protected boolean supports(Class<?> clazz) {
        // Support any class that Jackson can handle
        return true;
    }

    @Override
    public boolean canRead(Class<?> clazz,
                            @Nullable MediaType mediaType) {
        return supports(clazz) && isYamlMediaType(mediaType);
    }

    @Override
    public boolean canWrite(Class<?> clazz,
                             @Nullable MediaType mediaType) {
        return supports(clazz) && isYamlMediaType(mediaType);
    }

    private boolean isYamlMediaType(
            @Nullable MediaType mediaType) {
        if (mediaType == null) return true;
        return MEDIA_TYPE_YAML.isCompatibleWith(mediaType) ||
               MEDIA_TYPE_YAML_TEXT.isCompatibleWith(mediaType);
    }

    @Override
    protected Object readInternal(
            Class<?> clazz,
            HttpInputMessage inputMessage)
            throws IOException, HttpMessageNotReadableException {
        try {
            return yamlMapper.readValue(
                inputMessage.getBody(), clazz);
        } catch (JsonProcessingException ex) {
            throw new HttpMessageNotReadableException(
                "YAML parse error: " + ex.getMessage(),
                ex, inputMessage);
        }
    }

    @Override
    protected void writeInternal(
            Object object,
            HttpOutputMessage outputMessage)
            throws IOException, HttpMessageNotWritableException {
        try {
            // Set Content-Type with charset
            outputMessage.getHeaders()
                .setContentType(MEDIA_TYPE_YAML);

            yamlMapper.writeValue(
                outputMessage.getBody(), object);

        } catch (JsonProcessingException ex) {
            throw new HttpMessageNotWritableException(
                "Could not write YAML: " + ex.getMessage(),
                ex);
        }
    }
}

// Registration
@Override
public void extendMessageConverters(
        List<HttpMessageConverter<?>> converters) {
    converters.add(new YamlHttpMessageConverter());
}

// Usage
@GetMapping(value="/products",
            produces="application/yaml")
public List<Product> getYaml() {
    return productService.findAll();
}
// GET /products Accept: application/yaml
// → YamlHttpMessageConverter selected
// → Products serialised as YAML
```

---

### Custom Serialiser and Deserialiser

```java
// Custom serialiser for Money type
public class MoneySerializer
    extends JsonSerializer<Money> {

    @Override
    public void serialize(Money money,
                           JsonGenerator gen,
                           SerializerProvider provider)
            throws IOException {
        gen.writeStartObject();
        gen.writeNumberField("amount",
            money.getAmount().doubleValue());
        gen.writeStringField("currency",
            money.getCurrency().getCurrencyCode());
        gen.writeStringField("formatted",
            money.format());
        gen.writeEndObject();
    }
}

// Custom deserialiser for Money type
public class MoneyDeserializer
    extends JsonDeserializer<Money> {

    @Override
    public Money deserialize(JsonParser p,
                              DeserializationContext ctxt)
            throws IOException {
        JsonNode node = p.getCodec().readTree(p);
        BigDecimal amount = node.get("amount")
            .decimalValue();
        Currency currency = Currency.getInstance(
            node.get("currency").asText());
        return new Money(amount, currency);
    }
}

// Register globally via module
public class MoneyModule extends SimpleModule {
    public MoneyModule() {
        addSerializer(Money.class, new MoneySerializer());
        addDeserializer(Money.class, new MoneyDeserializer());
    }
}

@Bean
public Jackson2ObjectMapperBuilderCustomizer moneyCustomizer() {
    return builder -> builder.modules(new MoneyModule());
}

// OR register on class:
@JsonSerialize(using = MoneySerializer.class)
@JsonDeserialize(using = MoneyDeserializer.class)
public class Money { ... }

// JSON output:
// {"amount":99.99,"currency":"USD","formatted":"$99.99"}
```

---

### Jackson Configuration for REST API

```java
// Production-grade ObjectMapper configuration
@Configuration
public class JacksonConfig {

    @Bean
    @Primary
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();

        // Date/Time handling
        mapper.registerModule(new JavaTimeModule());
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        mapper.setDateFormat(
            new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSZ"));

        // Unknown fields
        mapper.configure(
            DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES,
            false);

        // Null values
        mapper.setSerializationInclusion(
            JsonInclude.Include.NON_NULL);

        // Empty collections — include them
        // mapper.setSerializationInclusion(
        //     JsonInclude.Include.NON_EMPTY); // would exclude []

        // Enum handling
        mapper.configure(
            DeserializationFeature.READ_ENUMS_USING_TO_STRING,
            true);
        mapper.configure(
            SerializationFeature.WRITE_ENUMS_USING_TO_STRING,
            true);

        // BigDecimal handling
        mapper.enable(
            DeserializationFeature.USE_BIG_DECIMAL_FOR_FLOATS);
        mapper.configure(
            SerializationFeature.WRITE_BIGDECIMAL_AS_PLAIN,
            true);

        // Fail on numbers for expected strings
        mapper.configure(
            MapperFeature.ALLOW_COERCION_OF_SCALARS,
            false);

        return mapper;
    }
}
```

---

### Polymorphic Type Handling

```java
// Handling class hierarchies in JSON

// Base class with type info
@JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,
    include = JsonTypeInfo.As.PROPERTY,
    property = "type"
)
@JsonSubTypes({
    @JsonSubTypes.Type(value = CardPayment.class,
                       name = "card"),
    @JsonSubTypes.Type(value = BankTransfer.class,
                       name = "bank_transfer"),
    @JsonSubTypes.Type(value = CryptoPayment.class,
                       name = "crypto")
})
public abstract class Payment {
    private BigDecimal amount;
    private Currency currency;
}

public class CardPayment extends Payment {
    private String cardNumber;
    private String cvv;
    private LocalDate expiry;
}

public class BankTransfer extends Payment {
    private String accountNumber;
    private String routingNumber;
}

// JSON for CardPayment:
// {
//   "type": "card",
//   "amount": 99.99,
//   "currency": "USD",
//   "cardNumber": "4111111111111111",
//   "cvv": "123",
//   "expiry": "2025-12-01"
// }

// Controller:
@PostMapping("/payments")
public Payment processPayment(
        @RequestBody Payment payment) {
    // Jackson deserialises to correct subtype based on "type" field
    if (payment instanceof CardPayment cp) {
        return cardPaymentService.process(cp);
    } else if (payment instanceof BankTransfer bt) {
        return bankTransferService.process(bt);
    }
    throw new UnsupportedOperationException();
}
```

---

### ResponseBodyAdvice — Pre-Serialisation Hook

```java
// Intercepts body before writing via MessageConverter
@ControllerAdvice
public class CompressionAdvice
    implements ResponseBodyAdvice<Object> {

    @Override
    public boolean supports(
            MethodParameter returnType,
            Class<? extends HttpMessageConverter<?>> converterType) {
        // Apply to all JSON responses
        return MappingJackson2HttpMessageConverter.class
            .isAssignableFrom(converterType);
    }

    @Override
    public Object beforeBodyWrite(
            @Nullable Object body,
            MethodParameter returnType,
            MediaType selectedContentType,
            Class<? extends HttpMessageConverter<?>> converterType,
            ServerHttpRequest request,
            ServerHttpResponse response) {

        if (body == null) return null;

        // Example: wrap in standard API envelope
        String requestId = extractRequestId(request);
        response.getHeaders()
            .add("X-Request-ID", requestId);

        // Return modified body — this is what gets serialised
        if (body instanceof Page<?> page) {
            return PageResponse.of(page);
            // Converts Spring Data Page to custom format
        }

        return body;
    }
}
```

---

### Edge Case — Converter Registration Order Matters

```java
@Override
public void extendMessageConverters(
        List<HttpMessageConverter<?>> converters) {

    // PROBLEM: MappingJackson2XmlHttpMessageConverter is registered
    // BEFORE Jaxb2RootElementHttpMessageConverter
    // For application/xml:
    // → MappingJackson2XmlHttpMessageConverter checks canWrite()
    //   → supports(MyClass.class) = true (Jackson can write anything)
    //   → JSON → YES (Jackson XML wins even if JAXB is available)

    // If you want JAXB for @XmlRootElement classes:
    // Move Jaxb2RootElementHttpMessageConverter BEFORE Jackson XML

    int jacksonXmlIndex = -1;
    for (int i = 0; i < converters.size(); i++) {
        if (converters.get(i) instanceof
                MappingJackson2XmlHttpMessageConverter) {
            jacksonXmlIndex = i;
        }
    }

    if (jacksonXmlIndex >= 0) {
        // Insert JAXB converter BEFORE Jackson XML
        converters.add(jacksonXmlIndex,
            new Jaxb2RootElementHttpMessageConverter());
    }

    // Now for @XmlRootElement classes + application/xml:
    // JAXB converter checked first → wins
    // For non-@XmlRootElement classes + application/xml:
    // JAXB canWrite() = false → Jackson XML wins
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
`MappingJackson2HttpMessageConverter.canWrite(Product.class, application/json)` returns `true`. Which conditions must BOTH be true?

A) Jackson is on the classpath AND `Product` has `@JsonSerializable`
B) `supports(Product.class)` returns `true` AND `canWrite(application/json)` returns `true`
C) `Product` has no-arg constructor AND all fields have getters
D) `ObjectMapper` can serialise `Product` AND `Content-Type` header is set

**Answer: B**
`AbstractHttpMessageConverter.canWrite()` calls two methods: `supports(clazz)` — can this converter handle this class? And `canWrite(mediaType)` — is this media type in the supported list? For `MappingJackson2HttpMessageConverter`: `supports(Product.class)` → checks if Jackson can serialise it (true for POJOs). `canWrite(application/json)` → checks if `application/json` is compatible with `[application/json, application/*+json]`. Both must be true.

---

**Q2 — Select All That Apply**
`StringHttpMessageConverter` has default charset ISO-8859-1. Which of the following are TRUE?

A) A `String` containing `"é"` may be corrupted when sent if client expects UTF-8
B) Setting `defaultCharset=UTF-8` fixes all encoding issues
C) The charset used for writing is determined by the `Content-Type` header's charset, falling back to `defaultCharset`
D) Spring Boot auto-configures `StringHttpMessageConverter` with UTF-8
E) `StringHttpMessageConverter` is only used when `Content-Type: text/plain`

**Answer: A, C, D**
A is correct — ISO-8859-1 cannot represent all Unicode characters correctly for all clients. B is partially true but incomplete — if `Content-Type: text/plain; charset=UTF-8` is set in request/response, that charset is used regardless of `defaultCharset`. C is the accurate description of charset selection. D is correct — Spring Boot's auto-configuration sets UTF-8. E is wrong — `StringHttpMessageConverter` supports `*/*` as well, not just `text/plain`.

---

**Q3 — Ordering**
When `@ResponseBody` method returns `Product`, order the processing steps:

- `ResponseBodyAdvice.beforeBodyWrite()` called
- `MappingJackson2HttpMessageConverter.writeInternal()` executes
- `Content-Type: application/json` set on response headers
- `getProducibleMediaTypes()` finds `[application/json, application/xml]`
- Compatible media types sorted by specificity
- `MappingJackson2HttpMessageConverter.canWrite()` returns `true`

**Correct Order:**
1. `getProducibleMediaTypes()` finds `[application/json, application/xml]`
2. Compatible media types sorted by specificity
3. `MappingJackson2HttpMessageConverter.canWrite()` returns `true`
4. `ResponseBodyAdvice.beforeBodyWrite()` called
5. `Content-Type: application/json` set on response headers
6. `MappingJackson2HttpMessageConverter.writeInternal()` executes

---

**Q4 — MCQ**
`@JsonView(Views.Public.class)` is on a `@GetMapping` method returning `User`. How does `MappingJackson2HttpMessageConverter` process this?

A) `@JsonView` is a Spring annotation — Jackson doesn't know about it
B) `objectMapper.writerWithView(Views.Public.class)` creates a restricted writer that only includes `@JsonView(Views.Public.class)` annotated fields
C) All fields are serialised — `@JsonView` is only for deserialisation
D) A separate `ObjectMapper` instance is created per-view type at startup

**Answer: B**
`AbstractJackson2HttpMessageConverter.writeInternal()` detects when the return value is wrapped in `MappingJacksonValue` with a view class (set by `JsonViewReturnValueHandler`). It calls `objectMapper.writerWithView(viewClass)`. This creates an `ObjectWriter` configured to only serialise fields annotated with `@JsonView` where the view class is assignable from the specified view. Same `ObjectMapper` instance, different `ObjectWriter` configuration.

---

**Q5 — True/False**
`ResourceHttpMessageConverter` loads the entire file into memory before writing to the response, making it unsuitable for large files.

**Answer: False**
`ResourceHttpMessageConverter.writeInternal()` uses streaming: `StreamUtils.copy(resource.getInputStream(), outputMessage.getBody())`. This streams data from the resource's `InputStream` to the response `OutputStream` in chunks (default 4096 bytes). The entire file is never held in memory. This is why `ResponseEntity<Resource>` is suitable for large file downloads.

---

**Q6 — MCQ**
`Jaxb2RootElementHttpMessageConverter` and `MappingJackson2XmlHttpMessageConverter` are both registered. A `Product` class has `@XmlRootElement`. `GET /products` with `Accept: application/xml`. Which converter is used?

A) Always `MappingJackson2XmlHttpMessageConverter` — Jackson takes priority
B) `Jaxb2RootElementHttpMessageConverter` — registered first (Spring default order)
C) The converter registered FIRST in the list whose `canWrite()` returns `true`
D) Always `Jaxb2RootElementHttpMessageConverter` — `@XmlRootElement` requires JAXB

**Answer: C**
Content negotiation selects the first converter in the list that can write the type for the media type. In the default Spring MVC configuration, `MappingJackson2XmlHttpMessageConverter` is registered BEFORE `Jaxb2RootElementHttpMessageConverter`. Jackson XML checks `supports(Product.class)` → true (Jackson can serialise anything). It wins. To change this, explicitly reorder converters in `extendMessageConverters()`.

---

**Q7 — Select All That Apply**
Which `HttpMessageConverter` features are TRUE?

A) The converter chain is walked for EACH request when selecting a converter
B) `canRead()` and `canWrite()` can return different answers for the same class
C) `GenericHttpMessageConverter` is needed to handle `List<Product>` generics correctly
D) Converter selection is cached — same type + media type always uses same converter
E) `extendMessageConverters()` preserves existing converters

**Answer: B, C, E**
A is wrong — converter selection for the same `MethodParameter` IS cached in `HandlerMethodReturnValueHandlerComposite`. B is true — a converter could support writing JSON but not reading it (asymmetric). C is true — `GenericHttpMessageConverter` handles parameterized types like `List<Product>` by using `Type` instead of just `Class`. D is partially true — the return value handler is cached per `MethodParameter`, but the converter selection within `writeWithMessageConverters()` is not directly cached (though `canWrite()` is typically O(1)). E is true — `extendMessageConverters()` adds to existing list, unlike `configureMessageConverters()` which replaces.

---

**Q8 — Scenario**
A custom `HttpMessageConverter<Product>` is registered. It claims `canWrite(Product.class, */*) = true`. The request has `Accept: application/json`. A registered `MappingJackson2HttpMessageConverter` also claims `canWrite(Product.class, application/json) = true`. Which converter is selected?

A) Custom converter — registered last, has `*/*` wildcard
B) `MappingJackson2HttpMessageConverter` — `application/json` is more specific than `*/*`
C) Custom converter — registered last has highest priority
D) `MappingJackson2HttpMessageConverter` — Jackson always takes priority

**Answer: B**
`writeWithMessageConverters()` finds compatible pairs by matching acceptable types against producible types. The compatible pair `(application/json, application/json)` from Jackson is MORE SPECIFIC than `(application/json, */*)`  from the custom converter. `sortBySpecificityAndQuality` sorts more specific pairs first. Jackson is selected. The `*/*` wildcard support of the custom converter gives lower specificity in the negotiation.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `configureMessageConverters()` removes Jackson entirely**
`configureMessageConverters(List)` REPLACES the entire default converter chain. If Jackson is not explicitly re-added, ALL REST endpoints return 406 for `Accept: application/json`. Always use `extendMessageConverters()` to ADD converters or modify existing ones. Using `configure` is almost always a mistake.

**Trap 2 — `StringHttpMessageConverter` supports `*/*`**
Many developers don't realise `StringHttpMessageConverter` supports `*/*` (all media types). For `String` return values in `@RestController`, if the client sends `Accept: application/json`, `StringHttpMessageConverter` COULD match (via `*/*`). But `MappingJackson2HttpMessageConverter` supports `application/json` specifically and wins on specificity. The confusion: why doesn't `String` get serialised as JSON `"some text"`? Because `StringHttpMessageConverter` matches first in some configs. Always check converter order.

**Trap 3 — Converter `canWrite()` uses `isCompatibleWith()` not exact match**
`canWrite(mediaType)` checks if the supported media type `isCompatibleWith(mediaType)`. `*/*` is compatible with everything. `application/*` is compatible with any `application/` subtype. This means a converter declaring `*/*` can write ANY media type, even ones it was never designed for. The wildcard coverage is broader than most developers expect.

**Trap 4 — Jackson XML vs JAXB converter order**
Default Spring MVC registration order puts `MappingJackson2XmlHttpMessageConverter` BEFORE `Jaxb2RootElementHttpMessageConverter`. For `@XmlRootElement` classes with XML response, Jackson XML wins by default (it supports any class). To use JAXB for annotated classes, manually reorder converters in `extendMessageConverters()`.

**Trap 5 — ObjectMapper is a singleton — thread safety**
The `ObjectMapper` instance in `MappingJackson2HttpMessageConverter` is shared across all requests. `ObjectMapper` itself is thread-safe for reading/writing (immutable configuration). However: if you call `objectMapper.configure()` or `objectMapper.setDateFormat()` after bean creation in a multi-threaded context — race condition. Configure ObjectMapper only during application startup, never per-request.

**Trap 6 — `@JsonIgnoreProperties` vs `@JsonIgnore` scope**
`@JsonIgnoreProperties({"field1"})` on class — applies to BOTH serialisation AND deserialisation. Incoming JSON with `field1` is ignored; outgoing JSON omits `field1`.
`@JsonIgnore` on field — also both directions by default.
`@JsonProperty(access=READ_ONLY)` — only in deserialisation (ignored for serialisation).
`@JsonProperty(access=WRITE_ONLY)` — only in serialisation.
Many developers use `@JsonIgnore` intending only to hide from output, inadvertently preventing the field from being set from input too.

**Trap 7 — `HttpMessageNotReadableException` vs `HttpMessageNotWritableException`**
`HttpMessageNotReadableException` — during `@RequestBody` deserialisation (JSON parse failure, type mismatch). → HTTP 400 (handled by `DefaultHandlerExceptionResolver`).
`HttpMessageNotWritableException` — during `@ResponseBody` serialisation (Jackson can't serialise return value). → HTTP 500 (unhandled, propagates as server error).
Different HTTP status codes, different directions. Exam questions test which exception maps to which status.

---

## 5️⃣ SUMMARY SHEET

```
HTTPMESSAGECONVERTER INTERFACE
─────────────────────────────────────────────────────
canRead(class, mediaType)     → can this converter deserialise?
canWrite(class, mediaType)    → can this converter serialise?
getSupportedMediaTypes()      → what media types it handles
read()  → HTTP body → Java object  (deserialisation)
write() → Java object → HTTP body  (serialisation)

GenericHttpMessageConverter: handles parameterized types (List<Product>)

DEFAULT CONVERTER CHAIN (registration order)
─────────────────────────────────────────────────────
ByteArrayHttpMessageConverter   → byte[]          → application/octet-stream, */*
StringHttpMessageConverter      → String          → text/plain, */* (charset: ISO-8859-1!)
ResourceHttpMessageConverter    → Resource        → */* (streaming)
SourceHttpMessageConverter      → Source          → application/xml, text/xml
AllEncompassingFormHttpMessageConverter → form    → x-www-form-urlencoded, multipart
MappingJackson2HttpMessageConverter → Object      → application/json, application/*+json
MappingJackson2XmlHttpMessageConverter → Object   → application/xml, text/xml
Jaxb2RootElementHttpMessageConverter → @XmlRoot  → application/xml, text/xml

First converter with canWrite()=true WINS

STRINGHTTPMESSAGECONVERTER TRAP
─────────────────────────────────────────────────────
Default charset: ISO-8859-1 (NOT UTF-8)
Fix: converters.add(0, new StringHttpMessageConverter(StandardCharsets.UTF_8))
Spring Boot: auto-configures UTF-8

CONFIGURE VS EXTEND
─────────────────────────────────────────────────────
configureMessageConverters(): REPLACES ALL defaults (Jackson removed!)
extendMessageConverters():    ADDS TO existing list (Jackson preserved)
Use extendMessageConverters() — almost always the correct choice

JACKSON OBJECTMAPPER
─────────────────────────────────────────────────────
Singleton — shared across all requests — thread-safe for read/write
Configure ONLY at startup — never modify per-request
Key features:
  @JsonView            → field-level serialisation filtering
  @JsonIgnore          → exclude from both directions
  @JsonProperty        → rename + access control
  @JsonInclude         → include/exclude null/empty
  @JsonTypeInfo        → polymorphic type handling
  @JsonFormat          → date/number formatting
  setSerializationInclusion(NON_NULL) → omit null fields globally
  FAIL_ON_UNKNOWN_PROPERTIES=false → ignore unknown JSON fields

EXCEPTION TYPES
─────────────────────────────────────────────────────
HttpMessageNotReadableException → deserialise failure → HTTP 400
  Handled by DefaultHandlerExceptionResolver

HttpMessageNotWritableException → serialise failure → HTTP 500
  NOT handled by DefaultHandlerExceptionResolver
  Propagates as server error

RESOURCEHTTPMESSAGECONVERTER
─────────────────────────────────────────────────────
Streaming: StreamUtils.copy() → chunks, never loads full file
ResponseEntity<Resource> → suitable for large file downloads
Content-Length set from resource.contentLength()

JACKSON XML VS JAXB
─────────────────────────────────────────────────────
Default order: MappingJackson2XmlHttpMessageConverter BEFORE JAXB
  → Jackson XML wins for @XmlRootElement + application/xml
To use JAXB: reorder in extendMessageConverters()
  → Insert Jaxb2RootElementHttpMessageConverter BEFORE Jackson XML

RESPONSEBODYADVICE
─────────────────────────────────────────────────────
Called inside writeWithMessageConverters() BEFORE converter.write()
Can modify body, add headers, change Content-Type
Returning null from beforeBodyWrite() → no body written
Applied to @ResponseBody and ResponseEntity responses

CONVERTER SELECTION SPECIFICITY
─────────────────────────────────────────────────────
application/json > application/* > */*
More specific producible type beats wildcard even at lower q value
First converter in chain with matching canWrite() wins for same specificity

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "configureMessageConverters() replaces ALL defaults — use extendMessageConverters()"
• "StringHttpMessageConverter default charset: ISO-8859-1, NOT UTF-8"
• "ObjectMapper is a singleton — thread-safe read/write, configure at startup only"
• "ResourceHttpMessageConverter streams — never loads full file into memory"
• "HttpMessageNotReadableException → 400; HttpMessageNotWritableException → 500"
• "MappingJackson2XmlHttpMessageConverter registered BEFORE JAXB by default"
• "canWrite() uses isCompatibleWith() — */* matches everything"
• "@JsonIgnore excludes from BOTH serialisation AND deserialisation"
• "GenericHttpMessageConverter needed for List<Product> generic type handling"
• "ResponseBodyAdvice.beforeBodyWrite(): returning null suppresses body entirely"
```

---
