# Spring Execution Contexts and Hook Points

See also: [spring-request-lifecycle.md](spring-request-lifecycle.md) for the hooks along a single HTTP request's path specifically (filters, interceptors, AOP, `ResponseBodyAdvice`), and [../system-design/message-queue-vs-pubsub.md](../system-design/message-queue-vs-pubsub.md) for the queue vs pub/sub distinction referenced below.

Spring gives you many places to attach code that runs outside the normal "controller calls service calls repository" call stack — at container startup/shutdown, on another thread, in reaction to an event, or bound to a transaction boundary. This is a survey of those hook points, grouped by what triggers them, plus a dedicated look at the Request Context and how errors are handled (or silently swallowed) in each one.

## 1. Bean lifecycle hooks

Code that runs as a *specific bean* is created or destroyed by the container — once per bean's lifetime, not per request.

| Hook | Fires | Notes |
|---|---|---|
| `@PostConstruct` (JSR-250) | After dependency injection completes, before the bean is put into service | Most common way to run init logic that needs injected fields already set |
| `@PreDestroy` (JSR-250) | Just before the container destroys the bean (e.g. on shutdown) | Cleanup — close a connection pool, flush a buffer |
| `InitializingBean.afterPropertiesSet()` / `DisposableBean.destroy()` | Same two moments, interface-based instead of annotation-based | Rarely used directly now; `@PostConstruct`/`@PreDestroy` are preferred |
| `@Bean(initMethod = "...", destroyMethod = "...")` | Same two moments, for beans you don't own the source of (e.g. a third-party class) | Lets you wire lifecycle methods without editing the class |
| `BeanPostProcessor` | Before/after **every** bean's initialization, container-wide | How Spring itself implements `@Autowired` and AOP proxy creation — write one to intercept *all* beans, not just yours |
| `BeanFactoryPostProcessor` | Even earlier — after bean *definitions* are loaded but before any bean is instantiated | Lets you modify bean definitions themselves (e.g. `${...}` placeholder resolution) |
| `ApplicationRunner` / `CommandLineRunner` | Once, right after the whole `ApplicationContext` has started | Startup tasks: seed data, warm a cache, validate config |
| `SmartLifecycle` | `start()`/`stop()`, with explicit ordering via `getPhase()` | For components that must start *after* other infrastructure is up (e.g. a message listener that shouldn't consume until a cache is warm) |

```java
@Component
class CacheWarmer {
    @PostConstruct
    void warmUp() {
        cache.load(); // runs once, after fields are injected
    }

    @PreDestroy
    void flush() {
        cache.persist(); // runs once, before shutdown
    }
}
```

## 2. Deferred / async execution — "run this elsewhere or later"

| Mechanism | Runs on | Notes |
|---|---|---|
| Raw `Thread` / `ExecutorService.submit()` | A thread you manage yourself | No Spring involvement — you own the pool and its lifecycle |
| `@Async` + `TaskExecutor` | A Spring-managed thread pool | AOP-proxy-based like `@Transactional` — same self-invocation gotcha applies (see below); returns `void`, `Future<T>`, or `CompletableFuture<T>` |
| `CompletableFuture` composition | Whatever executor you chain it with | Plain Java; often used to fan out several `@Async` calls and combine results |
| `@Scheduled` + `TaskScheduler` | A scheduled thread pool, on a cron/fixed-rate/fixed-delay timer | Recurring background jobs, not triggered by any request |
| Servlet `AsyncContext` / Spring `DeferredResult`/`Callable`/`WebAsyncTask` | Request thread released back to Tomcat while work continues elsewhere | Covered as `WebAsyncManager` in [spring-request-lifecycle.md](spring-request-lifecycle.md) |
| Message queue producer/consumer (`@KafkaListener`, `@RabbitListener`, `@JmsListener`) | A different process entirely, possibly a different machine | Survives app restarts — the only option here with durability |

```java
@Service
class ReportService {
    @Async
    CompletableFuture<Report> generate(Long orderId) {
        return CompletableFuture.completedFuture(build(orderId)); // runs on a pooled thread, caller doesn't block
    }
}
```

## 3. Event-driven hooks — "publish, let others react"

| Mechanism | Timing | Notes |
|---|---|---|
| `ApplicationEventPublisher.publishEvent()` + `@EventListener` | Synchronous, same thread, same transaction, by default | Decouples publisher from listener — the publisher doesn't know or care who's listening |
| `@EventListener` + `@Async` | Asynchronous, on a pooled thread | Combine both annotations to make a specific listener non-blocking |
| `@TransactionalEventListener(phase = ...)` | Deferred until a transaction boundary: `BEFORE_COMMIT`, `AFTER_COMMIT` (default), `AFTER_ROLLBACK`, `AFTER_COMPLETION` | The right tool when a listener should only run *if* the originating transaction actually succeeds |

```java
@Service
class OrderService {
    @Transactional
    Order placeOrder(OrderRequest req) {
        Order order = repository.save(new Order(req));
        publisher.publishEvent(new OrderPlacedEvent(order.getId())); // fires immediately, same tx
        return order;
    }
}

@Component
class EmailListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    void onOrderPlaced(OrderPlacedEvent event) {
        emailClient.sendConfirmation(event.orderId()); // only runs if the order actually committed
    }
}
```

## 4. Transaction-boundary hooks (lower-level)

`TransactionSynchronizationManager.registerSynchronization(...)` is what `@TransactionalEventListener` is built on top of — register a `TransactionSynchronization` callback (`beforeCommit`, `afterCommit`, `afterCompletion`) directly, for cases that need this without going through the event system at all.

## 5. Ambient "current context" holders — state to read, not code to define

These aren't hook points you write code *into* — they're thread-bound state you read *from*, wherever you are in the call stack, without it being passed as a parameter.

| Holder | Carries |
|---|---|
| `RequestContextHolder` | The current `HttpServletRequest`/`HttpServletResponse`, locale — accessible from a service layer that has no direct access to the request |
| `SecurityContextHolder` | The current authenticated `Authentication`/principal |
| MDC (SLF4J) | Logging correlation IDs (e.g. a request/trace ID) attached to every log line without threading it through every method signature |
| Reactor `Context` (WebFlux) | The reactive equivalent — not thread-bound, since a reactive chain can hop threads, so it travels with the subscription instead |

**Gotcha**: the thread-bound ones (`RequestContextHolder`, `SecurityContextHolder`, MDC) do **not** automatically propagate into a new thread — spinning up work via `@Async`/`ExecutorService` loses them unless explicitly copied across (e.g. a `TaskDecorator` that wraps the submitted `Runnable` to re-set MDC/SecurityContext on the new thread before running).

### Request Context in detail

`RequestContextHolder` deserves a closer look since it's the one of these four ambient holders that's part of Spring MVC specifically (not a general Spring or logging concern):

- **Populated by**: in a normal Spring MVC app, `DispatcherServlet` publishes it automatically for every request (`publishContext` defaults to `true`) via `RequestContextHolder.setRequestAttributes(...)`, so any `@Controller` → `@Service` → `@Repository` call chain always has it available. Outside `DispatcherServlet`'s reach — a filter that runs *before* it, or code on a non-request thread — register `RequestContextListener` (a `ServletContextListener`, auto-registered by Spring Boot's web starter) or add `RequestContextFilter` explicitly earlier in the chain.
- **Holds**: a `ServletRequestAttributes` wrapping the current `HttpServletRequest`/`HttpServletResponse`, exposing two attribute scopes — `RequestAttributes.SCOPE_REQUEST` (cleared at the end of the request) and `SCOPE_SESSION` (backed by the `HttpSession`).
- **Relation to `@RequestScope` beans (section 6 below)**: a `@RequestScope` bean *is* this mechanism — Spring stores the bean instance as a request attribute under `SCOPE_REQUEST`, and the scoped proxy injected into your singleton resolves through `RequestContextHolder` on every method call. `RequestContextHolder` is the primitive; `@RequestScope` is a convenience built on top of it.

```java
@Service
class AuditService {
    void logAccess() {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        log.info("access from {}", request.getRemoteAddr()); // no need to pass HttpServletRequest through every layer
    }
}
```

- **Failure mode**: calling `RequestContextHolder.currentRequestAttributes()` with no request bound to the current thread (e.g. inside a `@Scheduled` job, a `@KafkaListener`, or a plain `@Async` method without propagation) throws `IllegalStateException: No thread-bound request found` — see "How errors are handled in each context" below.

## 6. Context-scoped beans

Bean scopes tied to a container-recognized lifecycle rather than the default singleton: `@RequestScope` (one instance per HTTP request), `@SessionScope` (per HTTP session), `@ApplicationScope` (effectively singleton, but explicit), `@RefreshScope` (Spring Cloud Config — rebuilds the bean when config changes).

## How errors are handled in each context

Every hook point above has a different default failure mode — some fail loudly and stop the app, some are silently swallowed unless you go looking, and some only affect part of the pipeline. This matters more than which mechanism to reach for.

1. **Bean lifecycle hooks**: an exception from `@PostConstruct`, `@PreDestroy` (see exception below), `InitializingBean.afterPropertiesSet()`, `BeanPostProcessor`, or `BeanFactoryPostProcessor` fails that bean's creation and — since this happens during context startup — takes down the whole `ApplicationContext` (`BeanCreationException`); the app never finishes starting. `ApplicationRunner`/`CommandLineRunner` throwing does the same, since `SpringApplication.run()` rethrows it. The one exception: `@PreDestroy` failures happen during *shutdown*, so Spring just logs them and continues destroying the remaining beans — there's nothing left to fail. Nothing in this category retries automatically.
2. **Deferred/async execution** — this is where "who sees the exception" varies most:
   - Raw `Thread`: an uncaught exception kills that thread silently (default handler just prints to stderr); nothing tells the code that started it. `ExecutorService.submit(Runnable)` behaves the same; `submit(Callable)` instead captures it in the returned `Future`, surfacing only when `.get()` is called (as `ExecutionException`).
   - `@Async void` methods: the caller already returned, so the exception can't go back to them — it's routed to `AsyncUncaughtExceptionHandler` (default just logs it). Override `AsyncConfigurer.getAsyncUncaughtExceptionHandler()` to change that.
   - `@Async` methods returning `Future<T>`/`CompletableFuture<T>`: the exception is captured in the future and only surfaces if the caller calls `.get()` or chains `.exceptionally()`/`.handle()` — ignore the future and the failure is invisible.
   - `@Scheduled`: caught and logged by the default `LoggingErrorHandler`; the schedule keeps firing on its next cycle regardless — one failed run doesn't cancel future ones. Customize via `ScheduledTaskRegistrar.setErrorHandler(...)`.
   - Message listeners (`@KafkaListener`/`@RabbitListener`/`@JmsListener`): governed entirely by the container's configured error handler and ack mode — typically retry/backoff, then either a dead-letter topic/queue (if configured) or the message is discarded/logged. There's no language-level default here; it's whatever your container factory is configured to do.
3. **Event-driven hooks**:
   - `@EventListener` (synchronous, default): the exception propagates straight back up to whatever called `publishEvent()` — same call stack, so it can roll back a `@Transactional` publisher exactly like any other exception thrown inline. **Gotcha**: `SimpleApplicationEventMulticaster` stops at the first listener that throws — any other listeners for that same event that haven't run yet are simply skipped, not just the failing one.
   - `@EventListener` + `@Async`: same as any `@Async void` method above — goes to `AsyncUncaughtExceptionHandler`, never reaches the publisher.
   - `@TransactionalEventListener(AFTER_COMMIT)` (the default phase): runs after the transaction has already committed, so a failure here **cannot** roll anything back — the transaction manager catches and just logs an exception thrown from an after-commit synchronization callback.
4. **Transaction-boundary hooks** (`TransactionSynchronizationManager.registerSynchronization`): timing determines the blast radius — an exception from `beforeCommit()` can still abort the commit (turning it into a rollback); an exception from `afterCommit()`/`afterCompletion()` is just logged, since the outcome is already fixed by then.
5. **Ambient context holders**: not really "handled" so much as failing in a characteristic way when misused. `RequestContextHolder.currentRequestAttributes()` throws `IllegalStateException: No thread-bound request found` when called off a request thread — a hard, immediate failure. `SecurityContextHolder.getContext().getAuthentication()` fails softer: with no authenticated user it typically returns an anonymous token rather than throwing, so a missing-auth bug shows up as "silently acting as anonymous" rather than a crash — easy to miss.
6. **Context-scoped beans**: since `@RequestScope`/`@SessionScope` beans resolve through the same request-attribute mechanism as `RequestContextHolder`, injecting one into a singleton and calling it outside a real request thread fails the same way — `BeanCreationException` wrapping `IllegalStateException: No thread-bound request found` — thrown at the scoped-proxy's first method call, not at injection time.

Exception handling for filters, interceptors, the controller, and AOP-advised service calls along the HTTP request path itself is covered in full in [spring-request-lifecycle.md](spring-request-lifecycle.md)'s "The exception path" section — not repeated here.

## Centralized handling per context — what plays the role of `@ExceptionHandler`

For the request path, `@RestControllerAdvice` + `@ExceptionHandler` is the answer to "where do I put exception-handling logic once, instead of a try/catch in every controller method." Each other context has its own answer to that same question — some are just as centralized, some have no such thing and force a local try/catch instead.

| Context | Centralized mechanism | Registered how | How it compares to `@ExceptionHandler` |
|---|---|---|---|
| Request path (baseline) | `@ExceptionHandler` in a `@RestControllerAdvice` | Auto-detected component, dispatched by exception type | The reference point — matched globally across (or per-controller within) the app |
| `@Async void` methods | `AsyncUncaughtExceptionHandler` | One bean, via `AsyncConfigurer.getAsyncUncaughtExceptionHandler()` | Just as centralized — one implementation covers every `@Async void` method app-wide. Does **not** cover `Future`/`CompletableFuture`-returning methods; those still need `.exceptionally()`/`.handle()` at each call site |
| `@Scheduled` methods | A custom `ErrorHandler` | `SchedulingConfigurer.configureTasks(registrar)` → `registrar.setErrorHandler(...)` | Centralized the same way — set once, applies to every scheduled task registered through it |
| `@KafkaListener` / `@RabbitListener` / `@JmsListener` | `CommonErrorHandler` (Kafka) / `RabbitListenerErrorHandler` / `ErrorHandler` (JMS) | Set on the listener container factory (applies to every listener using that factory), or per-method via `@KafkaListener(errorHandler = "...")`/`@RabbitListener(errorHandler = "...")` | The closest true analogue — a dedicated bean type, centrally wired, that can decide retry vs. dead-letter vs. discard, much like `@ExceptionHandler` decides the HTTP response |
| `@EventListener` (sync) | No dedicated bean type | Either try/catch inside each listener (so one listener's failure doesn't stop others, given the "first exception wins" gotcha above), or replace the default `ApplicationEventMulticaster` bean to change that stop-on-first-failure behavior globally | Weaker support than the others — there's no annotation-driven equivalent, only a lower-level container override |
| Bean lifecycle hooks / runners | `ApplicationListener<ApplicationFailedEvent>` | Registered like any listener bean | Observational only — it fires *after* startup has already failed, so it's for logging/alerting before the JVM exits, not recovery. The only real fix is a try/catch inside the `@PostConstruct`/runner method itself |
| Transaction sync hooks / `@TransactionalEventListener` | None | Try/catch inside the callback/listener itself | No framework-level hook exists; Spring's logging of after-commit failures is the extent of built-in support |

```java
// @Async void / Future — centralized handler, mirrors @RestControllerAdvice's role for the async world
@Configuration
@EnableAsync
class AsyncConfig implements AsyncConfigurer {
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> log.error("Async method {} failed", method.getName(), ex);
    }
}

// @Scheduled — centralized handler for every scheduled task
@Configuration
@EnableScheduling
class SchedulingConfig implements SchedulingConfigurer {
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        registrar.setErrorHandler(ex -> log.error("Scheduled task failed", ex));
    }
}

// @KafkaListener — closest analogue to @RestControllerAdvice: decides retry vs. dead-letter, not just logging
@Bean
ConcurrentKafkaListenerContainerFactory<?, ?> kafkaListenerContainerFactory() {
    var factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setCommonErrorHandler(new DefaultErrorHandler(new DeadLetterPublishingRecoverer(kafkaTemplate)));
    return factory; // applies to every @KafkaListener using this factory
}
```

## Deciding between async/decoupling mechanisms

Several of the above solve the same underlying problem — "don't do this work inline, right now, on this thread" — but fit different constraints.

| Question | Real-life example (Thread/`@Async`) | Real-life example (`ApplicationEventPublisher`) | Real-life example (Message queue) |
|---|---|---|---|
| Does the work need to survive this JVM crashing/restarting? | No — a lost report-generation task is just retried by the user | No — same process, same lifetime as the app | Yes — an order-confirmation email queued in Kafka is still there after a deploy restarts the consumer |
| Does another *process/service* need to react, not just another class in the same app? | No — stays in-process | No — stays in-process | Yes — e.g. a separate inventory service reacting to `OrderPlaced` |
| Do you need built-in retry/backoff/dead-letter handling for failures? | No — you'd hand-roll retry logic yourself | No — a failed listener just throws, caught by whatever called `publishEvent` | Yes — most brokers give you redelivery and DLQs out of the box |
| Is it just "don't block the caller while this finishes," nothing more? | Yes — the simplest, lowest-ceremony option | Overkill for this alone | Overkill for this alone |
| Do you want to decouple "what happened" from "who cares," so new reactions can be added later without touching the publisher? | No — caller and callee stay directly coupled | Yes — that's exactly what `@EventListener` decoupling buys you | Yes, and it works across service boundaries too |

## Real-life analogy

Think of a company. `@PostConstruct`/`@PreDestroy` are a new hire's onboarding checklist and exit interview — they run exactly once, tied to that one employee's lifecycle, not to anything a customer does. `BeanPostProcessor` is HR policy applied to *every* new hire automatically (badge photo, laptop provisioning), regardless of role. `@Async`/`ExecutorService` is delegating a task to a coworker and moving on with your day — you don't wait around. `ApplicationEventPublisher` is posting a notice on the company bulletin board ("the Q3 numbers are in") — you don't know or care who reads it, and anyone who joins later doesn't get notified of past notices. `@TransactionalEventListener(AFTER_COMMIT)` is that same notice, but held back until the board meeting that approves the Q3 numbers actually concludes successfully — no point announcing numbers that later get rejected. A message queue is mailing a physical letter to a supplier in a different building entirely — it survives even if your building briefly loses power, and the supplier processes it on their own schedule. `RequestContextHolder`/`SecurityContextHolder`/MDC are like a visitor badge that silently identifies "who's currently walking the halls" to anyone who checks, without that person re-introducing themselves at every door — but if they walk into a *different* building (a new thread), the badge doesn't automatically follow unless someone hands them a fresh one.

## Common gotchas

- **`@Async` self-invocation**: same proxy issue as `@Transactional` — calling an `@Async` method from another method in the *same class* bypasses the proxy and runs synchronously on the caller's thread.
- **Thread-bound context doesn't cross thread boundaries for free**: `SecurityContextHolder`/MDC/`RequestContextHolder` need explicit propagation (`TaskDecorator`, `DelegatingSecurityContextExecutor`) into `@Async`/`ExecutorService` work — otherwise the new thread sees no user, no request, no correlation ID.
- **`BeanFactoryPostProcessor` vs `BeanPostProcessor`**: easy to confuse by name — the former runs on bean *definitions* before any bean exists; the latter runs on actual bean *instances* during initialization.
- See "How errors are handled in each context" above for exception-propagation specifics per mechanism (`@EventListener`'s same-transaction propagation, `@Async`'s silent-by-default failures, etc.) — not repeated here.
