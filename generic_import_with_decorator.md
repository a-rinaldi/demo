The code provided defines a **configuration class** for Spring that creates a bean called importRowsStrategies. The Format **enum** is used in the importRowsStrategies bean to map each format to a specific implementation of the ImportRowsStrategy **interface**. Overall, the strategy pattern is used to provide a flexible way to import rows from different file formats, while the decorator pattern is used to add an extra layer of protection to the import operation.

```java
@Bean
public ImportRowsStrategies importRowsStrategies(
        CompaniesRepository companiesRepository,
        PlatformTransactionManager transactionManager,
        ExecutorService executor,
        NotificationService notificationService,
        EventHub eventHub,
        MessageSource messageSource,
        PlatformTransactionManager platformTransactionManager
) {
    final Map<Format, ImportRowsStrategy> strategies = Maps.<Format, ImportRowsStrategy>builder()
            .add(Format.XLSX, new PremiumOnlyStrategyDecorator(new XlsxImportRowsStrategy(), companiesRepository, platformTransactionManager))
            .add(Format.CSV, new PremiumOnlyStrategyDecorator(new CsvImportRowsStrategy(companiesRepository, notificationService, eventHub, transactionManager, executor, messageSource), companiesRepository, platformTransactionManager))
            .toMap();
    return new ImportRowsStrategies(strategies);
}
```

The purpose of this bean is to provide a way to import rows from a CSV or XLSX file in a transactional manner.
```java
public class ImportRowsStrategies {
    
    private final Map<Format, ImportRowsStrategy> strategies;

    public ImportRowsStrategies(Map<Format, ImportRowsStrategy> strategies) {
        this.strategies = strategies;
    }

    public ImportRowsStrategy get(Format format) {
        return Optional.ofNullable(strategies.get(format))
                .orElseThrow(() -> new IllegalArgumentException(String.format("No strategies available for format %s", format)));
    }
    
}
```
The PremiumOnlyStrategyDecorator class uses the **Decorator** pattern to add the premium access check feature to the ImportRowsStrategy interface.
```java
public class PremiumOnlyStrategyDecorator implements ExportStrategy, ImportRowsStrategy {

    private final PlatformTransactionManager platformTransactionManager;
    private ExportStrategy innerExportStrategy;
    private ImportRowsStrategy innerImportStrategy;
    private final CompaniesRepository companiesRepository;

    public PremiumOnlyStrategyDecorator(ExportStrategy inner, CompaniesRepository companiesRepository, PlatformTransactionManager platformTransactionManager) {
        this.innerExportStrategy = inner;
        this.companiesRepository = companiesRepository;
        this.platformTransactionManager = platformTransactionManager;
    }

    public PremiumOnlyStrategyDecorator(ImportRowsStrategy inner, CompaniesRepository companiesRepository, PlatformTransactionManager platformTransactionManager) {
        this.innerImportStrategy = inner;
        this.companiesRepository = companiesRepository;
        this.platformTransactionManager = platformTransactionManager;
    }

    private void checkPremium(User user) {
        final TransactionTemplate tx = new TransactionTemplate(platformTransactionManager);
        tx.setPropagationBehavior(TransactionTemplate.PROPAGATION_REQUIRES_NEW);
        final Company company = tx.execute((status) -> {
            return companiesRepository.load(user.companyId);
        });
        if (!isAdmin(user.role) && !company.premium) {
            throw new PremiumFeatureUnavailableException("Premium feature");
        }
    }

    @Override
    public <T extends ImportRowsEntityInterface, REPO> void importStream(
            Class<T> schemaClazz,
            EventType eventType,
            REPO repo,
            InputStream is,
            User user,
            int companyId,
            Optional<Character> delimiter) {
        checkPremium(user);
        innerImportStrategy.importStream(schemaClazz, eventType, repo, is, user, companyId, delimiter);
    }
}
```

The method **importStream** is declared as generic because it can accept any type of rows as long as they implement the ImportRowsEntityInterface **interface**. This interface defines a contract for objects that can be imported via a stream of data. The use of generics allows the importStream method to be flexible and reusable. The method is executed asynchronously in order to improve performance. 

```java
public CsvImportRowsStrategy(
            CompaniesRepository companiesRepository,
            NotificationService notificationService,
            EventHub eventHub,
            PlatformTransactionManager platformTransactionManager,
            ExecutorService executor,
            MessageSource messageSource
    ) {
        this.companiesRepository = companiesRepository;
        this.platformTransactionManager = platformTransactionManager;
        this.executor = executor;
        this.eventHub = eventHub;
        this.notificationService = notificationService;
        this.messageSource = messageSource;
    }

    @Override
    public <T extends ImportRowsEntityInterface, REPO> void importStream(
            Class<T> schemaClazz,
            EventType eventType,
            final REPO repo,
            InputStream is,
            User user,
            final int companyId,
            Optional<Character> delimiter) {
        ExecutorService otherExecutor = Executors.newFixedThreadPool(OTHER_EXECUTOR_THREAD);
        final TransactionTemplate tx = new TransactionTemplate(platformTransactionManager);
        tx.setPropagationBehavior(TransactionTemplate.PROPAGATION_REQUIRES_NEW);
        final CsvMapper csvMapper = new CsvMapper();
        final CsvSchema schema = CsvSchema.emptySchema()
                .withHeader()
                .withNullValue("")
                .withQuoteChar('"')
                .withColumnSeparator(delimiter.map(d -> d).orElseGet(() -> ','));
        try {
            logger.info("Trying to import value for class: %s", schemaClazz);
            Company company = tx.execute((status) -> {
                return companiesRepository.load(companyId);
            });
            final MappingIterator<T> readValues = csvMapper
                    .configure(MapperFeature.ACCEPT_CASE_INSENSITIVE_PROPERTIES, true)
                    .enable(DeserializationFeature.READ_UNKNOWN_ENUM_VALUES_USING_DEFAULT_VALUE)
                    .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
                    .readerFor(schemaClazz).with(schema).readValues(is);
            StreamSupport.stream(Spliterators.spliteratorUnknownSize(readValues, 0), true)
                    .collect(parallel(entity -> {
                        /*This represent the batch executed BATCH times*/
                        try {
                            return tx.execute((status) -> {
                                return entity.save(repo, companyId);
                            });
                        } catch (Exception ex) {
                            Locale locale = new Locale(company.language);
                            entity.setError(ex, messageSource, locale);
                            return entity;
                        }
                    }, toList(), executor, BATCH))
                    .orTimeout(100000, MILLISECONDS)
                    /*Consuming the result. Send notification.*/
                    .handleAsync((entityList, ex) -> {
                        if (ex != null) {
                            logger.error("Got generic exception: %s", ex);
                            eventHub.notifyListeners(Event.event(EventType.IMPORT_ERROR, ex), companyId);
                            return null;
                        }
                        List<String> errorList = entityList.stream()
                                .filter(e -> schemaClazz.isInstance(e))
                                .map(e -> e.toString())
                                .collect(Collectors.toList());
                        ImportResponse importResponse = ImportResponse.of(entityList.size(), errorList.size());
                        eventHub.notifyListeners(Event.event(eventType, importResponse), companyId);
                        final Map<String, Object> model = new HashMap<>();
                        model.put("importResponse", importResponse);
                        model.put("company", company);
                        model.put("errors", errorList);
                        notificationService.sendInstitutionalEmail(
                                String.format("import-%s", company.language),
                                EmailAddress.of(user.email, user.name),
                                model,
                                Optional.empty());
                        return entityList;
                    }, otherExecutor);
        } catch (IOException ex) {
            logger.error("Unable to process CSV import", ex);
            throw new Failure(Problem.of("IMPORT_ERROR", ex.getLocalizedMessage()));
        }
    }
}
```
Click to watch the final output [![video](https://img.youtube.com/vi/2y6faw4qO-M/maxresdefault.jpg)](https://www.youtube.com/watch?v=2y6faw4qO-M)
