# TOPIC 3.5 — DATA BINDING: @ModelAttribute, WebDataBinder, VALIDATION

---

## 1️⃣ CONCEPTUAL EXPLANATION

### What Data Binding Really Is

Data binding is the process of **converting and populating a Java object from HTTP request data** — typically form parameters, query strings, or path variables. It is not just setting field values. It involves: finding or creating the target object, applying type conversion, handling nested objects, managing collections, running validation, recording errors, and exposing results to the view. Understanding the full pipeline is what separates surface-level knowledge from mastery.

---

### The Data Binding Architecture

```
HTTP Request Parameters (Strings)
        │
        ▼
WebDataBinder
  ├── DataBinder (base — field access + type conversion)
  ├── WebDataBinder (adds web-specific: field markers, HTML checkbox)
  └── ServletRequestDataBinder (adds servlet request binding)
        │
        ├── ConversionService (String → target type)
        ├── PropertyEditors (legacy type conversion)
        ├── Validator chain (JSR-303, custom)
        └── BindingResult (errors container)
        │
        ▼
Populated Java Object + BindingResult
        │
        ▼
Model (stored for view rendering)
```

---

### @ModelAttribute — Complete Internal Flow

```java
// Complete @ModelAttribute resolution flow:
// ServletModelAttributeMethodProcessor.resolveArgument()

@Override
@Nullable
public final Object resolveArgument(
        MethodParameter parameter,
        @Nullable ModelAndViewContainer mavContainer,
        NativeWebRequest webRequest,
        @Nullable WebDataBinderFactory binderFactory)
        throws Exception {

    Assert.state(mavContainer != null, "...");

    // STEP 1: Get attribute name
    String name = ModelFactory.getNameForParameter(parameter);
    // → Uses annotation value if @ModelAttribute("name")
    // → Otherwise derives from type name: "product" from Product

    ModelAttribute ann =
        parameter.getParameterAnnotation(ModelAttribute.class);

    if (ann != null) {
        mavContainer.setBinding(name, ann.binding());
        // binding=false → disable binding (model attribute set
        //   by @ModelAttribute method, no request param binding)
    }

    Object attribute = null;
    BindingResult bindingResult = null;

    // STEP 2: Look for existing attribute in model
    if (mavContainer.containsAttribute(name)) {
        attribute = mavContainer.getModel().get(name);
        // Found in model — possibly set by @ModelAttribute method
    }

    // STEP 3: Look in @SessionAttributes if not in model
    if (attribute == null) {
        attribute = sessionAttributesHandler
            .retrieveAttribute(webRequest, name);
        // From HttpSession — if @SessionAttributes("name") on controller
        if (attribute != null) {
            mavContainer.addAttribute(name, attribute);
        }
    }

    // STEP 4: Create new instance if not found anywhere
    if (attribute == null) {
        attribute = createAttribute(name, parameter,
            binderFactory, webRequest);
        // → Default constructor: new Product()
        // → OR factory method (see below)
    }

    // STEP 5: Bind request parameters
    if (bindingDisabled(name, mavContainer)) {
        // binding=false → skip binding
        bindingResult = binderFactory
            .createBinder(webRequest, attribute, name)
            .getBindingResult();
    } else {
        // Create WebDataBinder
        WebDataBinder binder =
            binderFactory.createBinder(
                webRequest, attribute, name);
        // → Applies @InitBinder methods

        if (binder.getTarget() != null) {
            // BIND: maps request params to object fields
            bindRequestParameters(binder, webRequest);
            // → binder.bind(request parameters)

            // VALIDATE: if @Valid or @Validated present
            validateIfApplicable(binder, parameter);
            // → Runs JSR-303 + custom validators
            // → Errors stored in binder.getBindingResult()

            if (binder.getBindingResult().hasErrors() &&
                    isBindExceptionRequired(binder, parameter)) {
                // BindingResult NOT present as next parameter
                // → Throw exception
                throw new BindException(
                    binder.getBindingResult());
            }
        }

        // STEP 6: Store binding result
        Map<String, Object> bindingResultModel =
            binder.getBindingResult()
                .getModel();
        // Returns: {
        //   "product": <product instance>,
        //   "org.springframework.validation.BindingResult.product":
        //       <BindingResult>
        // }
        mavContainer.removeAttributes(bindingResultModel);
        mavContainer.addAllAttributes(bindingResultModel);
        bindingResult = binder.getBindingResult();
    }

    return binder.convertIfNecessary(
        binder.getTarget(),
        parameter.getParameterType(),
        parameter);
}
```

---

### WebDataBinder — Architecture and Capabilities

```java
// WebDataBinder hierarchy:
DataBinder
    └── WebDataBinder
            └── ExtendedServletRequestDataBinder (actual used in Spring MVC)

// Key fields:
public class DataBinder {
    private Object target;          // The object being bound
    private String objectName;      // e.g., "product"

    // ALLOWED/DISALLOWED fields — security controls
    private String[] allowedFields;     // Only these fields bound
    private String[] disallowedFields;  // Never bound
    private String[] requiredFields;    // Must be present, else FieldError

    // Type conversion
    private ConversionService conversionService;

    // Legacy type conversion
    private PropertyEditorRegistry propertyEditorRegistry;

    // Error collection
    private AbstractPropertyBindingResult bindingResult;

    // Validators
    private List<Validator> validators = new ArrayList<>();

    // Auto-grow collections?
    private boolean autoGrowNestedPaths = true;

    // Nested path auto-growth limit
    private int autoGrowCollectionLimit = DEFAULT_AUTO_GROW_COLLECTION_LIMIT;
    // Default: 256 elements — prevents OutOfMemoryError from malicious input
}
```

---

### Binding Process — How Parameters Map to Fields

```java
// ExtendedServletRequestDataBinder.bind()
@Override
public void bind(ServletRequest request) {
    MutablePropertyValues mpvs =
        new ServletRequestParameterPropertyValues(request);
    // Extracts all request params as MutablePropertyValues

    MultipartRequest multipartRequest =
        WebUtils.getNativeRequest(
            request, MultipartRequest.class);
    if (multipartRequest != null) {
        bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
    }

    addBindValues(mpvs, request);
    // Adds special values: "_fieldname" checkboxes, etc.

    doBind(mpvs);
}

protected void doBind(MutablePropertyValues mpvs) {
    // Apply allowed/disallowed field filtering
    checkAllowedFields(mpvs);
    checkRequiredFields(mpvs);
    applyPropertyValues(mpvs);
}

protected void applyPropertyValues(MutablePropertyValues mpvs) {
    try {
        // BeanWrapper handles the actual field setting
        getPropertyAccessor().setPropertyValues(
            mpvs, isIgnoreUnknownFields(),
            isIgnoreInvalidFields());
        // ignoreUnknownFields=true by default:
        //   request param "nonExistentField" silently ignored
        // ignoreInvalidFields=false by default:
        //   type conversion failure → FieldError in BindingResult
    } catch (PropertyBatchUpdateException ex) {
        for (PropertyAccessException pae : ex.getPropertyAccessExceptions()) {
            getBindingResult().addError(toFieldError(pae));
        }
    }
}
```

**Field name → property mapping rules:**

```
HTTP param "name"          → product.setName(value)
HTTP param "price"         → product.setPrice(Double.parseDouble(value))
HTTP param "address.city"  → product.getAddress().setCity(value)
HTTP param "tags[0]"       → product.getTags().set(0, value)
HTTP param "tags[]"        → adds to product.getTags() (list append)
HTTP param "metadata[key]" → product.getMetadata().put("key", value)

NESTED OBJECTS:
  "address.city" splits on "."
  → BeanWrapper.setPropertyValue("address.city", "London")
  → If address is null → auto-creates Address (autoGrowNestedPaths=true)

COLLECTIONS:
  "tags[0]", "tags[1]" → List indexed access
  → If list is null → auto-creates ArrayList
  → autoGrowCollectionLimit=256 prevents OOM from "tags[9999999]"

MAPS:
  "metadata[key]" → Map key access
  → If map is null → auto-creates LinkedHashMap
```

---

### Type Conversion — The Full Chain

```
String parameter value
        │
        ▼
ConversionService.convert(value, targetType)
        │
        ├── Generic converters (Collections, Arrays, Maps)
        ├── Scalar converters:
        │     String → Integer, Long, Double, Boolean
        │     String → Enum (via name)
        │     String → Date (requires @DateTimeFormat or custom converter)
        │     String → LocalDate (requires @DateTimeFormat)
        │
        ├── Custom Converter<S,T> implementations
        │
        └── Fallback: PropertyEditor
              SimpleTypeConverter.convertIfNecessary()
              CustomDateEditor, etc. (legacy mechanism)

TYPE CONVERSION FAILURE:
  → FieldError added to BindingResult
  → codes: ["typeMismatch.product.price", "typeMismatch.price",
             "typeMismatch.java.lang.Double", "typeMismatch"]
  → message: "Failed to convert property value of type 'String'
              to required type 'java.lang.Double'"
```

---

### @InitBinder — Complete Analysis

```java
// @InitBinder customises WebDataBinder BEFORE binding occurs
// Scope: applies to all @ModelAttribute parameters in the controller
//        or only to named attribute (if name specified)

@Controller
public class ProductController {

    // Global @InitBinder — applies to ALL model attributes
    @InitBinder
    public void globalBinder(WebDataBinder binder) {
        // Security: only allow specific fields
        binder.setAllowedFields(
            "name", "price", "description", "category");

        // Custom PropertyEditor for legacy date format
        binder.registerCustomEditor(
            Date.class,
            new CustomDateEditor(
                new SimpleDateFormat("dd/MM/yyyy"),
                false  // allowEmpty
            )
        );

        // Custom Converter (preferred over PropertyEditor)
        binder.addCustomFormatter(
            new DateTimeFormatterRegistrar()
                .getFormatter(LocalDate.class));
    }

    // Scoped @InitBinder — ONLY for "product" model attribute
    @InitBinder("product")
    public void productBinder(WebDataBinder binder) {
        binder.setDisallowedFields("id", "createdAt", "version");
        // Prevents mass assignment: POST body cannot set "id"
        // Even if "id" is in request params → silently ignored
    }

    // @InitBinder — custom validator
    @InitBinder
    public void registerValidator(WebDataBinder binder) {
        binder.setValidator(new ProductValidator());
        // REPLACES default validator
        // Use addValidators() to ADD alongside JSR-303
    }

    @PostMapping("/products")
    public String create(
            @Valid @ModelAttribute("product") Product product,
            BindingResult result) { ... }
}
```

**@InitBinder execution order:**
```
1. @ControllerAdvice @InitBinder methods (global — in @Order order)
2. Controller-local @InitBinder methods (applied per attribute name)
3. WebBindingInitializer.initBinder() (if configured globally)
```

**@InitBinder parameter types supported:**
```
WebDataBinder (required — the binder to configure)
WebRequest / HttpServletRequest / ServletRequest
Locale
```

---

### WebBindingInitializer — Global Binder Configuration

```java
// Applies to ALL bindings across ALL controllers
// Set on RequestMappingHandlerAdapter
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        // Add formatters/converters globally
        registry.addConverter(new StringToProductConverter());
        registry.addFormatterForFieldType(
            LocalDate.class,
            new DateTimeFormatterRegistrar()
                .getFormatter(LocalDate.class));
    }
    // addFormatters() is called by WebMvcConfigurationSupport
    // when building the ConversionService for RequestMappingHandlerAdapter
}

// Direct WebBindingInitializer configuration:
@Bean
public RequestMappingHandlerAdapter handlerAdapter() {
    RequestMappingHandlerAdapter adapter =
        new RequestMappingHandlerAdapter();
    adapter.setWebBindingInitializer(
        new ConfigurableWebBindingInitializer() {{
            setConversionService(conversionService());
            setValidator(globalValidator());
            // All binders get this config
        }}
    );
    return adapter;
}
```

---

### Validation — The Full Pipeline

```java
// Validation is triggered by @Valid or @Validated on the parameter

// @Valid: JSR-303 standard annotation (javax.validation / jakarta.validation)
// @Validated: Spring annotation — adds group support
//             @Validated(CreateGroup.class) → only validations in CreateGroup

// STEP 1: Determine validators to apply
// WebDataBinder has a validator list
// Default: SmartValidator wrapping Hibernate Validator (if on classpath)
// Additional: custom validators added via @InitBinder

// STEP 2: Run validation
SmartValidator validator = getValidator(binder);
validator.validate(target, errors);
// → Hibernate Validator processes @NotNull, @Size, @Email, etc.
// → Custom Validator.validate() called if added

// STEP 3: Store errors
// Errors stored in BindingResult (extends Errors)
// FieldErrors: per-field validation failures
// ObjectErrors (global errors): cross-field / object-level failures

// STEP 4: Error code generation
// For @NotNull violation on "name" in Product class:
codes = [
    "NotNull.product.name",    // annotation.objectName.fieldName
    "NotNull.name",            // annotation.fieldName
    "NotNull.java.lang.String", // annotation.fieldType
    "NotNull"                  // annotation
]
// MessageSource resolves these codes for display messages
```

---

### Validation Groups — @Validated Advanced Usage

```java
// Define groups
public interface CreateGroup {}
public interface UpdateGroup {}

// Apply to constraints
public class Product {
    @NotNull(groups=UpdateGroup.class) // Only on update
    private Long id;

    @NotNull(groups={CreateGroup.class, UpdateGroup.class})
    @Size(min=2, max=100,
          groups={CreateGroup.class, UpdateGroup.class})
    private String name;

    @NotNull(groups=CreateGroup.class) // Required on create
    @DecimalMin(value="0.01",
                groups={CreateGroup.class, UpdateGroup.class})
    private BigDecimal price;
}

// Apply groups in controller
@PostMapping("/products")
public String create(
        @Validated(CreateGroup.class)
        @ModelAttribute Product product,
        BindingResult result) { ... }
// Only CreateGroup constraints checked

@PutMapping("/products/{id}")
public String update(
        @PathVariable Long id,
        @Validated(UpdateGroup.class)
        @ModelAttribute Product product,
        BindingResult result) { ... }
// Only UpdateGroup constraints checked
```

---

### Custom Validator — Implementing Validator Interface

```java
// Spring's Validator interface (not JSR-303)
public class ProductValidator implements Validator {

    @Override
    public boolean supports(Class<?> clazz) {
        return Product.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        Product product = (Product) target;

        // ValidationUtils convenience methods
        ValidationUtils.rejectIfEmptyOrWhitespace(
            errors, "name", "field.required",
            "Name is required");

        if (product.getPrice() != null &&
                product.getPrice().compareTo(
                    BigDecimal.ZERO) <= 0) {
            errors.rejectValue(
                "price",           // field name
                "price.invalid",   // error code
                new Object[]{product.getPrice()},  // args for message
                "Price must be positive"   // default message
            );
        }

        // Cross-field validation (global error)
        if (product.getDiscountPrice() != null &&
                product.getPrice() != null &&
                product.getDiscountPrice()
                    .compareTo(product.getPrice()) >= 0) {
            errors.reject(
                "discount.invalid",
                "Discount price must be less than regular price"
            );
        }

        // Nested object validation
        if (product.getAddress() != null) {
            try {
                errors.pushNestedPath("address");
                ValidationUtils.rejectIfEmpty(
                    errors, "city", "field.required");
            } finally {
                errors.popNestedPath();
            }
        }
    }
}

// Register in @InitBinder
@InitBinder
public void initBinder(WebDataBinder binder) {
    binder.addValidators(new ProductValidator());
    // addValidators: ADDS alongside JSR-303
    // setValidator: REPLACES all validators
}
```

---

### BindingResult — Complete API

```java
// BindingResult extends Errors
public interface BindingResult extends Errors {

    // Get the bound object
    Object getTarget();

    // Model representation: {"product": product, "BindingResult.product": this}
    Map<String, Object> getModel();

    // Field-level errors
    List<FieldError> getFieldErrors();
    List<FieldError> getFieldErrors(String field);
    FieldError getFieldError(String field);

    // Object-level errors (global)
    List<ObjectError> getGlobalErrors();
    ObjectError getGlobalError();

    // Summary
    boolean hasErrors();
    int getErrorCount();
    boolean hasFieldErrors();
    boolean hasFieldErrors(String field);
    boolean hasGlobalErrors();

    // Rejecting values programmatically
    void rejectValue(String field, String errorCode);
    void rejectValue(String field, String errorCode,
                     String defaultMessage);
    void rejectValue(String field, String errorCode,
                     Object[] errorArgs, String defaultMessage);
    void reject(String errorCode, String defaultMessage);

    // Raw field value (before type conversion)
    Object getRawFieldValue(String field);

    // Resolved message (via MessageSource)
    String getFieldErrorCode(String field);
}

// FieldError carries:
FieldError fieldError = result.getFieldError("price");
fieldError.getField();           // "price"
fieldError.getRejectedValue();   // "abc" (original String value)
fieldError.getCodes();           // ["typeMismatch.product.price", ...]
fieldError.getDefaultMessage();  // Human-readable default
fieldError.isBindingFailure();   // true if type conversion failed
                                 // false if validation constraint failed
```

---

### @ModelAttribute Methods — Class-Level vs ControllerAdvice

```java
// Controller-level @ModelAttribute — runs for ALL methods in this controller
@Controller
@RequestMapping("/checkout")
public class CheckoutController {

    // Runs BEFORE every handler in CheckoutController
    @ModelAttribute("categories")
    public List<Category> populateCategories() {
        return categoryService.findAll();
        // Performance trap: DB call on every request
    }

    // Named @ModelAttribute — adds to model with specified key
    @ModelAttribute("cart")
    public Cart getOrCreateCart(HttpSession session) {
        Cart cart = (Cart) session.getAttribute("cart");
        if (cart == null) {
            cart = new Cart();
            session.setAttribute("cart", cart);
        }
        return cart;
    }

    // Void @ModelAttribute — adds via Model parameter
    @ModelAttribute
    public void addCommonData(Model model,
                               HttpServletRequest request) {
        model.addAttribute("requestUri",
            request.getRequestURI());
        model.addAttribute("timestamp",
            Instant.now());
    }
}

// @ControllerAdvice @ModelAttribute — runs for ALL controllers in scope
@ControllerAdvice
public class GlobalModelPopulator {

    // Runs BEFORE every controller handler in the application
    @ModelAttribute("currentUser")
    public User getCurrentUser(Principal principal) {
        if (principal == null) return null;
        return userService.findByUsername(
            principal.getName());
    }

    // Scoped to specific controllers
    @ControllerAdvice(
        assignableTypes = {ProductController.class,
                           OrderController.class},
        annotations = RestController.class
    )
    public class ScopedAdvice { ... }
}
```

---

### @SessionAttributes — Session Lifecycle Management

```java
@Controller
@SessionAttributes({
    "cart",          // Cart object stored in session
    "checkoutForm",  // CheckoutForm stored in session
    "userProfile"    // User profile stored in session
})
@RequestMapping("/checkout")
public class CheckoutController {

    // FLOW:
    // 1. Handler adds "cart" to model
    //    → @SessionAttributes stores it in HttpSession

    @GetMapping("/step1")
    public String step1(Model model) {
        // "cart" from @ModelAttribute method → in model
        // @SessionAttributes sees "cart" in model
        // → saves model["cart"] to session["cart"]
        return "checkout/step1";
    }

    // 2. Next request: @ModelAttribute "cart" lookup
    //    → checks model first (not there)
    //    → checks session → FOUND
    //    → restores from session to model

    @PostMapping("/step2")
    public String step2(
            @ModelAttribute("cart") Cart cart,
            // cart retrieved from session (not request params)
            Model model) {
        // Add items from step2 form to existing cart
        cart.setShippingAddress(/* ... */);
        // cart still in model → saved back to session
        return "checkout/step2";
    }

    // 3. Complete → clear session attributes
    @PostMapping("/complete")
    public String complete(
            @ModelAttribute("cart") Cart cart,
            @ModelAttribute("checkoutForm") CheckoutForm form,
            SessionStatus status,
            RedirectAttributes redirectAttrs) {

        orderService.place(cart, form);

        // Clear ALL @SessionAttributes for this controller
        status.setComplete();
        // → HttpSession attributes "cart", "checkoutForm",
        //   "userProfile" removed from session

        redirectAttrs.addFlashAttribute("success", "Order placed!");
        return "redirect:/orders/confirmation";
    }
}
```

---

### HTML Checkbox Binding — Special Case

```java
// HTML checkboxes have a quirk:
// If checkbox is UNCHECKED, no parameter is sent at all
// Spring MVC handles this via hidden fields

// In JSP/Thymeleaf:
// <input type="checkbox" name="active" value="true"/>
// When checked: request has "active=true"
// When unchecked: request has NO "active" parameter

// Spring's solution: hidden field with "_" prefix
// <input type="hidden" name="_active" value="on"/>
// Spring DataBinder:
//   If "active" present → use its value
//   If "active" absent BUT "_active" present → set field to false/null

// With Spring tag library (handles automatically):
// <spring:checkbox path="active"/>
// Generates both checkbox AND hidden field

// Spring MVC handling in DataBinder:
protected void checkFieldMarkers(MutablePropertyValues mpvs) {
    PropertyValue[] pvArray = mpvs.getPropertyValues();
    for (PropertyValue pv : pvArray) {
        // Find "_active" (field marker prefix "_")
        if (pv.getName().startsWith(FIELD_MARKER_PREFIX)) {
            String field = pv.getName().substring(
                FIELD_MARKER_PREFIX.length());
            if (!mpvs.contains(field)) {
                // "active" not in request → checkbox unchecked
                // Set to empty/false
                mpvs.add(field, getEmptyValue(field, ...));
            }
        }
    }
}
```

---

### Binding Security — Mass Assignment Prevention

```java
// VULNERABILITY: If Product has 'admin' or 'role' fields,
// a malicious request could set them via form submission

// POST /products
// Body: name=Widget&price=9.99&role=ADMIN&admin=true

@Controller
public class ProductController {

    // FIX 1: setAllowedFields — whitelist
    @InitBinder("product")
    public void protectProduct(WebDataBinder binder) {
        binder.setAllowedFields(
            "name", "price", "description", "categoryId");
        // ONLY these fields will be bound
        // "role", "admin", "id" silently ignored even if in request
    }

    // FIX 2: setDisallowedFields — blacklist
    @InitBinder("product")
    public void blacklistProduct(WebDataBinder binder) {
        binder.setDisallowedFields("id", "role", "admin",
            "createdAt", "updatedAt", "version");
        // These NEVER bound even if present in request
        // Other fields ARE bound
    }

    // FIX 3: Use DTOs (best practice)
    // ProductCreateRequest — only has fields you want bound
    // No security fields at all
    @PostMapping("/products")
    public String create(
            @Valid @ModelAttribute ProductCreateRequest dto,
            BindingResult result) {
        Product product = mapper.toProduct(dto);
        productService.save(product);
        return "redirect:/products";
    }
}
```

---

## 2️⃣ CODE EXAMPLES

### Complete Form Controller with All Features

```java
@Controller
@RequestMapping("/products")
@SessionAttributes("productForm")
public class ProductFormController {

    // Pre-populate model for ALL methods
    @ModelAttribute("categories")
    public List<Category> getCategories() {
        return categoryService.findAll();
    }

    // Initialize empty form for session storage
    @ModelAttribute("productForm")
    public ProductForm initForm() {
        return new ProductForm();
    }

    // Global binder configuration
    @InitBinder
    public void initGlobalBinder(WebDataBinder binder) {
        // Register custom date formatter
        binder.addCustomFormatter(
            new DateTimeFormatterRegistrar()
                .getFormatter(LocalDate.class));
    }

    // Product-specific binder
    @InitBinder("productForm")
    public void initProductBinder(WebDataBinder binder) {
        // Security: whitelist bindable fields
        binder.setAllowedFields(
            "name", "price", "description",
            "categoryId", "releaseDate", "tags");

        // Add domain validator
        binder.addValidators(new ProductFormValidator());
    }

    // Show empty form
    @GetMapping("/new")
    public String showForm() {
        // "productForm" from @ModelAttribute method
        // "categories" from @ModelAttribute method
        return "product/form";
    }

    // Process form submission
    @PostMapping
    public String processForm(
            @Validated({CreateGroup.class})
            @ModelAttribute("productForm") ProductForm form,
            BindingResult result,  // MUST be adjacent!
            SessionStatus sessionStatus,
            RedirectAttributes redirectAttrs) {

        // Check for binding + validation errors
        if (result.hasErrors()) {
            // Log field errors for debugging
            result.getFieldErrors().forEach(e ->
                log.debug("Error on field '{}': {} (rejected: {})",
                    e.getField(),
                    e.getDefaultMessage(),
                    e.getRejectedValue())
            );
            // model has "productForm" with user input preserved
            // model has "categories" from @ModelAttribute method
            // model has "BindingResult.productForm"
            return "product/form"; // Show form again
        }

        // Save
        Product saved = productService.create(form);

        // Clear session
        sessionStatus.setComplete();

        // Flash for redirect
        redirectAttrs.addFlashAttribute(
            "successMessage",
            "Product '" + saved.getName() + "' created!");

        return "redirect:/products/" + saved.getId();
    }
}

// ProductForm DTO with constraints
public class ProductForm {

    @NotBlank(groups={CreateGroup.class, UpdateGroup.class})
    @Size(min=2, max=100,
          groups={CreateGroup.class, UpdateGroup.class})
    private String name;

    @NotNull(groups={CreateGroup.class, UpdateGroup.class})
    @DecimalMin(value="0.01",
                groups={CreateGroup.class, UpdateGroup.class})
    private BigDecimal price;

    @NotNull(groups=CreateGroup.class)
    private Long categoryId;

    @DateTimeFormat(iso=DateTimeFormat.ISO.DATE)
    private LocalDate releaseDate;

    @Size(max=10, message="Maximum 10 tags")
    private List<String> tags = new ArrayList<>();

    // getters/setters
}
```

---

### Custom Validator With Cross-Field Logic

```java
// Validator for password confirmation
public class PasswordChangeValidator implements Validator {

    @Override
    public boolean supports(Class<?> clazz) {
        return PasswordChangeRequest.class
            .isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        PasswordChangeRequest req =
            (PasswordChangeRequest) target;

        // Individual field validations
        ValidationUtils.rejectIfEmpty(errors,
            "currentPassword", "required",
            "Current password required");

        ValidationUtils.rejectIfEmpty(errors,
            "newPassword", "required",
            "New password required");

        if (errors.hasErrors()) {
            return; // Skip cross-field if individual errors
        }

        // Password strength check
        if (req.getNewPassword().length() < 8) {
            errors.rejectValue("newPassword",
                "password.too.short",
                new Object[]{8},
                "Password must be at least {0} characters");
        }

        // Cross-field: confirmation must match
        if (!req.getNewPassword()
                .equals(req.getConfirmPassword())) {
            // Global error (not field-specific)
            errors.reject(
                "password.mismatch",
                "Passwords do not match");
            // Also add field error on confirmPassword
            errors.rejectValue("confirmPassword",
                "password.mismatch",
                "Must match new password");
        }

        // Cannot reuse current password
        if (req.getNewPassword()
                .equals(req.getCurrentPassword())) {
            errors.rejectValue("newPassword",
                "password.same.as.current",
                "New password must differ from current");
        }
    }
}
```

---

### Custom Type Converter — String to Money

```java
// Register custom converter for Money type
@Component
public class StringToMoneyConverter
    implements Converter<String, Money> {

    @Override
    public Money convert(String source) {
        // "19.99 USD" → Money(19.99, Currency.USD)
        String[] parts = source.trim().split("\\s+");
        if (parts.length != 2) {
            throw new ConversionFailedException(
                TypeDescriptor.valueOf(String.class),
                TypeDescriptor.valueOf(Money.class),
                source,
                new IllegalArgumentException(
                    "Expected format: 'amount currency'"));
        }
        return new Money(
            new BigDecimal(parts[0]),
            Currency.getInstance(parts[1]));
    }
}

// Register globally
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToMoneyConverter());
    }
}

// Usage in controller — type conversion happens automatically
@PostMapping("/orders")
public String createOrder(
        @RequestParam Money totalAmount) {
    // GET /orders?totalAmount=99.99+USD
    // String "99.99 USD" → Money(99.99, USD) via converter
}

@ModelAttribute
public void bindMoney(
        @ModelAttribute("order") Order order) {
    // Form field "payment.amount" = "50.00 EUR"
    // → order.getPayment().setAmount(Money(50.00, EUR))
}
```

---

### Binding Nested Objects and Collections

```java
// Domain model with nesting
public class Order {
    private Long id;
    private Customer customer;          // Nested object
    private List<OrderLine> lines;      // Collection
    private Map<String, String> metadata; // Map

    // Getters/setters
}

// Controller
@PostMapping("/orders")
public String createOrder(
        @ModelAttribute Order order,
        BindingResult result) {
    // Request params that bind to nested structure:
    // customer.name=John
    //   → order.getCustomer().setName("John")
    //   → If customer is null → auto-created (autoGrowNestedPaths)
    //
    // customer.address.city=London
    //   → order.getCustomer().getAddress().setCity("London")
    //   → Auto-creates customer AND address if null
    //
    // lines[0].productId=42
    //   → order.getLines().get(0).setProductId(42)
    //   → Auto-grows list to index 0
    //
    // lines[0].quantity=2
    //   → order.getLines().get(0).setQuantity(2)
    //
    // lines[1].productId=99
    //   → Auto-grows list to index 1
    //
    // metadata[reference]=ORD-2024-001
    //   → order.getMetadata().put("reference", "ORD-2024-001")
    //   → Auto-creates map if null
    //
    // SECURITY RISK: lines[999999].quantity=1
    //   → Auto-grows list to 1,000,000 elements → OOM
    //   → Protected by autoGrowCollectionLimit=256

    if (result.hasErrors()) return "order/form";
    orderService.save(order);
    return "redirect:/orders";
}

// HTML form fields that generate these params:
// <input name="customer.name" value="John"/>
// <input name="customer.address.city" value="London"/>
// <input name="lines[0].productId" value="42"/>
// <input name="lines[0].quantity" value="2"/>
// <input name="metadata[reference]" value="ORD-2024-001"/>
```

---

## 3️⃣ EXAM-STYLE QUESTIONS

---

**Q1 — MCQ**
`WebDataBinder.autoGrowCollectionLimit` has a default value. What is its purpose and default?

A) Limits string length during conversion — default 1024
B) Limits auto-growth of collections during binding — default 256
C) Limits the number of request parameters processed — default 100
D) Limits recursion depth for nested objects — default 10

**Answer: B**
`autoGrowCollectionLimit` (default: 256) prevents malicious requests from causing `OutOfMemoryError` by sending `list[999999]=value`. Auto-growing a collection to index 999999 would allocate a million-element list. The 256 limit means any index above 256 causes a binding error rather than list growth.

---

**Q2 — Select All That Apply**
Which of the following correctly describe `@InitBinder` method rules?

A) Can return any type — return value is ignored
B) `@InitBinder("product")` only applies when binding a "product" named attribute
C) `@ControllerAdvice` `@InitBinder` methods execute BEFORE controller-local ones
D) Can receive `HttpServletRequest` as a parameter
E) Can be used to call `binder.setValidator()` which REPLACES all validators

**Answer: B, C, D, E**
A is wrong — `@InitBinder` methods must return `void`. B is correct — named binding is scoped. C is correct — global (ControllerAdvice) before local (controller). D is correct — `HttpServletRequest` is a valid `@InitBinder` parameter. E is correct — `setValidator()` replaces; use `addValidators()` to add alongside existing.

---

**Q3 — Ordering**
For `@Valid @ModelAttribute("product") Product product, BindingResult result`, order these operations:

- `WebDataBinder.bind()` maps request params to Product fields
- `BindingResult` stored in model
- Product retrieved from session or created new
- JSR-303 validation runs
- `WebDataBinder` created + `@InitBinder` methods applied
- Product stored in model

**Correct Order:**
1. Product retrieved from session (via `@SessionAttributes`) or created new
2. `WebDataBinder` created + `@InitBinder` methods applied
3. `WebDataBinder.bind()` maps request params to Product fields
4. JSR-303 validation runs (triggered by `@Valid`)
5. Product stored in model as "product"
6. `BindingResult` stored in model as `BindingResult.product`

---

**Q4 — MCQ**
`binder.setAllowedFields("name", "price")` is called in `@InitBinder`. A request arrives with params: `name=Widget&price=9.99&id=1&role=admin`. What fields are bound?

A) All fields including `id` and `role`
B) Only `name` and `price` — `id` and `role` silently ignored
C) `name` and `price` bound; `id` and `role` cause `FieldError`
D) `IllegalArgumentException` thrown for disallowed fields

**Answer: B**
`setAllowedFields` is a whitelist. `checkAllowedFields()` in `DataBinder.doBind()` removes any `PropertyValue` whose name is not in the allowed list. `id` and `role` are simply not passed to `BeanWrapper.setPropertyValues()`. No error, no exception — silently ignored. This is the intended security behaviour.

---

**Q5 — True/False**
`@Valid` and `@Validated` are fully interchangeable — both support validation groups.

**Answer: False**
`@Valid` (JSR-303/Bean Validation standard) does NOT support groups at the annotation level. You cannot write `@Valid(groups=CreateGroup.class)`.
`@Validated` (Spring) DOES support groups: `@Validated(CreateGroup.class)`.
Both trigger validation. `@Validated` is the correct choice when you need to activate specific validation groups.

---

**Q6 — Code Prediction**
```java
@Controller
public class FormCtrl {

    @InitBinder("data")
    public void initDataBinder(WebDataBinder binder) {
        binder.setAllowedFields("name");
    }

    @PostMapping("/submit")
    public String submit(
            @ModelAttribute("data") MyForm data,
            BindingResult result) {
        System.out.println(data.getName() + " " + data.getAge());
        return "result";
    }
}
```
`POST /submit?name=Alice&age=30` — what is printed?

A) `Alice 30`
B) `Alice 0` (or `Alice null`)
C) `IllegalArgumentException` — age not allowed
D) `BindException` — age field rejected

**Answer: B**
`@InitBinder("data")` applies only to the "data" model attribute. `setAllowedFields("name")` → only `name` is bound. `age` is silently ignored. `data.getAge()` returns the default value: `0` for `int`, `null` for `Integer`. No exception, no error. Output: `Alice 0` (if `age` is `int`) or `Alice null` (if `Integer`).

---

**Q7 — MCQ**
`BindingResult.getFieldError("price").isBindingFailure()` returns `true`. What does this mean?

A) JSR-303 `@NotNull` constraint failed on `price`
B) The `price` field value failed type conversion (String → Double)
C) The `price` field was rejected by `setDisallowedFields()`
D) A custom validator rejected the `price` field value

**Answer: B**
`isBindingFailure()` returns `true` when the error originated from a **type conversion failure** during the binding phase — not from a validation constraint. For example, `price=abc` cannot be converted to `Double`. Validation constraint failures (B, C, D) have `isBindingFailure()=false`. This distinction matters for UI: binding failures should show "Invalid format" while validation failures show domain-specific messages.

---

**Q8 — Scenario**
A `@Controller` has `@SessionAttributes("cart")`. Request 1 (GET) adds a `Cart` to the model. Request 2 (POST) modifies the cart. Request 3 (POST /complete) calls `sessionStatus.setComplete()`. What happens to the `Cart` in the session after Request 3?

A) `Cart` remains in session — `setComplete()` only marks completion, not removal
B) `Cart` is removed from `HttpSession` — `SessionAttributesHandler.cleanupAttributes()` is called
C) `Cart` is removed from model but remains in session
D) `Cart` is set to `null` in session

**Answer: B**
`SessionStatus.setComplete()` marks the controller's session as complete. At the end of request processing, `ModelFactory.updateModel()` calls `SessionAttributesHandler.cleanupAttributes(webRequest)` which calls `webRequest.removeAttribute("cart", WebRequest.SCOPE_SESSION)` for each attribute declared in `@SessionAttributes`. The `Cart` is completely removed from `HttpSession`.

---

## 4️⃣ TRICK ANALYSIS

**Trap 1 — `@InitBinder` method must return `void`**
`@InitBinder` annotated methods MUST return `void`. If they return any value, Spring throws `IllegalStateException: @InitBinder methods should return void`. No other return type is supported. This is checked at startup when `RequestMappingHandlerAdapter` builds the `WebDataBinderFactory`.

**Trap 2 — `setValidator()` vs `addValidators()` — different effects**
`binder.setValidator(v)` REPLACES all validators including the default JSR-303 validator. After this, `@NotNull`, `@Size`, etc. do NOT run. Only your custom validator runs.
`binder.addValidators(v)` ADDS to the existing list. JSR-303 + your validator both run.
Almost always you want `addValidators()`. Using `setValidator()` accidentally removes Hibernate Validator.

**Trap 3 — `BindingResult` with `@RequestBody` does NOT suppress exceptions**
For `@ModelAttribute` + `BindingResult`: validation errors go into `BindingResult`, no exception thrown, method executes.
For `@RequestBody` + `@Valid` + `BindingResult`: `MethodArgumentNotValidException` is thrown BEFORE method executes regardless of `BindingResult` presence.
The `BindingResult` pattern only works for `@ModelAttribute` form binding. For REST validation, use `@ExceptionHandler`.

**Trap 4 — `autoGrowNestedPaths=true` creates objects automatically**
If `product.address` is null and request param `product.address.city=London` arrives, Spring auto-creates `Address` via default constructor and sets it on `product`. If `Address` has no default constructor → `BeanInstantiationException`. This is surprising behavior. Set `autoGrowNestedPaths=false` if you want to prevent automatic object creation.

**Trap 5 — HTML checkbox binding requires hidden field**
Unchecked HTML checkbox → no parameter sent → field NOT bound → old value persists. The hidden `_fieldname` field tells Spring the checkbox was submitted (just unchecked). Without the hidden field or Spring tag library: update form where user unchecks a checkbox → old `true` value remains in the model. `@ModelAttribute` binding doesn't see the absence of a parameter as "set to false".

**Trap 6 — `@SessionAttributes` scope is per-controller class**
`@SessionAttributes("cart")` stores `cart` in session with the controller's class name as a namespace key (internally). Two different controllers both declaring `@SessionAttributes("cart")` store SEPARATE session entries. They don't share session data. `sessionStatus.setComplete()` on one controller clears only that controller's session attributes.

**Trap 7 — `@ModelAttribute` method order is NOT guaranteed (within same tier)**
`@ControllerAdvice` `@ModelAttribute` methods run before controller-local ones. But within the same tier (multiple `@ControllerAdvice` beans), order is determined by `@Order`. Without `@Order`, the order is UNDEFINED. If `@ModelAttribute` method B depends on model attributes set by method A (same controller), order is undefined unless explicit dependency declared.

---

## 5️⃣ SUMMARY SHEET

```
DATA BINDING PIPELINE
─────────────────────────────────────────────────────
HTTP Request Params (String)
  → WebDataBinder.bind()
  → checkAllowedFields() / checkRequiredFields()
  → BeanWrapper.setPropertyValues()
  → ConversionService / PropertyEditors
  → FieldErrors on type conversion failure
  → validateIfApplicable() [@Valid/@Validated]
  → BindingResult populated
  → Object + BindingResult added to model

@MODELATTRIBUTE RESOLUTION PRIORITY
─────────────────────────────────────────────────────
1. Already in model (from @ModelAttribute method) → use + bind on top
2. In @SessionAttributes session → retrieve + bind on top
3. Not found → create new instance → bind from scratch

WEBDATABINDER KEY SETTINGS
─────────────────────────────────────────────────────
setAllowedFields(...)       → whitelist (only these bound)
setDisallowedFields(...)    → blacklist (never bound, silent)
setRequiredFields(...)      → must be present, else FieldError
setValidator(v)             → REPLACES all validators (dangerous!)
addValidators(v)            → ADDS to existing validators (correct)
autoGrowNestedPaths=true    → auto-creates nested null objects
autoGrowCollectionLimit=256 → prevents OOM from large indexes

@INITBINDER RULES
─────────────────────────────────────────────────────
Must return void — anything else → IllegalStateException
@InitBinder("name") → only for that named model attribute
@InitBinder (no name) → applies to ALL bindings in controller
Execution order: @ControllerAdvice first, controller-local second
Parameters: WebDataBinder (required), WebRequest/HttpServletRequest, Locale

VALIDATION TRIGGERS
─────────────────────────────────────────────────────
@Valid       → JSR-303 standard, NO group support
@Validated   → Spring annotation, group support: @Validated(CreateGroup.class)
Both trigger SmartValidator (Hibernate Validator) + custom validators

VALIDATION OUTCOME
─────────────────────────────────────────────────────
@ModelAttribute + BindingResult:
  → Errors stored in BindingResult, method EXECUTES
  → result.hasErrors() → return to form view

@ModelAttribute WITHOUT BindingResult:
  → BindException thrown if errors
  → Method does NOT execute

@RequestBody + @Valid + BindingResult:
  → MethodArgumentNotValidException thrown regardless
  → BindingResult presence does NOT suppress exception

BINDINGRESUILT.isBindingFailure()
─────────────────────────────────────────────────────
true  → type conversion failure (String → type failed)
false → validation constraint failure (@NotNull, @Size, etc.)

@SESSIONATTRIBUTES LIFECYCLE
─────────────────────────────────────────────────────
Model attribute added → stored in session (if name matches @SessionAttributes)
Next request → retrieved from session → placed in model
Handler adds to model → saved back to session
sessionStatus.setComplete() → ALL declared attrs removed from session

CHECKBOX BINDING
─────────────────────────────────────────────────────
Unchecked checkbox → no param sent → Spring doesn't know it was submitted
Solution: hidden field "_fieldname" → Spring sees checkbox was in form
Spring tag <form:checkbox> handles this automatically

MASS ASSIGNMENT PROTECTION
─────────────────────────────────────────────────────
Approach 1: setAllowedFields() → whitelist (preferred for security)
Approach 2: setDisallowedFields() → blacklist (miss one field = vulnerability)
Approach 3: DTO pattern → only contains bindable fields (best practice)

AUTOGROW LIMITS
─────────────────────────────────────────────────────
autoGrowCollectionLimit = 256 (default)
  → list[999999] → binding error (not OOM)
autoGrowNestedPaths = true (default)
  → null nested objects auto-created
  → requires default constructor

INTERVIEW ONE-LINERS
─────────────────────────────────────────────────────
• "@InitBinder must return void — any other return type = IllegalStateException"
• "setValidator() replaces all validators; addValidators() adds alongside"
• "@Valid has no group support; @Validated(Group.class) does"
• "BindingResult with @RequestBody: exception still thrown — not suppressed"
• "autoGrowCollectionLimit=256 prevents OOM from malicious list[999999]"
• "isBindingFailure()=true: type conversion failed; false: constraint failed"
• "@SessionAttributes: sessionStatus.setComplete() removes all declared attrs from session"
• "HTML checkbox: unchecked = no param sent; use _fieldname hidden field"
• "setAllowedFields() whitelist: fields not listed silently ignored (no error)"
• "@ControllerAdvice @InitBinder runs before controller-local @InitBinder"
```

---
