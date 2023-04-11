The conversation method annotated with @Bean creates an instance of **DialogflowConversation** using the projectId and tokenFile values provided as input parameters.
```java
@Bean
    public Conversation conversation(
            @Value("${connector.dialogflow.projectId}") String projectId,
            @Value("${connector.dialogflow.tokenfile}") String tokenFile
    ) throws IOException {
        final Credentials credentials = ServiceAccountCredentials.fromStream(new FileInputStream(tokenFile));
        return new DialogflowConversation(credentials, projectId);
    }
```
The code below defines a class called DialogflowConversation which implements the Conversation interface. The class allows to interact with the Dialogflow API, which is a natural language understanding platform developed by Google. The query method of the class accepts a natural language query and a language code as input, and returns a list of maps representing the fulfillment messages from Dialogflow.
```java
public class DialogflowConversation implements Conversation {

    private final SessionsClient client;
    private final String projectId;

    public DialogflowConversation(Credentials credentials, String projectId) {
        try {
            final CredentialsProvider credentialsProvider = FixedCredentialsProvider.create(credentials);
            final SessionsSettings sessionsSettings = SessionsSettings.newBuilder().setCredentialsProvider(credentialsProvider).build();
            this.client = SessionsClient.create(sessionsSettings);
        } catch (IOException ex) {
            throw new IllegalStateException("Unable to instantiate Dialogflow Sessions Client", ex);
        }
        this.projectId = projectId;
    }

    public List<Map<String, Object>> query(String query, String language) {
        final String sessionId = UUID.randomUUID().toString();
        final SessionName session = SessionName.of(projectId, sessionId);
        final TextInput textInput = TextInput.newBuilder().setText(query).setLanguageCode(language).build();
        final QueryInput queryInput = QueryInput.newBuilder().setText(textInput).build();
        final DetectIntentResponse response = client.detectIntent(session, queryInput);
        final QueryResult result = response.getQueryResult();
        return result.getFulfillmentMessagesList().stream()
              .map((message) -> message.getPayload()).map(this::map).collect(Collectors.toList());
    }
}
```
Filan output [video](https://www.youtube.com/watch?v=GK5pCqQ5YKM)
