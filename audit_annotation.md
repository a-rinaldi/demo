This snippet is used to keep track of changes made to entities (repositories agnostic) and to provide a record of these changes for auditing purposes through aspect-oriented programming (AOP), specifically the **AspectJ** framework.
The **EventAnnotation** interface is used to annotate methods in the Repository interface that require logging. The annotations in the EventAnnotation interface specify the type of event being logged, such as the creation of a new entity or the editing of an existing entity. The Class parameter specifies the class of the entity being logged. 
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(value = {ElementType.METHOD})
@Inherited
@Documented
public @interface EventAnnotation {

    EventReference mainEntity() default EventReference.NO_REFERENCE;
    
    TypeMap[] types() default {};

    DescriptionMap[] descriptions() default {};

    Class<? extends EventResponseInterface> clazz() default EventResponseInterface.class;
    
    boolean read() default false;

    public @interface TypeMap {
        public KeyEnum key();
        public EventType value();
    }
    
    public @interface DescriptionMap {
        public KeyEnum key();
        public String value();
    }
    
    public enum KeyEnum{
        NEW,
        EDIT,
        REMOVE
    }
    
    boolean companyPushNotification() default false;
    
    boolean customerPushNotification() default false;
    
    OperatorRole[] pushForRoles() default {};

}
```
The event is logged using the **AfterReturning** pointcut, which is triggered after a method with the **EventAnnotation** annotation is executed.
The **EventAspect** class contains the processEvent method, which extracts the information about the event from the annotation and the return value of the annotated method. This information includes the type of event, the description of the event, and the reference to the entity being logged. The information is used to create a log entry that is sent to the **EventsFacade**, which is responsible for processing the log entries. The **EventHub** and **PushApiService** classes are also used to process the log entries and send push notifications to relevant parties.

```java
@Aspect
@Component
public class EventAspect {

    @Autowired
    private CompanyEventsFacade eventsFacade;
    @Autowired
    private EventHub eventHub;
    @Autowired
    private MessageSource messageSource;
    @Autowired
    private RegionalSetup regionalSetup;
    @Value("${development}")
    boolean development;
    @Autowired
    private SessionFactory hibernate;
    @Autowired
    private PushApiService pushService;
    @Autowired
    private CompaniesFacade companiesFacade;
    @Autowired
    private CustomersFacade customersFacade;
    @Autowired
    private ObjectMapper objectMapper;

    private final ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

    @AfterReturning(
            pointcut = "@annotation(EventAnnotation)",
            returning = "result")
    public void processEvent(JoinPoint joinPoint, Object result) {
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.initialize();
        final MethodSignature methodSignature = (MethodSignature) joinPoint
                .getSignature();
        Method method = methodSignature.getMethod();
        /*Annotated arg for reference*/
        final EventAnnotation event = method.getAnnotation(EventAnnotation.class);
        final boolean isCompanyPushNotification = event.companyPushNotification();
        final boolean isCustomerPushNotification = event.customerPushNotification();
        final boolean entityIsMainReference = event.mainEntity().equals(EventReference.NO_REFERENCE);
        final EventAnnotation.TypeMap[] types = event.types();
        final OperatorRole[] pushForRoles = event.pushForRoles();
        final boolean read = event.read();
        EventAnnotation.DescriptionMap[] descriptions = event.descriptions();
        Class<? extends EventResponseInterface> clazz = event.clazz();
        Optional<? extends EventResponseInterface> request = Stream.of(joinPoint.getArgs())
                .filter(EventResponseInterface.class::isInstance)
                .map(clazz::cast)
                .findFirst();
        EventResponseInterface response = Optional.ofNullable(result)
                .map(clazz::cast)
                .orElseThrow(() -> new Failure(Problem.of("UNEXPECTED_CAST", String.format("Result can not be casted to %s", clazz.getCanonicalName()))));
        boolean isSaleEvent = response.isSalesEvent();
        EventReference reference = response.getEventReference();
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        String principalName = Optional.ofNullable(authentication)
                .map(a -> a.getName())
                .orElseGet(() -> "Scheduled/Automated Job");
        EventType type = request
                .flatMap(r -> (entityIsMainReference ? r.getReferenceId() : r.getReferencedId()).flatMap(id -> Stream.of(types)
                .filter(e -> e.key().equals(EventAnnotation.KeyEnum.EDIT))
                .map(e -> e.value())
                .findFirst()))
                .orElseGet(() -> types[0].value());
        String message = request
                .flatMap(r -> (entityIsMainReference ? r.getReferenceId() : r.getReferencedId()).flatMap(id -> Stream.of(descriptions)
                .filter(e -> e.key().equals(EventAnnotation.KeyEnum.EDIT))
                .map(e -> e.value())
                .findFirst()))
                .orElseGet(() -> descriptions[0].value());

        Locale locale = new Locale(regionalSetup.instanceLanguage.id);
        MessageFormat formatter = new MessageFormat(messageSource.getMessage(message, null, locale), locale);
        String description = formatter.format(new Object[]{response.getMessageSourceParameters()});

        /*Must be inside a transaction as we are setting the hook within the repositories*/
        eventsFacade.save(
                response.getCompanyId(),
                type,
                read,
                isSaleEvent,
                Optional.ofNullable(description),
                principalName,
                reference,
                response.getReferenceId());
                
        /*Ensure the event is notified to listeners only AFTER transaction is committed*/
        hibernate.getCurrentSession().getTransaction().registerSynchronization(new Synchronization() {
            @Override
            public void beforeCompletion() {
            }

            @Override
            public void afterCompletion(final int status) {
                if (status == Status.STATUS_COMMITTED && !isSaleEvent) {
                    CompanyResponse company = companiesFacade.load(response.getCompanyId());
                    /*mapping proxied entities result in exception in completable future as no session is present.*/
                    Object mappedResponse = response.getMappedResponse();
                    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
                    Optional<User> maybeAuthenticatedUser = Optional.ofNullable(authentication)
                            .map(auth -> {
                                /*AnonymousAuthenticationToken returned if principal does not have a role. 
                                reference: https://docs.spring.io/spring-security/site/docs/4.0.x/reference/html/anonymous.html*/
                                if (auth instanceof AnonymousAuthenticationToken) {
                                    return null;
                                }
                                return (User) auth.getPrincipal();
                            });
                    /*
                    let the operation complete, then asyncronusly push the event and notification. Push subscriptions for a company may be a long list.
                    Worst case 30 operators, each of them with 5 devices = 150 push notification
                     */
                    final CompletableFuture<Boolean> future = CompletableFuture.supplyAsync(() -> {
                        eventHub.notifyListeners(Event.event(type, mappedResponse), response.getCompanyId());
                        try {
                            /*this could be used to save the action instead of the current implementation*/
                            String message = response.getMeaningfulMessage(locale, type);
                            if (isCompanyPushNotification) {
                                pushService.sendPushMessage(company.pushSubscriptions, message.getBytes(), pushForRoles, maybeAuthenticatedUser);
                            }
                            /* Notifications are only for marketplace customer
                            Notification can be addressed for both company and customer 
                            (eg: booking changed by company operator A. Operator B and user SHOULD receive notification)*/
                            if (isCustomerPushNotification) {
                                response.getOptionalCustomerId().ifPresent((customerId) -> {
                                    CustomerResponse customer = customersFacade.loadById(customerId).orElseThrow(() -> new Failure(Problem.of("NOT_FOUND", "Customer not found")));
                                    try {
                                        if (customer.marketplaceCustomer != null) {
                                            /*ngsw expects a json payload*/
                                            String jsonNotification = objectMapper.writeValueAsString(PushApiNotification.of(Notification.of("Help4u", message, "h4u-logo-192X192", "h4u-badge")));
                                            pushService.sendPushMessage(customer.marketplaceCustomer.pushSubscriptions, jsonNotification.getBytes(), maybeAuthenticatedUser);
                                        }
                                    } catch (JsonProcessingException ex) {
                                        /*noop*/
                                    }
                                });
                            }
                        } catch (NoSuchMessageException ex) {
                            /*noop*/
                        }
                        return true;
                    }, executor);
                    future.exceptionally(ex -> {
                        logger.warn("Unable to send events/push notification '%s'", ex);
                        return false;
                    });
                } else if (status == Status.STATUS_COMMITTED && isSaleEvent) {
                    eventHub.notifyAdminListeners(Event.event(type, response.getMappedResponse()));
                }
            }
        });

    }
}
```
Final output [![video](https://img.youtube.com/vi/D25yQMO2nGk/maxresdefault.jpg)](https://www.youtube.com/watch?v=D25yQMO2nGk)

