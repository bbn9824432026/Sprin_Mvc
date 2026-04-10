# Complete Deep Dive: Method Return Values and Their Handling in Spring MVC

Let me guide you through the mirror image of argument resolution - how Spring MVC processes what your controller methods return. While argument resolution is about getting data into your method, return value handling is about taking what your method produces and turning it into an HTTP response. This system is equally sophisticated and understanding it completely is essential for mastering Spring MVC.

## The Fundamental Challenge of Return Value Processing

Think about the variety of things a controller method might return. You could return a String that should be interpreted as a view name. Or a String that should be written directly to the response as text. Or a domain object that should be serialized to JSON. Or a ResponseEntity that specifies status code, headers, and body. Or a Callable that runs asynchronously. Or a stream of events. Or nothing at all.

Spring needs to look at what you returned, understand what it represents based on its type and the context (annotations, controller type, etc.), and then do the right thing to produce an HTTP response. This is return value handling, and it uses the same chain-of-responsibility pattern we saw with argument resolution.

## The Return Value Handler Chain and Its Operation

Just like argument resolvers, Spring maintains an ordered list of `HandlerMethodReturnValueHandler` implementations. Each handler knows how to process certain types of return values. When your method finishes executing and returns something, Spring walks through this chain asking each handler: "can you handle this return type?" The first one that says yes wins and becomes responsible for processing that return value.

The interface is simple. Each handler implements `supportsReturnType()` to indicate whether it can handle a particular return type, and `handleReturnValue()` to actually process the value. The chain walk happens once per method signature and the result is cached, just like with argument resolvers. Subsequent requests for the same method use the cached handler directly.

But there's a critical difference from argument resolution. Return value handlers don't just produce a value - they interact with a `ModelAndViewContainer` object that determines what happens next in the request processing flow. This container holds the model, the view (or view name), and most importantly, a boolean flag called `requestHandled`.

Understanding this flag is the key to understanding return value handling. When a handler sets `requestHandled=true`, it's telling Spring: "I've completely handled this request, the response has been written, don't do anything else." When it leaves `requestHandled=false`, it's saying: "I've set up the model and view, now you need to render it."

This single boolean is the entire difference between REST endpoints that return JSON and traditional MVC endpoints that render views. REST handlers set `requestHandled=true` after writing the response. View-based handlers leave it false and Spring's view resolution machinery takes over.

## The Handler Chain Order and Why It Matters

The order of handlers in the chain is crucial because it determines precedence when multiple handlers could theoretically handle the same return type. Spring registers its built-in handlers in a carefully chosen order.

Specific type-based handlers come first. `ModelAndViewMethodReturnValueHandler` handles `ModelAndView` objects. `ViewMethodReturnValueHandler` handles `View` interface implementations. These are registered early because they're unambiguous - if you return these types, there's no question what you intend.

Then come the async handlers - `CallableMethodReturnValueHandler`, `DeferredResultMethodReturnValueHandler`, `ReactiveTypeReturnValueHandler`. These handle various forms of asynchronous processing. They're early in the chain because async return types need special handling that can't be deferred to later handlers.

The critical handler for REST endpoints is `RequestResponseBodyMethodProcessor`. It checks whether your method or containing class has `@ResponseBody`. This is where the `@RestController` magic happens - remember that `@RestController` is `@Controller` plus class-level `@ResponseBody`, so every method in a `@RestController` has `@ResponseBody` through meta-annotation resolution. This handler comes before the view-related handlers, which is why `@ResponseBody` takes precedence.

Then comes `HttpEntityMethodProcessor` which handles `ResponseEntity` and `HttpEntity` return types. This is your most powerful return type for REST endpoints because it gives you complete control over status code, headers, and body.

After these come the view-related handlers. `ViewNameMethodReturnValueHandler` handles String return values, interpreting them as view names. `MapMethodProcessor` handles Map returns by adding them to the model.

At the very end are handlers for void returns and null values, which have context-dependent behavior.

## Understanding String Returns - The Most Confusing Case

String return values are probably the most confusing aspect of return value handling because their meaning changes completely based on context. Let me explain this in detail because it's a constant source of bugs.

In a traditional `@Controller` class, when you return a String, the `ViewNameMethodReturnValueHandler` claims it. This handler interprets the String as a logical view name. It calls `mavContainer.setViewName(returnValue)` and leaves `requestHandled=false`. Spring's view resolution machinery then takes over. The view name gets passed to your configured ViewResolver, which might turn "product/list" into "/WEB-INF/views/product/list.jsp" or a Thymeleaf template or whatever view technology you're using.

Special view names like "redirect:/products" or "forward:/details" get special handling. The "redirect:" prefix creates a `RedirectView` which sends an HTTP 302 redirect. The "forward:" prefix creates a forward to another handler without a redirect - the same request continues to a different URL.

But in a `@RestController` class, everything changes. Remember that `@RestController` has class-level `@ResponseBody`. The `RequestResponseBodyMethodProcessor` checks for `@ResponseBody` on the method or class, and it does this check before `ViewNameMethodReturnValueHandler` gets a chance. So the processor claims the String return value.

What does it do with a String? It treats it as response body content. It uses `StringHttpMessageConverter` to write the String directly to the response as text. If you return "product/list" from a `@RestController` method, the literal text "product/list" gets written to the response body. It's not interpreted as a view name. There's no view resolution. There's no redirect.

This is the number one mistake developers make when starting with `@RestController`. They write `return "redirect:/success"` expecting a redirect, and instead they get the literal text "redirect:/success" as the response body. To do a redirect from a `@RestController`, you need to use `ResponseEntity` with a proper redirect status code and Location header.

## ModelAndView - Explicit Control Over Everything

The `ModelAndView` return type gives you complete control over both the model and the view in one object. When you return `ModelAndView`, the `ModelAndViewMethodReturnValueHandler` processes it.

You can set the view in three ways. You can set a view name as a String, which gets resolved by the ViewResolver just like a String return would. You can set an actual View object if you want to bypass view resolution. Or you can set a view name with a redirect or forward prefix for special navigation.

You add model attributes by calling `addObject()` with key-value pairs. These become available to your view template. The handler extracts everything from the `ModelAndView` and puts it into the `ModelAndViewContainer`. The model attributes go into the model. The view or view name goes into the container. And `requestHandled` stays false so view rendering proceeds.

You can also set an HTTP status code on the `ModelAndView` (since Spring 5). This is useful for returning error pages with proper status codes like 404 or 500 while still rendering a view.

The `ModelAndView` approach is verbose compared to just returning a String and using a Model parameter, but it's useful when you need to construct everything programmatically or when the view name is determined dynamically based on complex logic.

## ResponseEntity - Complete Control for REST Endpoints

`ResponseEntity` is the most powerful return type for REST APIs. It gives you complete control over the HTTP status code, all headers, and the response body, all in one object. The `HttpEntityMethodProcessor` handles it.

When you return `ResponseEntity<Product>`, the handler extracts three things from it: the HTTP status, the HTTP headers, and the body. It processes them in a specific order that's important to understand.

First, it sets the status code on the `HttpServletResponse`. This must happen before writing the body because once you start writing to the response, the status is locked in.

Second, it sets all the headers from the `ResponseEntity`'s HttpHeaders object onto the response. Again, headers must be set before the body is written because the response might be committed as soon as you write to it.

Third, it writes the body using the same message converter mechanism that `@ResponseBody` uses. It performs content negotiation based on the Accept header, finds an appropriate `HttpMessageConverter`, and uses it to serialize the body.

Finally, it sets `requestHandled=true` because the response is complete. No view resolution needed.

The builder pattern for `ResponseEntity` is elegant and expressive. `ResponseEntity.ok(body)` gives you 200 with a body. `ResponseEntity.created(location).body(body)` gives you 201 with a Location header and body. `ResponseEntity.notFound().build()` gives you 404 with no body. `ResponseEntity.noContent().build()` gives you 204 with no body.

You can set any header you want. ETag for caching. Last-Modified for conditional requests. Cache-Control for cache directives. Content-Disposition for downloads. Custom headers for whatever your API needs.

The key difference between `ResponseEntity` and plain `@ResponseBody` is control. With `@ResponseBody`, you can only control the body. The status is always 200 unless you use `@ResponseStatus` annotation, which is static - you can't change it based on runtime conditions. You can't set custom headers at all (except through ResponseBodyAdvice which we'll discuss later). `ResponseEntity` gives you full programmatic control over the entire response.

## The Critical RequestHandled Flag

Let me emphasize this because it's central to understanding the entire system. The `ModelAndViewContainer` has a boolean field called `requestHandled`. This field determines whether Spring will attempt view resolution and rendering after your handler method completes.

When a return value handler sets `requestHandled=true`, it's declaring that the HTTP response has been fully constructed. The body has been written, headers have been set, status has been set. Spring's job is done. The `DispatcherServlet` checks this flag after return value handling and if it's true, it skips all the view resolution machinery and just completes the request.

When `requestHandled=false`, the handler is saying it has prepared a model and identified a view, but the response hasn't been written yet. Spring needs to resolve the view name to an actual View object, then render that view with the model to produce HTML (or whatever the view produces), then write that rendered content to the response.

This is why REST and traditional MVC can coexist in the same application. REST handlers (`@ResponseBody`, `ResponseEntity`) set the flag to true. View-based handlers (String view names, `ModelAndView`) leave it false. They produce different types of responses through this one mechanism.

Understanding this flag also helps you understand some surprising behaviors. If you inject `HttpServletResponse` into a handler method and write to its output stream, then return a view name, the view rendering might fail because the response was already committed. The `ServletResponseMethodArgumentResolver` that handles `HttpServletResponse` parameters sets `requestHandled=true` because it assumes you're writing the response directly.

## Void Returns and Their Context-Dependent Behavior

The void return type has three completely different behaviors depending on context, and understanding all three is important.

In a `@RestController`, when you return void, the `RequestResponseBodyMethodProcessor` handles it. The method has `@ResponseBody` through the class annotation, so the processor claims void returns. It sets `requestHandled=true` with no body. This is useful when you're writing directly to the `HttpServletResponse` - you get the response output stream, write your bytes (maybe a file download), and return void to say "I handled it, don't do anything else."

In a `@Controller` where you write to the response and the response gets committed, void return also results in `requestHandled=true`. Spring detects that the response is committed and knows it can't render a view over an already-written response.

But in a `@Controller` where the response is not committed, void return has different behavior. The `ViewNameMethodReturnValueHandler` accepts void returns. It leaves `requestHandled=false`. Spring's `DefaultRequestToViewNameTranslator` then derives a view name from the request URL. If the URL is "/products/list", the view name becomes "products/list". The ViewResolver tries to find a view with that name.

This third case surprises many developers. They write a void method that sets model attributes, intending just to set up the model without specifying a view. But Spring derives a view name and tries to render it. If no view exists with that derived name, you get an error during view resolution.

The lesson: only use void returns in `@Controller` when you're explicitly writing to the response and want no view resolution. If you want a specific view, return its name. If you want no view, use `@ResponseBody` or write to the response to commit it.

## Asynchronous Return Types and Their Processing

Asynchronous return types fundamentally change the request processing model. Instead of your handler method doing all the work on the servlet container thread and producing a complete response before returning, async handlers return immediately with a promise of a future result. The servlet thread is released back to the pool. When the result is ready, Spring dispatches a new request to process it.

The `Callable<T>` return type is the simplest async pattern. You return a `Callable` which is just a task that produces a value. Spring's `CallableMethodReturnValueHandler` takes this callable and submits it to a `TaskExecutor` - typically a thread pool. Your callable runs on a thread pool thread, not a servlet thread. When it completes and returns a value, Spring triggers a second dispatch of the request. This second dispatch processes the result as if it were the original return value.

This pattern is great for offloading CPU-intensive work from servlet threads. The servlet thread returns immediately after handing off the work. The thread pool thread does the heavy lifting. The servlet thread pool stays available for other requests.

But there's an important implication for interceptors and filters. The `postHandle()` method of interceptors does not run after the first return - it runs after the second dispatch when the async result is processed. If you have cleanup logic in `postHandle()`, it runs on a different thread, at a different time, than the original handler execution.

`DeferredResult<T>` is different from `Callable` in a crucial way. With `Callable`, Spring manages the async execution in a thread pool. With `DeferredResult`, you manage it. You create a `DeferredResult` and return it immediately. Spring releases the servlet thread. At some point in the future - could be in another thread, could be in a callback from an external service, could be anywhere - you call `setResult()` on the `DeferredResult`. This triggers the second dispatch.

The power of `DeferredResult` is that no thread is held waiting for the result. With `Callable`, a thread pool thread is occupied executing your callable. With `DeferredResult`, there's no thread at all - you're waiting for an external event. This is perfect for long-polling, waiting for messages from a queue, waiting for external API calls, or any scenario where you're waiting for something that will notify you when ready.

You can set a timeout on `DeferredResult`. If the result isn't set within the timeout period, Spring can throw an exception or use a default value you provide. This prevents requests from hanging forever if the result never arrives.

`CompletableFuture<T>` is handled by adapting it to a `DeferredResult`. The `DeferredResultMethodReturnValueHandler` recognizes `CompletableFuture` and bridges it to the `DeferredResult` mechanism. When the future completes, the result is set on the bridged `DeferredResult`, triggering the second dispatch.

`WebAsyncTask<T>` wraps a `Callable` with additional configuration. You can set timeouts, timeout handlers, error handlers, and completion callbacks. This gives you more control than plain `Callable` for async processing.

## Reactive Types and Their Integration

Spring MVC has special support for Project Reactor types even though Spring MVC itself is not reactive. The `ReactiveTypeReturnValueHandler` bridges the reactive world to the servlet world.

When you return `Mono<T>`, the handler subscribes to it and adapts it to a `DeferredResult`. The `Mono` completes (or errors) at some point in the future, which sets the result on the `DeferredResult`, which triggers the second dispatch. The returned value gets processed as if you had returned it directly.

This lets you use reactive clients like WebClient in Spring MVC controllers. You make a non-blocking HTTP call that returns a `Mono<ResponseEntity>`, and Spring MVC handles it transparently. But remember that while the call is non-blocking, it still ties up servlet infrastructure. True reactive benefits require a reactive runtime like Spring WebFlux on Netty.

`Flux<T>` has special handling when the client accepts Server-Sent Events (media type `text/event-stream`). Instead of collecting all elements and returning them as an array, the handler switches to `SseEmitter` mode. Each element emitted by the `Flux` becomes an SSE event sent to the client. The connection stays open and events stream as they're produced.

This is powerful for real-time updates. Your handler returns a `Flux` that emits events as they occur, and the client receives them as a stream of SSE events. No polling needed.

## Streaming Response Body

For large files or streaming data, holding the entire content in memory and writing it all at once is inefficient. `StreamingResponseBody` solves this.

When you return `StreamingResponseBody` (usually wrapped in `ResponseEntity` to set headers), the `StreamingResponseBodyReturnValueHandler` starts async processing. It gives you an `OutputStream` on an async thread, and you write to it directly. You can stream gigabytes of data without holding it all in memory.

This is perfect for file downloads, CSV exports, log streaming, or any scenario where you're producing a large response incrementally. The key is that your streaming callback runs on a separate thread - the servlet thread is released immediately.

`SseEmitter` is similar but specifically for Server-Sent Events protocol. You return an `SseEmitter` and it starts async mode. You then send events to it by calling `emitter.send()`. Each event goes directly to the client. You control when events are sent based on your application logic.

The emitter stays open until you call `complete()` or the client disconnects or a timeout occurs. This pattern is perfect for real-time dashboards, live notifications, progress updates, or any push-based communication pattern.

## Content Negotiation in Body Writing

For handlers that write response bodies - `@ResponseBody` methods, `ResponseEntity`, async types that resolve to values - Spring performs content negotiation to determine how to serialize the return value.

The process starts with the Accept header from the request. This header tells Spring what media types the client can understand. It might be `application/json`, `application/xml`, `text/html`, or `*/*` for "anything."

Spring then determines what media types it can produce. If your `@RequestMapping` has a `produces` attribute, that constrains the options. Otherwise, Spring looks at all registered `HttpMessageConverter` instances and asks each one: "can you write this return value type?" The converters that say yes contribute their supported media types to the producible set.

Spring finds the intersection - media types that are both acceptable to the client and producible by Spring. It sorts this intersection by specificity and quality values from the Accept header. It picks the most specific, highest quality match.

Then it walks through the message converters again, asking: "can you write this value as this specific media type?" The first converter that says yes wins. Spring calls its `write()` method to serialize the value to the response.

If no converter can handle the selected media type, Spring throws `HttpMediaTypeNotAcceptableException`, which becomes HTTP 406 Not Acceptable. This tells the client: "I understood your request, but I can't produce a response in a format you'll accept."

This content negotiation is why the same endpoint can return JSON to one client and XML to another based on their Accept headers, as long as you have both Jackson and JAXB converters registered.

## ResponseBodyAdvice - Intercepting Response Writing

`ResponseBodyAdvice` is a powerful hook that lets you intercept the response writing process right before the message converter serializes the body. You can modify the body, wrap it in a standard envelope, add metadata, or even suppress it entirely.

You implement two methods. The `supports()` method indicates whether your advice should apply to a particular return type and converter combination. The `beforeBodyWrite()` method receives the body about to be written and can transform it.

This is commonly used for standard API response wrapping. Your handlers return domain objects, but your API specification requires all responses wrapped in a standard envelope with success flag, timestamp, and data field. Your `ResponseBodyAdvice` implementation wraps every response automatically.

You can also use it to add response headers. The `beforeBodyWrite()` method receives the `ServerHttpResponse` which gives you access to headers. You could add custom headers to every response without modifying every handler method.

One critical detail: if you return null from `beforeBodyWrite()`, Spring treats that as "don't write a body at all." The response will have status and headers but no content. This is occasionally useful but usually a mistake when you meant to return the original body unchanged.

`ResponseBodyAdvice` implementations are registered through `@ControllerAdvice` classes. Spring finds all `@ControllerAdvice` beans that implement `ResponseBodyAdvice` and includes them in the response processing chain.

## Map Returns and Their Dual Nature

Map return values have dual behavior depending on whether `@ResponseBody` is present, just like String returns.

In a `@Controller` without `@ResponseBody`, when you return `Map<String, Object>`, the `MapMethodProcessor` handles it. It calls `mavContainer.addAllAttributes(map)` to add all the map entries as model attributes. Then it leaves `requestHandled=false`. Spring derives a view name and renders a view with those model attributes available.

This is occasionally useful for handlers that want to return multiple model attributes without explicitly adding them one by one. But it's confusing because the Map itself doesn't appear in the model - its entries do.

In a `@RestController`, the class-level `@ResponseBody` causes `RequestResponseBodyMethodProcessor` to claim the Map return. It serializes the Map to JSON using Jackson. The response body contains the JSON representation of the map. This is much more common for REST APIs.

The annotation context completely changes what a Map return means. Without it, it's model attributes. With it, it's JSON. This is why removing `@RestController` from a REST controller and replacing it with `@Controller` causes every Map-returning method to break - they'll try to add to the model and render views instead of returning JSON.

## HttpHeaders Return Type

You can return just headers with no body by returning `HttpHeaders`. The `HttpHeadersReturnValueHandler` takes all the headers and sets them on the response, then sets `requestHandled=true` with no body written.

This is useful for HEAD requests where the client only wants headers, not content. Or for OPTIONS requests where you want to send allowed methods and other metadata without a body.

You construct an `HttpHeaders` object, add whatever headers you need - Content-Length, Content-Type, ETag, Cache-Control, custom headers - and return it. Spring sets them all and completes the request.

## The View Return Type

You can return an actual `View` object instead of a view name String. The `ViewMethodReturnValueHandler` handles this by calling `mavContainer.setView(view)` with the actual View object.

This bypasses view name resolution entirely. You're providing the actual View implementation that will render the response. This is rarely needed because view name resolution is usually sufficient, but it gives you maximum control when you need it.

You might use this when you're dynamically selecting between different view implementations based on complex logic, or when you're creating custom View implementations for special rendering needs.

## Error Handling and Return Value Processing

When return value handling fails - maybe no handler supports the return type, maybe content negotiation fails, maybe a message converter throws an exception - Spring's exception handling machinery takes over.

If no handler supports the return type, you get `IllegalArgumentException` which becomes HTTP 500. This indicates a programming error in your controller - you returned something Spring doesn't know how to handle.

If content negotiation fails to find a compatible media type, you get `HttpMediaTypeNotAcceptableException` which becomes HTTP 406. The client asked for a format you can't produce.

If a message converter fails during serialization, you get `HttpMessageNotWritableException` which becomes HTTP 500. The converter couldn't serialize your object - maybe it's not serializable, maybe there's a circular reference, maybe a field type isn't supported.

These errors indicate different kinds of problems and produce different status codes to help clients understand what went wrong.

## Custom Return Value Handlers

You can create custom return value handlers just like custom argument resolvers. You implement `HandlerMethodReturnValueHandler` with `supportsReturnType()` and `handleReturnValue()`.

Your `supportsReturnType()` method examines the `MethodParameter` (which represents the return type) and decides if you claim it. You can check the type, check annotations on the method, check the declaring class - whatever you need.

Your `handleReturnValue()` method does whatever makes sense for that return type. Maybe you serialize it in a custom format. Maybe you write it to a message queue and return a receipt. Maybe you trigger a batch job and return its ID. You have complete control.

You register custom handlers through `WebMvcConfigurer.addReturnValueHandlers()`. Just like with argument resolvers, your handlers are appended after all built-ins. If a built-in claims the return type first, your handler never runs.

If you need to intercept before built-ins, you'd have to access the `RequestMappingHandlerAdapter` directly and replace its entire handler list. This is rarely needed.

## The Complete Processing Flow

Let me tie everything together with the complete flow from method return to HTTP response.

Your handler method executes and returns a value. Spring wraps this value along with the return type metadata into objects it can work with. It walks the return value handler chain, calling `supportsReturnType()` on each handler until one returns true. The cached handler for that return type is used on subsequent requests.

The winning handler's `handleReturnValue()` method executes. This method might write directly to the response (REST cases) or set up the model and view (traditional MVC cases). It definitely sets `requestHandled` appropriately.

The `DispatcherServlet` checks `requestHandled`. If true, it's done - the response is complete. If false, it proceeds with view resolution.

For view resolution, it uses the view name or View object from the `ModelAndViewContainer`. It finds a `ViewResolver` that can resolve that view name to a View object. It calls `render()` on the View, passing the model. The View produces output (HTML, JSON, whatever) and writes it to the response.

For REST cases with `requestHandled=true`, the handler already wrote the response during `handleReturnValue()`. For async cases, the initial return sets up async processing and releases the thread. When the async result arrives, Spring dispatches a second request that processes the actual result.

Throughout this flow, various extension points let you customize behavior. `ResponseBodyAdvice` intercepts before serialization. Custom handlers let you support new return types. Message converters let you serialize in custom formats. View resolvers let you use different template technologies.

---

# Final Summary for Mastery and Recall

Let me synthesize the essential principles you need to carry forward.

**The requestHandled flag is the entire REST vs MVC split**. Handlers that write responses directly set it to true and view resolution never happens. Handlers that set up model and view leave it false and Spring renders the view. This one boolean, set by return value handlers, determines the fundamental behavior of the endpoint. Understanding this flag explains why `@ResponseBody` methods don't render views and why void methods sometimes do.

**String return meaning changes completely with ResponseBody**. In `@Controller`, String is a view name processed by `ViewNameMethodReturnValueHandler` and rendered through ViewResolver. In `@RestController`, String is response body text processed by `RequestResponseBodyMethodProcessor` and written via `StringHttpMessageConverter`. The string "redirect:/home" in a REST controller is literal text in the response, not a redirect instruction. Context is everything.

**ResponseEntity gives complete control, ResponseBody only controls body**. With `@ResponseBody` alone, you can control what gets serialized but the status is always 200 (or whatever `@ResponseStatus` statically declares) and you cannot set custom headers. `ResponseEntity` lets you programmatically set status, every header, and the body. For REST APIs needing dynamic status codes or custom headers, `ResponseEntity` is essential.

**Void returns have three completely different behaviors**. In `@RestController`, void sets `requestHandled=true` with no body - useful when writing directly to response. In `@Controller` with committed response, same behavior. In `@Controller` with uncommitted response, view name is derived from URL and rendering proceeds. The third case surprises developers who expect void to mean "no view." It doesn't - it means "derive view from URL."

**Async return types release the servlet thread immediately**. `Callable`, `DeferredResult`, `CompletableFuture`, and reactive types all start async processing that releases the thread before the response is complete. The response completes later via a second dispatch. This changes when interceptor `postHandle()` methods run - they run on the second dispatch, not immediately after your handler. Long-running operations should use async types to avoid tying up servlet threads.

**Content negotiation selects both media type and converter**. Spring compares what the client accepts (Accept header) with what it can produce (from `@RequestMapping produces` or converter capabilities). It picks the best match, then finds a converter that can write that type as that media type. If negotiation succeeds but no converter exists, you get 406 Not Acceptable. Configure converters to support the media types your API needs.

**The handler chain order encodes precedence**. Type-specific handlers like `ModelAndViewMethodReturnValueHandler` come first and have highest priority. `RequestResponseBodyMethodProcessor` checking for `@ResponseBody` comes before view-related handlers, which is why `@ResponseBody` wins over String-as-view-name interpretation. Custom handlers added via configuration go last and cannot override built-in behavior.

**Map return without ResponseBody goes to model, with ResponseBody goes to JSON**. An unannotated `Map<String, Object>` return in `@Controller` calls `MapMethodProcessor` which adds map entries to the model for view rendering. The same Map in `@RestController` calls `RequestResponseBodyMethodProcessor` which serializes it to JSON. The annotation context completely changes behavior - removing `@RestController` breaks all Map-returning REST endpoints.

**DeferredResult timeout default value is a result, not an error**. Creating `DeferredResult` with just a timeout throws an exception on timeout. Creating it with a timeout and default value returns that default value as a normal result when timeout occurs. It's processed as if your code called `setResult()`. This lets you provide fallback responses for slow operations without error handling.

**ResponseBodyAdvice can transform or suppress bodies before serialization**. Implementing `beforeBodyWrite()` lets you wrap responses in standard envelopes, add metadata, or even return null to suppress the body entirely. It runs after handler execution but before the message converter serializes, giving you a chance to standardize all REST responses. Applied via `@ControllerAdvice` to affect multiple controllers.

**HttpEntityMethodProcessor extracts status and headers before writing body**. When processing `ResponseEntity`, it sets status first, then headers, then serializes body. This order matters because headers must be set before the response is committed by writing content. Trying to set headers after writing starts will fail silently or throw exceptions.

Understanding return value handling means understanding the journey from your method's return statement to bytes in the HTTP response. Different return types take different paths through the system. Some write directly and set `requestHandled=true`. Some set up model and view for later rendering. Some start async processing that completes later. The handler chain, guided by type and annotations, determines which path each return value takes. Master this routing and you master Spring MVC's output side.
