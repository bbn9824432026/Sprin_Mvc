# TOPIC 5.3 — MULTIPART FILE UPLOAD: MultipartResolver, @RequestPart, FILE HANDLING

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What Multipart Really Is

Multipart is an HTTP encoding format (`Content-Type: multipart/form-data`) where a single request body contains multiple "parts" separated by a boundary string. Each part has its own headers and body — one part might be a form field value, another might be binary file content. Spring MVC must parse this complex body before the handler method can access the uploaded data.

---

### Multipart Request Structure

```
POST /api/upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Length: 1234

------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="name"

Widget Pro
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="price"

19.99
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="image"; filename="product.jpg"
Content-Type: image/jpeg

[binary JPEG data]
------WebKitFormBoundary7MA4YWxkTrZu0gW
Content-Disposition: form-data; name="document"; filename="spec.pdf"
Content-Type: application/pdf

[binary PDF data]
------WebKitFormBoundary7MA4YWxkTrZu0gW--
```

---

### MultipartResolver — The Parsing Gateway

```java
// DispatcherServlet.checkMultipart() — called FIRST in doDispatch()
protected HttpServletRequest checkMultipart(
        HttpServletRequest request)
        throws MultipartException {

    if (this.multipartResolver != null &&
            this.multipartResolver.isMultipart(request)) {
        // Content-Type starts with "multipart/"?

        if (WebUtils.getNativeRequest(
                request,
                MultipartHttpServletRequest.class) != null) {
            // Already parsed (nested dispatch)
            return request;
        }

        // PARSE the multipart request
        return this.multipartResolver
            .resolveMultipart(request);
        // Returns MultipartHttpServletRequest wrapper
        // All parts now accessible via getFile(), getParts()
    }
    return request;
}

// CLEANUP always happens in doDispatch() finally:
if (multipartRequestParsed) {
    cleanupMultipart(processedRequest);
    // Deletes temp files from disk
    // Called even if exception thrown
}
```

---

### StandardServletMultipartResolver — Modern Approach

```java
// Uses Servlet 3.0 Part API — no additional libraries needed
public class StandardServletMultipartResolver
    implements MultipartResolver {

    // Bean name MUST be "multipartResolver"
    // DispatcherServlet.initMultipartResolver() looks up by name

    @Override
    public boolean isMultipart(HttpServletRequest request) {
        return StringUtils.startsWithIgnoreCase(
            request.getContentType(), "multipart/");
    }

    @Override
    public MultipartHttpServletRequest resolveMultipart(
            HttpServletRequest request)
            throws MultipartException {

        return new StandardMultipartHttpServletRequest(
            request, this.resolveLazily);
        // resolveLazily=false: parse all parts immediately
        // resolveLazily=true: parse on first access
    }

    @Override
    public void cleanupMultipart(
            MultipartHttpServletRequest request) {
        // Delete temp files
        if (!(request instanceof
                AbstractMultipartHttpServletRequest amr) ||
                amr.isResolved()) {
            try {
                for (Part part :
                        request.getParts()) {
                    if (request.getFile(part.getName())
                            != null) {
                        part.delete();
                        // Removes temp file from disk
                    }
                }
            } catch (Throwable ex) {
                LogFactory.getLog(getClass())
                    .warn("Failed to perform cleanup", ex);
            }
        }
    }
}
```

**Configuration requirements for `StandardServletMultipartResolver`:**

```java
// PLAIN SPRING MVC — must configure MultipartConfigElement
public class WebAppInitializer
    extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected void customizeRegistration(
            ServletRegistration.Dynamic registration) {

        registration.setMultipartConfig(
            new MultipartConfigElement(
                "/tmp",        // Temp file location
                5 * 1024 * 1024,   // Max file size: 5MB
                20 * 1024 * 1024,  // Max request size: 20MB
                0              // File size threshold (bytes before writing to disk)
                // 0 = write to disk immediately
                // > 0 = buffer in memory up to this size
            )
        );
    }

    // AND register the resolver bean:
}

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Bean(name = "multipartResolver")
    // Bean name MUST be exactly "multipartResolver"
    public MultipartResolver multipartResolver() {
        return new StandardServletMultipartResolver();
    }
}

// SPRING BOOT — auto-configured via MultipartAutoConfiguration:
// spring.servlet.multipart.enabled=true (default)
// spring.servlet.multipart.max-file-size=1MB
// spring.servlet.multipart.max-request-size=10MB
// spring.servlet.multipart.file-size-threshold=0
// spring.servlet.multipart.location=/tmp
// spring.servlet.multipart.resolve-lazily=false
```

---

### CommonsMultipartResolver — Legacy Approach

```java
// Uses Apache Commons FileUpload — additional dependency
// For Servlet 2.x compatibility or advanced configuration

@Bean(name = "multipartResolver")
public CommonsMultipartResolver multipartResolver() {
    CommonsMultipartResolver resolver =
        new CommonsMultipartResolver();

    resolver.setMaxUploadSize(
        20 * 1024 * 1024);     // 20MB total
    resolver.setMaxUploadSizePerFile(
        5 * 1024 * 1024);      // 5MB per file
    resolver.setMaxInMemorySize(
        4096);                  // 4KB in memory, rest to disk
    resolver.setDefaultEncoding("UTF-8");
    resolver.setUploadTempDir(
        new FileSystemResource("/tmp/uploads"));

    return resolver;
}

// CommonsMultipartResolver vs StandardServletMultipartResolver:
// CommonsMultipartResolver:
//   + More configuration options (per-file size, encoding)
//   + Works with Servlet 2.x
//   - Additional dependency (commons-fileupload)
//   - Less performant for large files
// StandardServletMultipartResolver (PREFERRED):
//   + Built into Servlet 3.0 API
//   + No extra dependencies
//   + Better performance
//   - Requires MultipartConfigElement configuration
```

---

### MultipartFile Interface — Complete API

```java
public interface MultipartFile extends InputStreamSource {

    // Part name (HTML input field name)
    String getName();
    // e.g., "image" from <input type="file" name="image"/>

    // Original filename from client
    @Nullable
    String getOriginalFilename();
    // e.g., "product-photo.jpg"
    // SECURITY: Do NOT use this directly for storage paths
    // Can contain path traversal: "../../../../etc/passwd"
    // Always sanitize with FilenameUtils.getName()

    // Content-Type from the part
    @Nullable
    String getContentType();
    // e.g., "image/jpeg"
    // SECURITY: This is client-controlled, not trusted
    // Always verify content by reading bytes

    // Is the file empty?
    boolean isEmpty();
    // true if size == 0 or no file selected

    // File size in bytes
    long getSize();

    // File content as byte array (loads entire file into memory)
    byte[] getBytes() throws IOException;
    // WARNING: for large files, use getInputStream() instead

    // File content as InputStream (streaming)
    InputStream getInputStream() throws IOException;
    // Preferred for large files

    // Spring Resource representation
    default Resource getResource() {
        return new MultipartFileResource(this);
    }

    // Transfer to File on filesystem
    void transferTo(File dest) throws IOException, IllegalStateException;
    void transferTo(Path dest) throws IOException, IllegalStateException;
    // After transferTo(), file is moved/copied to destination
    // Original temp file may be deleted
    // Can only be called once

    // Spring-specific: underlying Part (Servlet 3.0)
    default Part getPart() { return null; }
}
```

---

### @RequestPart vs @RequestParam — The Distinction

```java
@PostMapping("/upload")
public ResponseEntity<String> upload(

    // @RequestParam for MultipartFile:
    @RequestParam("file") MultipartFile file,
    // Resolves using RequestParamMethodArgumentResolver
    // Finds the "file" part in the multipart request
    // Works for: simple file upload

    // @RequestPart for MultipartFile:
    @RequestPart("file") MultipartFile file,
    // Resolves using RequestPartMethodArgumentResolver
    // Works for: file upload AND JSON body parts

    // @RequestParam for String form fields:
    @RequestParam("name") String name,
    // Gets the text value of "name" form field

    // @RequestPart for JSON body part:
    @RequestPart("metadata") ProductMetadata metadata,
    // Deserialises the "metadata" part as JSON
    // Uses HttpMessageConverter (Jackson)
    // Part must have Content-Type: application/json
    // This is what @RequestPart offers beyond @RequestParam

) { ... }

// KEY DIFFERENCE:
// @RequestParam("file") MultipartFile → gets the file bytes
// @RequestPart("data") MyDTO data → deserialises part as JSON/XML
//   via HttpMessageConverter — treats part body like @RequestBody

// BOTH work for MultipartFile — use @RequestParam for simple cases
// Use @RequestPart when the part has its own Content-Type
// and you want full body deserialization
```

---

### Request Mapping for Multipart

```java
// CORRECT: specify consumes for multipart endpoints
@PostMapping(
    value = "/products",
    consumes = MediaType.MULTIPART_FORM_DATA_VALUE
    // "multipart/form-data"
)
public ResponseEntity<Product> createProduct(
        @RequestParam("name") String name,
        @RequestParam("price") BigDecimal price,
        @RequestParam("image") MultipartFile image,
        @RequestParam(value="document", required=false)
            MultipartFile document) {

    // Validate file type (server-side)
    if (!isValidImageType(image)) {
        throw new ResponseStatusException(
            HttpStatus.BAD_REQUEST,
            "Only JPEG and PNG images allowed");
    }

    // Validate file size
    if (image.getSize() > 5 * 1024 * 1024) {
        throw new ResponseStatusException(
            HttpStatus.BAD_REQUEST,
            "Image must be less than 5MB");
    }

    // Process
    Product product = productService.create(
        name, price, image, document);
    return ResponseEntity.status(HttpStatus.CREATED)
        .body(product);
}
```

---

### Multiple File Upload Patterns

```java
@RestController
@RequestMapping("/api/files")
public class FileUploadController {

    // PATTERN 1: Multiple files, same field name
    @PostMapping("/batch")
    public List<String> uploadBatch(
            @RequestParam("files") List<MultipartFile> files) {
        // HTML: multiple <input type="file" name="files">
        // OR: <input type="file" name="files" multiple>
        return files.stream()
            .filter(f -> !f.isEmpty())
            .map(f -> fileService.store(f))
            .collect(Collectors.toList());
    }

    // PATTERN 2: Multiple files, different field names
    @PostMapping("/product-with-media")
    public Product createWithMedia(
            @RequestParam("name") String name,
            @RequestParam("thumbnail") MultipartFile thumbnail,
            @RequestParam("gallery") List<MultipartFile> gallery,
            @RequestParam("manual") MultipartFile manual) {
        return productService.createWithMedia(
            name, thumbnail, gallery, manual);
    }

    // PATTERN 3: MultipartFile array
    @PostMapping("/array-upload")
    public void uploadArray(
            @RequestParam("files") MultipartFile[] files) {
        for (MultipartFile file : files) {
            if (!file.isEmpty()) {
                fileService.store(file);
            }
        }
    }

    // PATTERN 4: Access all parts via MultipartHttpServletRequest
    @PostMapping("/raw")
    public void uploadRaw(
            MultipartHttpServletRequest request) {
        // Access all parts programmatically
        Map<String, MultipartFile> fileMap =
            request.getFileMap();
        // All files keyed by field name

        MultiValueMap<String, MultipartFile> multiFileMap =
            request.getMultiFileMap();
        // Multiple files per field name

        // Iterate all files:
        request.getFileMap().forEach((name, file) -> {
            log.info("Field: {}, File: {}, Size: {}",
                name,
                file.getOriginalFilename(),
                file.getSize());
            fileService.store(file);
        });
    }
}
```

---

### File Storage — Security and Best Practices

```java
@Service
public class FileStorageService {

    private final Path storageLocation;

    public FileStorageService(
            @Value("${upload.dir}") String uploadDir) {
        this.storageLocation = Paths.get(uploadDir)
            .toAbsolutePath()
            .normalize();
    }

    public String store(MultipartFile file) {
        // SECURITY CHECK 1: Empty file
        if (file.isEmpty()) {
            throw new StorageException(
                "Cannot store empty file");
        }

        // SECURITY CHECK 2: Sanitize filename
        // NEVER use getOriginalFilename() directly
        // Path traversal attack: "../../etc/passwd"
        String originalFilename =
            StringUtils.cleanPath(
                file.getOriginalFilename() != null ?
                    file.getOriginalFilename() : "unknown");

        // Remove path components — use filename only
        String safeFilename = FilenameUtils.getName(
            originalFilename);
        // "../../../etc/passwd" → "passwd" (still dangerous)
        // Better: generate random filename

        // SECURITY CHECK 3: Use random filename
        String fileExtension =
            FilenameUtils.getExtension(originalFilename);
        String storedFilename = UUID.randomUUID() + "."
            + fileExtension.toLowerCase();

        // SECURITY CHECK 4: Validate extension
        Set<String> ALLOWED_EXTENSIONS =
            Set.of("jpg", "jpeg", "png", "gif", "pdf", "doc");
        if (!ALLOWED_EXTENSIONS.contains(
                fileExtension.toLowerCase())) {
            throw new StorageException(
                "File type not allowed: " + fileExtension);
        }

        // SECURITY CHECK 5: Verify content type by reading bytes
        // Not trusting Content-Type header from client
        try {
            byte[] header = new byte[8];
            file.getInputStream().read(header);
            if (!isValidFileSignature(header,
                    fileExtension)) {
                throw new StorageException(
                    "File content doesn't match extension");
            }
        } catch (IOException ex) {
            throw new StorageException("File read failed", ex);
        }

        // SECURITY CHECK 6: Store outside web root
        Path targetPath = this.storageLocation
            .resolve(storedFilename)
            .normalize();

        // Ensure path is within storage directory
        if (!targetPath.startsWith(this.storageLocation)) {
            // Path traversal attempted
            throw new StorageException(
                "Cannot store file outside directory");
        }

        // STORE the file
        try {
            Files.copy(file.getInputStream(), targetPath,
                StandardCopyOption.REPLACE_EXISTING);
        } catch (IOException ex) {
            throw new StorageException(
                "Failed to store file", ex);
        }

        return storedFilename;
    }

    // Check magic bytes / file signature
    private boolean isValidFileSignature(byte[] header,
                                           String ext) {
        return switch (ext.toLowerCase()) {
            case "jpg", "jpeg" ->
                header[0] == (byte)0xFF &&
                header[1] == (byte)0xD8 &&
                header[2] == (byte)0xFF;
            case "png" ->
                header[0] == (byte)0x89 &&
                header[1] == 0x50 && // 'P'
                header[2] == 0x4E && // 'N'
                header[3] == 0x47;   // 'G'
            case "pdf" ->
                header[0] == 0x25 && // '%'
                header[1] == 0x50 && // 'P'
                header[2] == 0x44 && // 'D'
                header[3] == 0x46;   // 'F'
            default -> true;
        };
    }
}
```

---

### File Download — Serving Files

```java
@RestController
@RequestMapping("/api/files")
public class FileDownloadController {

    @Autowired
    private FileStorageService fileStorageService;

    // Streaming download — never loads entire file into memory
    @GetMapping("/{filename:.+}")
    public ResponseEntity<Resource> downloadFile(
            @PathVariable String filename,
            HttpServletRequest request) {

        Resource resource =
            fileStorageService.loadAsResource(filename);

        // Determine Content-Type
        String contentType = null;
        try {
            contentType = request.getServletContext()
                .getMimeType(
                    resource.getFile().getAbsolutePath());
        } catch (IOException ex) {
            log.info("Cannot determine content type");
        }

        if (contentType == null) {
            contentType = "application/octet-stream";
        }

        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType(contentType))
            .header(HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=\"" +
                resource.getFilename() + "\"")
            .body(resource);
        // ResourceHttpMessageConverter streams the file
        // Does NOT load entire file into memory
    }

    // Inline display (for images in browser)
    @GetMapping("/{filename:.+}/view")
    public ResponseEntity<Resource> viewFile(
            @PathVariable String filename) {
        Resource resource =
            fileStorageService.loadAsResource(filename);

        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION,
                "inline; filename=\"" +
                resource.getFilename() + "\"")
            // "inline" = display in browser
            // "attachment" = force download
            .body(resource);
    }
}
```

---

### @RequestPart with JSON Body Part

```java
// Mixed upload: file + JSON in same request
@PostMapping(
    value = "/products",
    consumes = MediaType.MULTIPART_FORM_DATA_VALUE
)
public Product createProduct(
        @RequestPart("product") ProductRequest productData,
        // ProductRequest deserialized from JSON part
        // Client sends this part with Content-Type: application/json
        // Jackson deserializes it
        @RequestPart("image") MultipartFile image,
        @RequestPart(value="documents",
                     required=false)
            List<MultipartFile> documents) {

    // productData fully populated by Jackson
    // image is the uploaded file
    return productService.create(productData, image, documents);
}

// Client sends:
// Content-Type: multipart/form-data
//
// Part "product":
//   Content-Type: application/json
//   {"name":"Widget","price":9.99,"categoryId":5}
//
// Part "image":
//   Content-Type: image/jpeg
//   [binary image data]
//
// @RequestPart + Jackson handles deserialization automatically
// @RequestParam("product") String → only gets raw string
// @RequestPart("product") ProductRequest → deserializes JSON
```

---

### Exception Handling for File Uploads

```java
@RestControllerAdvice
public class FileUploadExceptionHandler {

    // File too large (Spring's exception)
    @ExceptionHandler(MaxUploadSizeExceededException.class)
    public ResponseEntity<Map<String, String>> handleMaxSize(
            MaxUploadSizeExceededException ex) {
        return ResponseEntity
            .status(HttpStatus.PAYLOAD_TOO_LARGE)
            .body(Map.of(
                "error", "FILE_TOO_LARGE",
                "message", "File exceeds maximum allowed size",
                "maxSize",
                    ex.getMaxUploadSize() + " bytes"
            ));
    }

    // Multipart parse failure
    @ExceptionHandler(MultipartException.class)
    public ResponseEntity<Map<String, String>> handleMultipart(
            MultipartException ex) {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(Map.of(
                "error", "MULTIPART_ERROR",
                "message", "Failed to parse multipart request",
                "detail", ex.getMessage()
            ));
    }

    // Missing required file part
    @ExceptionHandler(
        MissingServletRequestPartException.class)
    public ResponseEntity<Map<String, String>> handleMissingPart(
            MissingServletRequestPartException ex) {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(Map.of(
                "error", "MISSING_PART",
                "message", "Required part missing: " +
                    ex.getRequestPartName()
            ));
    }
}
```

---

## 2️⃣ CODE EXAMPLES

### Complete File Upload Controller

```java
@RestController
@RequestMapping("/api/v1/files")
@Slf4j
public class FileController {

    @Autowired private FileStorageService storageService;
    @Autowired private FileMetadataRepository metadataRepo;

    @PostMapping(
        consumes = MediaType.MULTIPART_FORM_DATA_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    public ResponseEntity<FileUploadResponse> upload(
            @RequestParam("file") MultipartFile file,
            @RequestParam(value="description",
                          required=false) String description,
            @RequestParam(value="tags",
                          required=false) List<String> tags,
            Principal principal) {

        log.info("Upload: file={}, size={}, type={}",
            file.getOriginalFilename(),
            file.getSize(),
            file.getContentType());

        // Validate
        validateFile(file);

        // Store
        String storedFilename = storageService.store(file);

        // Save metadata
        FileMetadata metadata = FileMetadata.builder()
            .storedFilename(storedFilename)
            .originalFilename(file.getOriginalFilename())
            .contentType(file.getContentType())
            .size(file.getSize())
            .description(description)
            .tags(tags)
            .uploadedBy(principal.getName())
            .uploadedAt(Instant.now())
            .build();
        metadataRepo.save(metadata);

        // Build download URL
        String downloadUrl = "/api/v1/files/" +
            storedFilename;

        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(FileUploadResponse.builder()
                .fileId(metadata.getId())
                .filename(storedFilename)
                .originalFilename(
                    file.getOriginalFilename())
                .size(file.getSize())
                .downloadUrl(downloadUrl)
                .build());
    }

    @PostMapping(
        value = "/batch",
        consumes = MediaType.MULTIPART_FORM_DATA_VALUE
    )
    public List<FileUploadResponse> uploadBatch(
            @RequestParam("files")
                List<MultipartFile> files) {
        return files.stream()
            .filter(f -> !f.isEmpty())
            .map(f -> {
                validateFile(f);
                String stored = storageService.store(f);
                return FileUploadResponse.builder()
                    .filename(stored)
                    .originalFilename(
                        f.getOriginalFilename())
                    .size(f.getSize())
                    .build();
            })
            .collect(Collectors.toList());
    }

    private void validateFile(MultipartFile file) {
        if (file.isEmpty()) {
            throw new ResponseStatusException(
                HttpStatus.BAD_REQUEST, "Empty file");
        }
        if (file.getSize() > 10 * 1024 * 1024) {
            throw new ResponseStatusException(
                HttpStatus.PAYLOAD_TOO_LARGE,
                "File exceeds 10MB limit");
        }
        String ct = file.getContentType();
        Set<String> ALLOWED = Set.of(
            "image/jpeg", "image/png", "image/gif",
            "application/pdf", "text/plain");
        if (ct == null || !ALLOWED.contains(ct)) {
            throw new ResponseStatusException(
                HttpStatus.UNSUPPORTED_MEDIA_TYPE,
                "Content type not allowed: " + ct);
        }
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

**Q1 — MCQ**
`StandardServletMultipartResolver` bean is named `"fileUploadResolver"` instead of `"multipartResolver"`. What happens when a multipart request arrives?

A) The resolver is found by type — bean name doesn't matter
B) Spring Boot auto-detects the resolver regardless of name
C) `DispatcherServlet.initMultipartResolver()` looks up by name `"multipartResolver"` — not found → `multipartResolver` field is null → multipart NOT parsed → handler receives unparsed request
D) `IllegalStateException` at startup — wrong bean name

**Answer: C**
`initMultipartResolver()` uses `context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class)` where `MULTIPART_RESOLVER_BEAN_NAME = "multipartResolver"`. Wrong name → `NoSuchBeanDefinitionException` caught silently → `multipartResolver = null` → `checkMultipart()` returns original request → parts not accessible.

---

**Q2 — Select All That Apply**
Which are TRUE about `MultipartFile.getOriginalFilename()`?

A) It returns the filename exactly as provided by the client browser
B) It can contain path traversal sequences like `../../etc/passwd`
C) It is safe to use directly as a storage path component
D) It should be sanitized with `FilenameUtils.getName()` or `StringUtils.cleanPath()`
E) It always returns a non-null value

**Answer: A, B, D**
A: yes — whatever the browser sent. B: yes — malicious clients can inject path traversal. C: absolutely NOT — security vulnerability. D: yes — sanitization is mandatory. E: wrong — can be null if filename was not provided (file input without selection).

---

**Q3 — MCQ**
`@RequestPart("data") ProductDTO data` is declared as a parameter. The multipart part "data" has `Content-Type: application/json`. How is `data` populated?

A) Spring reads the part as a String and parses it manually
B) `RequestPartMethodArgumentResolver` reads the part and uses `HttpMessageConverter` (Jackson) to deserialize JSON → `ProductDTO`
C) `@RequestParam` resolver handles it — `@RequestPart` and `@RequestParam` behave identically
D) `WebDataBinder` binds form fields to `ProductDTO` properties

**Answer: B**
`RequestPartMethodArgumentResolver` reads the part's content and its `Content-Type`. Since `Content-Type: application/json`, it uses the `HttpMessageConverter` chain (same as `@RequestBody`) to deserialize. `MappingJackson2HttpMessageConverter` converts JSON bytes → `ProductDTO`. This is the key difference from `@RequestParam` which does form-field binding.

---

**Q4 — True/False**
`multipartResolver.cleanupMultipart()` is called only when file upload processing succeeds without exceptions.

**Answer: False**
`cleanupMultipart()` is called in `doDispatch()`'s `finally` block — it runs regardless of whether the request succeeded, threw an exception, or was rejected by an interceptor. Temp file cleanup is always guaranteed.

---

**Q5 — MCQ**
`spring.servlet.multipart.max-file-size=5MB` is set. A client uploads a 10MB file. What exception is thrown and what HTTP status is returned?

A) `IOException` → 500 Internal Server Error
B) `MaxUploadSizeExceededException` → typically 500 unless handled
C) `MultipartException` → 400 Bad Request
D) `FileSizeLimitExceededException` → 413 Payload Too Large automatically

**Answer: B**
`MaxUploadSizeExceededException` is thrown by the multipart resolver when size limit is exceeded. `DefaultHandlerExceptionResolver` does NOT handle this exception → it propagates as 500 unless a custom `@ExceptionHandler(MaxUploadSizeExceededException.class)` returns 413. Spring Boot's default error handling may return 500. Must add explicit handler for proper 413 response.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `multipartResolver` bean name is mandatory**
Only `MultipartResolver` is found by name. Every other special bean is found by type. Wrong name → silent failure → multipart not parsed. Always use `@Bean(name = "multipartResolver")`.

**Trap 2 — `getOriginalFilename()` is a security vulnerability if used raw**
Never use `getOriginalFilename()` directly as a file path. Always sanitize. Generate a UUID-based filename. Verify file content via magic bytes, not just extension or Content-Type header.

**Trap 3 — `transferTo()` can only be called once**
After `MultipartFile.transferTo(destination)`, the underlying temp file may be moved or deleted. Calling it again throws `IllegalStateException`. Read content with `getInputStream()` if you need to process AND save.

**Trap 4 — `Content-Type` from client is untrusted**
`file.getContentType()` returns whatever the client sent. A malicious client can send `Content-Type: image/jpeg` for an executable file. Always verify by reading magic bytes from the actual file content.

**Trap 5 — `MaxUploadSizeExceededException` needs explicit handler for 413**
Spring does not automatically return 413 for oversized uploads. Without `@ExceptionHandler(MaxUploadSizeExceededException.class)`, the response is 500. Always add an explicit handler returning `HttpStatus.PAYLOAD_TOO_LARGE`.

---

## 5️⃣ SUMMARY SHEET

```
MULTIPART PIPELINE
─────────────────────────────────────────────────────
checkMultipart() → resolveMultipart() → handler → cleanupMultipart()
cleanupMultipart() in finally → ALWAYS runs (temp file deletion)

MULTIPARTRESOLVER TYPES
─────────────────────────────────────────────────────
StandardServletMultipartResolver (PREFERRED):
  Bean name: "multipartResolver" (mandatory)
  Requires: MultipartConfigElement on DispatcherServlet
  Spring Boot: auto-configured

CommonsMultipartResolver (legacy):
  Requires: commons-fileupload dependency
  More configuration options

KEY SIZE LIMITS (Spring Boot properties)
─────────────────────────────────────────────────────
spring.servlet.multipart.max-file-size=1MB (default)
spring.servlet.multipart.max-request-size=10MB (default)
spring.servlet.multipart.file-size-threshold=0
  0 = write to disk immediately

@REQUESTPARAM vs @REQUESTPART
─────────────────────────────────────────────────────
@RequestParam MultipartFile: form-field binding, simple file
@RequestPart MultipartFile: same + checks part Content-Type
@RequestPart MyDTO: deserializes part via HttpMessageConverter
  Part must have Content-Type: application/json

MULTIPARTFILE SECURITY RULES
─────────────────────────────────────────────────────
1. Never use getOriginalFilename() as storage path
2. Sanitize: FilenameUtils.getName() + UUID rename
3. Validate extension whitelist
4. Verify content via magic bytes (not Content-Type header)
5. Store outside web root
6. Check path traversal after resolving

EXCEPTION HANDLING
─────────────────────────────────────────────────────
MaxUploadSizeExceededException → 500 by default
  Add @ExceptionHandler → return 413 PAYLOAD_TOO_LARGE
MultipartException → 400 Bad Request
MissingServletRequestPartException → 400 Bad Request

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "multipartResolver bean name must be exact — found by name not type"
• "cleanupMultipart() in finally — always runs, even on exception"
• "getOriginalFilename() is client-controlled — never use as storage path"
• "@RequestPart enables HttpMessageConverter-based deserialization of parts"
• "MaxUploadSizeExceededException → 500 unless explicitly handled as 413"
• "transferTo() once only — temp file may be deleted after"
• "Content-Type from client is untrusted — verify via magic bytes"
```

---

---

# TOPIC 5.4 — ASYNC PROCESSING: Callable, DeferredResult, WebAsyncTask

---

## 1️⃣ CONCEPTUAL EXPLANATION

### Why Async Processing Exists

Spring MVC's default thread model: one Servlet container thread per request, held for the entire duration. For long-running operations (slow DB queries, external API calls, message queue polling), this means:

```
Tomcat thread pool: 200 threads (default)
200 simultaneous slow requests → ALL threads occupied
Request 201 arrives → QUEUED (or rejected)
Each slow request wastes an expensive thread

ASYNC PROCESSING solution:
Request arrives → Thread A starts processing
Handler returns Callable/DeferredResult → Thread A RELEASED
Async thread/callback completes work → result ready
New thread B dispatches result → response written
Thread pool never saturated by waiting threads
```

---

### Servlet 3.0 Async — The Foundation

```java
// Servlet 3.0 AsyncContext — the underlying mechanism
// Spring's async support builds on this

// Manual Servlet 3.0 async (without Spring):
AsyncContext asyncContext = request.startAsync();
asyncContext.setTimeout(30000); // 30 seconds

ExecutorService executor = ...;
executor.submit(() -> {
    // Worker thread
    String result = computeExpensiveResult();

    // Must dispatch back to container thread
    asyncContext.dispatch("/result");
    // OR write directly:
    asyncContext.getResponse().getWriter().write(result);
    asyncContext.complete();
});
// Original thread is RELEASED immediately after startAsync()

// Spring's async abstractions (Callable, DeferredResult)
// hide this complexity behind clean interfaces
```

---

### Callable<T> — Thread Pool Based Async

```java
// Callable: Spring submits to configured task executor
// Simple async — just wrap in Callable

@GetMapping("/reports/generate")
public Callable<Report> generateReport(
        @RequestParam String type) {

    // This method returns IMMEDIATELY
    // Spring sees Callable → starts async processing

    return () -> {
        // This block runs on TaskExecutor thread pool
        // NOT on Servlet container thread
        Thread.sleep(5000); // Simulate slow work
        return reportService.generate(type);
        // When this returns, Spring dispatches result
        // back to Servlet container thread
        // ResponseBody serialization happens there
    };
}

// INTERNAL FLOW:
// 1. Servlet thread calls controller method
// 2. Method returns Callable immediately
// 3. CallableMethodReturnValueHandler.handleReturnValue()
// 4. WebAsyncManager.startCallableProcessing(callable)
//    → Creates AsyncContext via request.startAsync()
//    → Submits Callable to TaskExecutor
//    → Sets asyncManager.concurrentHandlingStarted=true
// 5. doDispatch() Phase 7: isConcurrentHandlingStarted()=true → return
// 6. Servlet thread released back to pool
// 7. TaskExecutor thread executes the Callable
// 8. Result stored in asyncManager
// 9. asyncContext.dispatch() → new request dispatch
// 10. Second doDispatch() call on new Servlet thread
//     → asyncManager.hasConcurrentResult()=true
//     → Result processed as return value
//     → @ResponseBody serialization → response written

// TASK EXECUTOR CONFIGURATION:
@Bean
public AsyncTaskExecutor mvcTaskExecutor() {
    ThreadPoolTaskExecutor executor =
        new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(50);
    executor.setQueueCapacity(100);
    executor.setThreadNamePrefix("mvc-async-");
    executor.initialize();
    return executor;
}

@Override
public void configureAsyncSupport(
        AsyncSupportConfigurer configurer) {
    configurer.setTaskExecutor(mvcTaskExecutor());
    configurer.setDefaultTimeout(30_000); // 30 seconds
}
```

---

### DeferredResult<T> — External Completion

```java
// DeferredResult: result set externally (event, callback, etc.)
// No thread pool — no thread holds the request at all

@GetMapping("/notifications/poll")
public DeferredResult<List<Notification>> longPoll(
        Principal principal) {

    // Create DeferredResult with timeout
    DeferredResult<List<Notification>> deferredResult =
        new DeferredResult<>(
            30_000L,           // timeout: 30 seconds
            Collections.emptyList()  // timeout result value
        );

    // Register callbacks
    deferredResult.onTimeout(() ->
        log.debug("Long poll timed out for {}",
            principal.getName()));

    deferredResult.onError(ex ->
        log.error("Long poll error for {}",
            principal.getName(), ex));

    deferredResult.onCompletion(() ->
        notificationService.unregister(
            principal.getName(), deferredResult));

    // Register with notification service
    // (event-driven: set result when event arrives)
    notificationService.register(
        principal.getName(), deferredResult);

    return deferredResult;
    // Servlet thread released immediately
    // No thread holds this request
    // When event arrives: notificationService calls
    //   deferredResult.setResult(notifications)
    // → triggers async dispatch → response written
}

// In notification service:
@Service
public class NotificationService {

    private final Map<String, DeferredResult<List<Notification>>>
        pendingResults = new ConcurrentHashMap<>();

    public void register(String userId,
                          DeferredResult<List<Notification>> result) {
        pendingResults.put(userId, result);
    }

    public void unregister(String userId,
                            DeferredResult<?> result) {
        pendingResults.remove(userId);
    }

    // Called when new notification arrives (e.g., from message broker)
    @EventListener
    public void onNotification(NotificationEvent event) {
        DeferredResult<List<Notification>> result =
            pendingResults.remove(event.getUserId());
        if (result != null) {
            result.setResult(
                List.of(event.getNotification()));
            // This triggers async dispatch
        }
    }
}
```

---

### WebAsyncTask<T> — Callable with Lifecycle Hooks

```java
// WebAsyncTask = Callable + timeout callback + error callback
// More control than raw Callable

@GetMapping("/data/expensive")
public WebAsyncTask<Data> getExpensiveData() {

    Callable<Data> callable = () -> {
        // This runs on task executor thread
        return dataService.computeExpensive();
    };

    WebAsyncTask<Data> asyncTask =
        new WebAsyncTask<>(10_000L, callable);
    // 10 second timeout

    asyncTask.onTimeout(() -> {
        // Called when timeout expires
        // Return value becomes the response
        // (or throw exception for error response)
        log.warn("Request timed out");
        return Data.empty(); // fallback data
        // OR: throw new AsyncRequestTimeoutException()
        // → 503 Service Unavailable
    });

    asyncTask.onError(ex -> {
        log.error("Async task failed", ex);
        throw new ResponseStatusException(
            HttpStatus.INTERNAL_SERVER_ERROR);
    });

    asyncTask.onCompletion(() ->
        log.debug("Async task completed"));

    return asyncTask;
}
```

---

### CompletableFuture<T> — Java 8+ Integration

```java
// DeferredResultMethodReturnValueHandler handles CompletableFuture
// Adapts to DeferredResult internally

@GetMapping("/external/data")
public CompletableFuture<ExternalData> getExternal() {
    // Non-blocking external API call
    return externalApiClient
        .getData()
        .thenApply(response ->
            mapper.toExternalData(response));
    // CompletableFuture.thenApply() = transformation
    // Spring adapts to DeferredResult
    // No thread held waiting for external API
}

// Using @Async service:
@GetMapping("/combined")
public CompletableFuture<CombinedResult> getCombined() {
    CompletableFuture<DataA> futureA =
        asyncService.getDataA(); // @Async method
    CompletableFuture<DataB> futureB =
        asyncService.getDataB(); // @Async method

    // Both run in parallel on @Async executor
    return futureA.thenCombine(futureB,
        (a, b) -> CombinedResult.of(a, b));
    // Completes when BOTH are done
    // Neither thread holds the Servlet thread
}
```

---

### Reactive Type Support — Mono<T> and Flux<T>

```java
// Spring MVC 5+ supports Reactor types
// Adapted via ReactiveTypeReturnValueHandler

@GetMapping("/products/{id}")
public Mono<Product> getProduct(@PathVariable Long id) {
    return webClient.get()
        .uri("/internal/products/{id}", id)
        .retrieve()
        .bodyToMono(Product.class);
    // Adapted to DeferredResult
    // Non-blocking HTTP call
}

// Flux as SSE streaming:
@GetMapping(value="/stream",
            produces=MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<Product> stream() {
    return Flux.interval(Duration.ofSeconds(1))
        .flatMap(i -> productRepo.findRandom())
        .take(100); // 100 products then complete
    // Routes to ResponseBodyEmitter (SseEmitter) path
    // Each Flux element emitted as SSE event
}
```

---

### AsyncHandlerInterceptor — Async-Aware Interceptors

```java
public class AsyncAwareInterceptor
    implements AsyncHandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) {
        // Called on FIRST dispatch AND SECOND dispatch
        // Check DispatcherType to distinguish:
        if (DispatcherType.ASYNC.equals(
                request.getDispatcherType())) {
            // Second dispatch (async result ready)
            log.debug("Async result dispatch");
            return true;
        }
        // First dispatch
        request.setAttribute("startTime",
            System.nanoTime());
        return true;
    }

    @Override
    public void afterConcurrentHandlingStarted(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) {
        // Called when async STARTS (first dispatch exits)
        // Use for cleanup that normally goes in postHandle
        // postHandle NOT called on first async dispatch
        log.debug("Async started for: {}",
            request.getRequestURI());
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                  HttpServletResponse response,
                                  Object handler,
                                  Exception ex) {
        // Called on SECOND dispatch completion
        // (after async result is fully processed)
        Long start = (Long) request.getAttribute("startTime");
        if (start != null) {
            long elapsed = System.nanoTime() - start;
            log.info("Total time (including async): {}ms",
                TimeUnit.NANOSECONDS.toMillis(elapsed));
        }
    }
}
```

---

## 2️⃣ CODE EXAMPLES

### Complete Async Controller Patterns

```java
@RestController
@RequestMapping("/api/async")
public class AsyncController {

    @Autowired
    private ReportService reportService;

    @Autowired
    private EventBus eventBus;

    // PATTERN 1: CPU-intensive work on thread pool
    @GetMapping("/report/{id}")
    public Callable<Report> generateReport(
            @PathVariable Long id) {
        return () -> reportService.generate(id);
    }

    // PATTERN 2: Long-polling / push notification
    @GetMapping("/events/{userId}")
    public DeferredResult<Event> waitForEvent(
            @PathVariable String userId) {
        DeferredResult<Event> result =
            new DeferredResult<>(15_000L);
        eventBus.subscribe(userId, event -> {
            result.setResult(event);
        });
        result.onCompletion(() ->
            eventBus.unsubscribe(userId));
        return result;
    }

    // PATTERN 3: Timeout with fallback
    @GetMapping("/data/with-timeout")
    public WebAsyncTask<Data> withTimeout() {
        return new WebAsyncTask<>(5000L,
            () -> dataService.fetch(),
            () -> Data.fallback()); // timeout supplier
    }

    // PATTERN 4: Multiple parallel async calls
    @GetMapping("/aggregated")
    public CompletableFuture<AggregatedResult> aggregated() {
        return CompletableFuture.allOf(
            serviceA.getAsync(),
            serviceB.getAsync(),
            serviceC.getAsync()
        ).thenApply(v -> AggregatedResult.of(
            serviceA.getAsync().join(),
            serviceB.getAsync().join(),
            serviceC.getAsync().join()
        ));
    }
}
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

**Q1 — MCQ**
A controller returns `Callable<Product>`. After `ha.handle()` in Phase 6 of `doDispatch()`, what is the value of `mavContainer.isRequestHandled()`?

A) `true` — response was written asynchronously
B) `false` — Callable just started processing, result not yet available
C) `null` — mavContainer not yet populated
D) `true` — CallableMethodReturnValueHandler marks it handled

**Answer: B**
`CallableMethodReturnValueHandler.handleReturnValue()` calls `asyncManager.startCallableProcessing()` which starts the async work. It does NOT set `mavContainer.requestHandled=true`. `ha.handle()` returns `null` (no ModelAndView). Phase 7 detects `isConcurrentHandlingStarted()=true` and exits `doDispatch()`. On the second dispatch (async result), the actual return value is processed.

---

**Q2 — Select All That Apply**
Which are TRUE about `DeferredResult`?

A) It uses a thread pool to execute the work
B) The result can be set from any thread at any time
C) `setResult()` triggers an async dispatch to DispatcherServlet
D) Timeout can be configured with a default value returned on timeout
E) `setErrorResult()` causes the exception to propagate to `@ExceptionHandler`

**Answer: B, C, D, E**
A is wrong — DeferredResult uses NO thread pool. The result is set externally (event listener, callback, another thread). B: any thread can call `setResult()`. C: `setResult()` internally calls `asyncContext.dispatch()`. D: `new DeferredResult<>(timeout, defaultValue)`. E: `setErrorResult(exception)` → exception propagates through Spring's exception handling chain.

---

**Q3 — Ordering**
For a `Callable<T>` return value, order the full async lifecycle:

- Second `doDispatch()` call processes result
- `CallableMethodReturnValueHandler` submits Callable to executor
- `asyncManager.isConcurrentHandlingStarted()=true` → doDispatch exits
- Response written with JSON result
- `afterConcurrentHandlingStarted()` called on interceptors
- Worker thread executes Callable, stores result
- `preHandle()` called on second dispatch

**Correct Order:**
1. `CallableMethodReturnValueHandler` submits Callable to executor
2. `asyncManager.isConcurrentHandlingStarted()=true` → doDispatch exits
3. `afterConcurrentHandlingStarted()` called on interceptors
4. Worker thread executes Callable, stores result
5. Second `doDispatch()` call processes result
6. `preHandle()` called on second dispatch
7. Response written with JSON result

---

**Q4 — True/False**
`AsyncHandlerInterceptor.afterConcurrentHandlingStarted()` is called AFTER the async result is complete.

**Answer: False**
`afterConcurrentHandlingStarted()` is called when async processing STARTS — on the FIRST dispatch, just before the Servlet thread is released. It is the replacement for `postHandle()` on the first dispatch (since `postHandle()` is not called when async starts). `afterCompletion()` is called on the second dispatch after the result is processed and response is written.

---

**Q5 — MCQ**
`configureAsyncSupport().setDefaultTimeout(30_000L)` is set. A `Callable<Product>` does not complete within 30 seconds. What happens?

A) `AsyncRequestTimeoutException` thrown → 503 if handled, 500 if not
B) The Callable is cancelled and 408 Request Timeout returned automatically
C) The thread is interrupted and the response is committed empty
D) Spring retries the Callable with fresh timeout

**Answer: A**
After timeout: Spring MVC sends `AsyncRequestTimeoutException`. `DefaultHandlerExceptionResolver` maps this to HTTP 503 Service Unavailable. If you have `@ExceptionHandler(AsyncRequestTimeoutException.class)`, you can return a different response. Without handling: 503 from `DefaultHandlerExceptionResolver`.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `preHandle()` called TWICE for async requests**
Interceptors' `preHandle()` is called on BOTH the first dispatch (before async starts) and the second dispatch (when result arrives). Code with side effects in `preHandle()` (rate limiting, counter increment) runs twice. Check `request.getDispatcherType() == DispatcherType.ASYNC` to skip on second pass.

**Trap 2 — `postHandle()` NOT called on first async dispatch**
`postHandle()` is skipped when `asyncManager.isConcurrentHandlingStarted()=true`. Interceptors relying on `postHandle()` for cleanup (logging response, modifying model) must implement `AsyncHandlerInterceptor.afterConcurrentHandlingStarted()` for async-aware cleanup.

**Trap 3 — DeferredResult result-setting thread safety**
`setResult()` can be called from ANY thread. If the DeferredResult has already timed out or been completed when `setResult()` is called, the call is silently ignored. Always check the return value of `setResult()` — it returns `false` if the result was not set (already completed/timed out).

**Trap 4 — Task executor must be configured for Callable**
Without configuring a `TaskExecutor` via `configureAsyncSupport()`, Spring uses `SimpleAsyncTaskExecutor` which creates a **new thread per request** — not a pool. This can exhaust system threads under load. Always configure `ThreadPoolTaskExecutor`.

**Trap 5 — Callable vs CompletableFuture — different thread management**
`Callable<T>` → Spring submits to configured `TaskExecutor` (controlled thread pool).
`CompletableFuture<T>` → runs on whatever executor the CompletableFuture uses (`ForkJoinPool.commonPool()` by default for `supplyAsync()` without explicit executor). Uncontrolled thread usage. Always specify an executor with `CompletableFuture.supplyAsync(task, myExecutor)`.

---

## 5️⃣ SUMMARY SHEET

```
ASYNC PROCESSING — WHY
─────────────────────────────────────────────────────
Thread-per-request: slow requests hold Servlet threads
Async: release Servlet thread → process on worker thread
→ More concurrent requests with same thread pool

THREE ASYNC MECHANISMS
─────────────────────────────────────────────────────
Callable<T>:
  → Spring submits to configured TaskExecutor
  → Simple wrapper around blocking operation
  → Default executor: SimpleAsyncTaskExecutor (thread per call!)
  → Always configure ThreadPoolTaskExecutor

DeferredResult<T>:
  → No thread pool — result set externally
  → Event-driven: set from callback/event listener
  → new DeferredResult<>(timeout, defaultValue)
  → setResult() → triggers async dispatch
  → setErrorResult(ex) → propagates to @ExceptionHandler

WebAsyncTask<T>:
  → Callable + timeout callback + error callback
  → onTimeout() → return fallback value or throw
  → Most control over lifecycle

CompletableFuture<T>:
  → Adapted to DeferredResult
  → Uses its own executor (configure explicitly!)

INTERCEPTOR ASYNC LIFECYCLE
─────────────────────────────────────────────────────
First dispatch (async starts):
  preHandle() ← called
  [handler returns Callable/DeferredResult]
  postHandle() ← NOT called (async started)
  afterConcurrentHandlingStarted() ← called
  [Servlet thread released]

Second dispatch (result ready):
  preHandle() ← called AGAIN
  [result processed as return value]
  postHandle() ← called
  afterCompletion() ← called

TIMEOUT HANDLING
─────────────────────────────────────────────────────
configureAsyncSupport().setDefaultTimeout(30_000L)
DeferredResult(timeout): per-result timeout
WebAsyncTask(timeout, callable): per-task timeout
Timeout → AsyncRequestTimeoutException → 503 (DefaultHandlerExceptionResolver)

DOUBLE PREHANDLE TRAP
─────────────────────────────────────────────────────
preHandle() called twice: first dispatch + second dispatch
Check: DispatcherType.ASYNC to skip second pass
Rate limiters, counters → would double-count

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "Callable: Spring submits to TaskExecutor — always configure ThreadPoolTaskExecutor"
• "DeferredResult: no thread pool — result set from any external thread"
• "preHandle() called TWICE for async — check DispatcherType.ASYNC"
• "postHandle() NOT called on first async dispatch — use afterConcurrentHandlingStarted()"
• "AsyncRequestTimeoutException → 503 via DefaultHandlerExceptionResolver"
• "CompletableFuture uses ForkJoinPool by default — specify executor explicitly"
• "DeferredResult.setResult() returns false if already completed — check return"
• "WebAsyncTask = Callable + timeout/error callbacks — most lifecycle control"
```

---

