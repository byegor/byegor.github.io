---
layout: default
title:  "Spring Webflux with Async DynamoDB"
date:   2020-02-21 15:14:54
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

    public DynamoDbService(DynamoDbAsyncClient client) {
        this.client = client;
    }

    @PostConstruct
    public void createTableIfNeeded() throws ExecutionException, InterruptedException {
        // check if table is already present
        ListTablesRequest request = ListTablesRequest.builder().exclusiveStartTableName(TABLE_NAME).build();
        CompletableFuture<ListTablesResponse> listTableResponse = client.listTables(request);

        CompletableFuture<CreateTableResponse> createTableRequest = listTableResponse
                .thenCompose(response -> {
                    boolean tableExist = response.tableNames().contains(TABLE_NAME);
                    if (!tableExist) {
                        //if table not present - create it
                        return createTable();
                    } else {
                        return CompletableFuture.completedFuture(null);
                    }
                });
        // wait for table creation
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

    public CompletableFuture<Optional<Event>> getEvent(String id) {
        Map<String, AttributeValue> key = new HashMap<>();
        key.put(ID_COLUMN, AttributeValue.builder().s(id).build());

        GetItemRequest getRequest = GetItemRequest.builder()
                .tableName(TABLE_NAME)
                .key(key)
                .attributesToGet(BODY_COLUMN)
                .build();

        return client.getItem(getRequest).thenApply(item -> {
            if (!item.hasItem()) {
                return Optional.empty();
            } else {
                Map<String, AttributeValue> itemAttr = item.item();
                String body = itemAttr.get(BODY_COLUMN).s();
                return Optional.of(new Event(id, body));
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