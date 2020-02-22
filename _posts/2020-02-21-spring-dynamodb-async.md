---
layout: default
title:  "Spring Webflux with Async DynamoDB"
date:   2020-02-21 15:14:54
tags:   ["spring","dynamodb", "async"]
repository_url: https://github.com/yegor-bond/poc/tree/master/spring-dynamodb-async 
---
## Overview

Starting from Spring Boot 2.0 (Spring framework 5.0), the framework provides support for asynchronous programming, 
so does AWS SDK starting with 2.0 version. In this post i will be exploring using asynchronous DynamoDB API 
and Spring Webflux by building simple reactive REST application. 
Let’s say we need to handle HTTP requests for retrieving or storing some Event(id:string, body: string). 
Event will be stored in DynamoDB.

It might be easier to simply look at the [code on Github](https://github.com/yegor-bond/poc/tree/master/spring-dynamodb-async) and follow it there.

## Dependencies
Let's start with Maven dependencies for WebFlux and DynamoDB SDK
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
        <version>2.2.4.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>dynamodb</artifactId>
        <version>2.10.65</version>
    </dependency>
</dependencies>
```
## DynamoDB

### Spring configuration

```java
@Configuration
public class AppConfig {
    @Value("${aws.accessKey}")
    String accessKey;

    @Value("${aws.secretKey}")
    String secretKey;

    @Value("${dynamodb.endpoint:}")
    String dynamoEndpoint;

    @Bean
    AwsBasicCredentials awsBasicCredentials(){
        return AwsBasicCredentials.create(accessKey, secretKey);
    }

    @Bean
    DynamoDbAsyncClient dynamoDbAsyncClient(AwsBasicCredentials awsBasicCredentials){
        DynamoDbAsyncClientBuilder clientBuilder = DynamoDbAsyncClient.builder();
        clientBuilder
                .credentialsProvider(StaticCredentialsProvider.create(awsBasicCredentials));
                if(!dynamoEndpoint.isEmpty()){
                    clientBuilder.endpointOverride(URI.create(dynamoEndpoint));
                }
        return clientBuilder.build();
    }
}
```
### Reactive DynamoDB service
Unfortunately, second version of AWS SDK doesn’t have support for DynamoDBMapper yet
(you can track mapper’s readiness [here](https://github.com/aws/aws-sdk-java-v2/issues/35)), 
so table creation, sending requests and parsing responses need to be done by “low level” API.
Of course all operations implemented in async way.
```java
@Service
public class DynamoDbService {

    public static final String TABLE_NAME = "events";
    public static final String ID_COLUMN = "id";
    public static final String BODY_COLUMN = "body";

    final DynamoDbAsyncClient client;

    @Autowired
    public DynamoDbService(DynamoDbAsyncClient client) {
        this.client = client;
    }

    //Creating table on startup if not exists
    @PostConstruct
    public void createTableIfNeeded() throws ExecutionException, InterruptedException {
        ListTablesRequest request = ListTablesRequest.builder().exclusiveStartTableName(TABLE_NAME).build();
        CompletableFuture<ListTablesResponse> listTableResponse = client.listTables(request);

        CompletableFuture<CreateTableResponse> createTableRequest = listTableResponse
                .thenCompose(response -> {
                    boolean tableExist = response.tableNames().contains(TABLE_NAME);
                    if (!tableExist) {
                        return createTable();
                    } else {
                        return CompletableFuture.completedFuture(null);
                    }
                });

        //Wait in synchronous manner for table creation
        createTableRequest.get();
    }

    public CompletableFuture<PutItemResponse> saveEvent(Event event) {
        Map<String, AttributeValue> item = new HashMap<>();
        item.put(ID_COLUMN, AttributeValue.builder().s(event.getUuid()).build());
        item.put(BODY_COLUMN, AttributeValue.builder().s(event.getBody()).build());

        PutItemRequest putItemRequest = PutItemRequest.builder()
                .tableName(TABLE_NAME)
                .item(item)
                .build();

        return client.putItem(putItemRequest);
    }

    public CompletableFuture<Event> getEvent(String id) {
        Map<String, AttributeValue> key = new HashMap<>();
        key.put(ID_COLUMN, AttributeValue.builder().s(id).build());

        GetItemRequest getRequest = GetItemRequest.builder()
                .tableName(TABLE_NAME)
                .key(key)
                .attributesToGet(BODY_COLUMN)
                .build();

        return client.getItem(getRequest).thenApply(item -> {
            if (!item.hasItem()) {
                return null;
            } else {
                Map<String, AttributeValue> itemAttr = item.item();
                String body = itemAttr.get(BODY_COLUMN).s();
                return new Event(id, body);
            }
        });
    }

    private CompletableFuture<CreateTableResponse> createTable() {

        CreateTableRequest request = CreateTableRequest.builder()
                .tableName(TABLE_NAME)

                .keySchema(KeySchemaElement.builder().attributeName(ID_COLUMN).keyType(KeyType.HASH).build())
                .attributeDefinitions(AttributeDefinition.builder().attributeName(ID_COLUMN).attributeType(ScalarAttributeType.S).build())
                .billingMode(BillingMode.PAY_PER_REQUEST)
                .build();

        return client.createTable(request);
    }
}
```
## Reactive REST Controller
A simple controller with GET method for retrieving event by id and POST method for saving events in DynamoDB. 
We can do it in two ways - implement it with annotations or get rid of annotations and do it in functional way.
There is no performance impact, in 99% cases it is absolutely based on individual preference what to use. 
##### Annotated Controllers
```java
@RestController
@RequestMapping("/event")
public class AnnotatedController {

    final DynamoDbService dynamoDbService;

    public AnnotatedController(DynamoDbService dynamoDbService) {
        this.dynamoDbService = dynamoDbService;
    }

    @GetMapping(value = "/{eventId}", produces = MediaType.APPLICATION_JSON_VALUE)
    public Mono<Event> getEvent(@PathVariable String eventId) {
        CompletableFuture<Event> eventFuture = dynamoDbService.getEvent(eventId);
        return Mono.fromCompletionStage(eventFuture);
    }

    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE)
    public void saveEvent(@RequestBody Event event) {
        dynamoDbService.saveEvent(event);
    }
}
```
##### Functional Endpoints
This is a lightweight functional programming model in which functions are used to route and handle requests.
```java
@Configuration
public class HttpRouter {

    @Bean
    public RouterFunction<ServerResponse> eventRouter(DynamoDbService dynamoDbService) {
        EventHandler eventHandler = new EventHandler(dynamoDbService);
        return RouterFunctions
                .route(GET("/eventfn/{id}")
                        .and(accept(APPLICATION_JSON)), eventHandler::getEvent)
                .andRoute(POST("/eventfn")
                        .and(accept(APPLICATION_JSON))
                        .and(contentType(APPLICATION_JSON)), eventHandler::saveEvent);
    }

    static class EventHandler {
        private final DynamoDbService dynamoDbService;

        public EventHandler(DynamoDbService dynamoDbService) {
            this.dynamoDbService = dynamoDbService;
        }

        Mono<ServerResponse> getEvent(ServerRequest request) {
            String eventId = request.pathVariable("id");
            CompletableFuture<Event> eventGetFuture = dynamoDbService.getEvent(eventId);
            Mono<Event> eventMono = Mono.fromFuture(eventGetFuture);
            return ServerResponse.ok().body(eventMono, Event.class);
        }

        Mono<ServerResponse> saveEvent(ServerRequest request) {
            Mono<Event> eventMono = request.bodyToMono(Event.class);
            eventMono.map(dynamoDbService::saveEvent);
            return ServerResponse.ok().build();
        }
    }
}
```