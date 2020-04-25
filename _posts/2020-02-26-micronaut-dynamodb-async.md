---
layout: default
title: "DynamoDB Client using Micronaut and GraalVM"
date:   2020-04-10 21:14
description: How to build Async DynamoDB client with Micronaut and Maven, build a native image with GraalVM  
repository_url: https://github.com/byegor/poc/tree/master/micronaut-dynamodb-async
tags: [java, aws, microservice, micronaut]
---
{:refdef: style="text-align: center;"}
![](/assets/micronaut-dynamodb.png)
{: refdef}

# 1. Overview

It will be a simple how-to article where I will be showing how to implement simple rest DynamoDB client using [Micronaut Framework](https://micronaut.io/) and Maven,
build a native image with [GraalVM](https://www.graalvm.org/) and simple comparison in resource usage between 
clients on Spring Boot and on Micronaut with GraalVM.

For those who are not familiar with Micronaut - it is a framework for building microservices and serverless applications.
One of the key differences between Spring Boot and Micronaut is that Micronaut doesn't use reflection to do IoC, 
so application startup time and memory consumption are not bound to the size of project codebase.

So our task is to handle HTTP requests for retrieving or storing some `Event(id:string, body: string)`. 
Events will be stored in DynamoDB.

It might be easier to simply look at the [code on Github](https://github.com/byegor/poc/tree/master/micronaut-dynamodb-async) and follow it there.

# 2. Maven 

Let's start with Maven runtime dependencies for Micronaut and DynamoDB SDK
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.micronaut</groupId>
            <artifactId>micronaut-bom</artifactId>
            <version>${micronaut.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>io.micronaut</groupId>
        <artifactId>micronaut-inject-java</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micronaut</groupId>
        <artifactId>micronaut-runtime</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micronaut</groupId>
        <artifactId>micronaut-http-server-netty</artifactId>
    </dependency>

    <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-java-sdk-dynamodb</artifactId>
        <version>1.11.762</version>
    </dependency>
</dependencies>
```

As Micronaut doesn't use reflection/annotation processing during startup but does it during build - 
we need to add annotation processors to maven-compiler-plugin.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.8.1</version>
    <configuration>
        <compilerArgs>
            <arg>-parameters</arg>
        </compilerArgs>
        <annotationProcessorPaths>
            <path>
                <groupId>io.micronaut</groupId>
                <artifactId>micronaut-inject-java</artifactId>
                <version>${micronaut.version}</version>
            </path>
            <path>
                <groupId>io.micronaut</groupId>
                <artifactId>micronaut-validation</artifactId>
                <version>${micronaut.version}</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
    <executions>
        <execution>
            <id>test-compile</id>
            <goals>
                <goal>testCompile</goal>
            </goals>
            <configuration>
                <compilerArgs>
                    <arg>-parameters</arg>
                </compilerArgs>
                <annotationProcessorPaths>
                    <path>
                        <groupId>io.micronaut</groupId>
                        <artifactId>micronaut-inject-java</artifactId>
                        <version>${micronaut.version}</version>
                    </path>
                    <path>
                        <groupId>io.micronaut</groupId>
                        <artifactId>micronaut-validation</artifactId>
                        <version>${micronaut.version}</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </execution>
    </executions>
</plugin>
```  

# 3. DynamoDB

###### 3.1 Configuration

A simple config where we set up connection to DynamoDB. For test purpose we need to specify `dynamoEndpoint`.
In case of real application we need to specify region instead of endpoint.

```java
@Factory
public class Config {

    @Bean
    AmazonDynamoDBAsync dynamoDbAsyncClient(Environment environment) {
        Optional<String> secretKey = environment.get("aws.secretkey", String.class);
        Optional<String> accessKey = environment.get("aws.accesskey", String.class);
        String endpoint = environment.get("dynamo.endpoint", String.class, "http://localhost:8000");
        if (!secretKey.isPresent() || !accessKey.isPresent()) {
            throw new IllegalArgumentException("Aws credentials not provided");
        }
        BasicAWSCredentials credentials = new BasicAWSCredentials(accessKey.get(), secretKey.get());
        AmazonDynamoDBAsyncClientBuilder clientBuilder = AmazonDynamoDBAsyncClientBuilder.standard()
                .withCredentials(new AWSStaticCredentialsProvider(credentials))
                .withEndpointConfiguration(
                        new AwsClientBuilder.EndpointConfiguration(endpoint, null)
                );

        return clientBuilder.build();
    }
}
```

###### 3.2 Async DynamoDB Service

Simple service for saving/retrieving event to/from DynamoDB. All async requests to aws are wrapped into RxJava 
constructions for easy handling of futures. 

```java
@Singleton
public class DynamoDBService {

    public static final String TABLE_NAME = "events";
    public static final String ID_COLUMN = "id";
    public static final String BODY_COLUMN = "body";

    private final AmazonDynamoDBAsync client;

    public DynamoDBService(AmazonDynamoDBAsync client) {
        this.client = client;
    }

    //Create DynamoDB table if not exists
    @PostConstruct
    public void createTableIfNotExists() {
        if (!isTableExists()) {
            createTable();
        }
    }
    
    public Maybe<Event> getEvent(String eventId) {
        Map<String, AttributeValue> searchCriteria = new HashMap<>();
        searchCriteria.put(ID_COLUMN, new AttributeValue().withS(eventId));
        // Building request to get event by Id
        GetItemRequest request = new GetItemRequest()
                .withTableName(TABLE_NAME)
                .withKey(searchCriteria)
                .withAttributesToGet(BODY_COLUMN); // lets retrieve only body as id we already have
        return Maybe.fromFuture(client.getItemAsync(request))
                .subscribeOn(Schedulers.io())
                .filter(result -> result.getItem() != null) // check that request returned something
                .map(result -> new Event(eventId, result.getItem().get(BODY_COLUMN).getS())); //building Event from response
    }

    public Single<String> saveEvent(String eventBody) {
        String id = UUID.randomUUID().toString();

        Map<String, AttributeValue> item = new HashMap<>();
        item.put(ID_COLUMN, new AttributeValue().withS(id));
        item.put(BODY_COLUMN, new AttributeValue().withS(eventBody));

        PutItemRequest putRequest = new PutItemRequest()
                .withTableName(TABLE_NAME)
                .withItem(item);

        return Single.fromFuture(client.putItemAsync(putRequest))
                .subscribeOn(Schedulers.io())
                .map(result -> id);
    }

    private boolean isTableExists() {
        ListTablesRequest tablesRequest = new ListTablesRequest()
                .withExclusiveStartTableName(TABLE_NAME);
        ListTablesResult result = client.listTables(tablesRequest);
        return result.getTableNames().contains(TABLE_NAME);
    }

    private CreateTableResult createTable() {
        KeySchemaElement keyDefinitions = new KeySchemaElement()
                .withAttributeName(ID_COLUMN)
                .withKeyType(KeyType.HASH);

        AttributeDefinition keyType = new AttributeDefinition()
                .withAttributeName(ID_COLUMN)
                .withAttributeType(ScalarAttributeType.S);

        CreateTableRequest request = new CreateTableRequest()
                .withTableName(TABLE_NAME)
                .withKeySchema(keyDefinitions)
                .withAttributeDefinitions(keyType)
                .withBillingMode(BillingMode.PAY_PER_REQUEST);

        return client.createTable(request);
    }
}
```

# 4. Controller

Here we gonna expose our REST Api with GET method for retrieving event from DynamoDB and POST for storing event. 
```java
@Controller("/event")
public class SimpleController {

    private final DynamoDBService dynamoDBService;

    public SimpleController(DynamoDBService dynamoDBService) {
        this.dynamoDBService = dynamoDBService;
    }

    @Get("/{eventId}")
    @Produces(MediaType.APPLICATION_JSON) 
    public Maybe<Event> getEvent(@PathVariable String eventId) {
        Maybe<Event> event = dynamoDBService.getEvent(eventId);
        return event;
    }

    @Post("/")
    @Produces(MediaType.APPLICATION_JSON)
    public Single<String> saveEvent(@Body String body) {
        Single<String> event = dynamoDBService.saveEvent(body);
        return event;
    }
}
```

# 5. Integration Test

###### 5.1 Maven dependencies
For running integration test with DynamoDB we need DynamoDBLocal dependency, which is not really the DynamoDB, 
but SQLite with implemented DynamoDB interfaces on top of it.
```xml
 <dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>DynamoDBLocal</artifactId>
    <version>1.12.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.micronaut</groupId>
    <artifactId>micronaut-http-client</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.micronaut.test</groupId>
    <artifactId>micronaut-test-junit5</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-api</artifactId>
    <version>5.6.0</version>
    <scope>test</scope>
</dependency>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-dependency-plugin</artifactId>
            <version>2.10</version>
            <executions>
                <execution>
                    <id>copy</id>
                    <phase>test-compile</phase>
                    <goals>
                        <goal>copy-dependencies</goal>
                    </goals>
                    <configuration>
                        <includeScope>test</includeScope>
                        <includeTypes>so,dll,dylib</includeTypes>
                        <!--Keep an eye on output directory - it will be used for starting dynamodb-->
                        <outputDirectory>${project.basedir}/target/native-libs</outputDirectory>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>

<repositories>
    <repository>
        <id>dynamodb-local-oregon</id>
        <name>DynamoDB Local Release Repository</name>
        <url>https://s3-us-west-2.amazonaws.com/dynamodb-local/release</url>
    </repository>
</repositories>
```
###### 5.2 DynamoDB server
Now we need to start DynamoDB before test runs, we can do it with jupiter Extension.

```java
public class LocalDynamoDbExtension implements AfterAllCallback, BeforeAllCallback {

    protected DynamoDBProxyServer server;

    public LocalDynamoDbExtension() {
        //here we set the path from "outputDirectory" of maven-dependency-plugin
        System.setProperty("sqlite4java.library.path", "target/native-libs");
    }

    @Override
    public void afterAll(ExtensionContext extensionContext) throws Exception {
        stopUnchecked(server);
    }

    @Override
    public void beforeAll(ExtensionContext extensionContext) throws Exception {
        this.server = ServerRunner
                .createServerFromCommandLineArgs(new String[]{"-inMemory", "-port", "8000"});
        server.start();
    }

    protected void stopUnchecked(DynamoDBProxyServer dynamoDbServer) {
        try {
            dynamoDbServer.stop();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```
###### 5.3 Running test
Now we can create an integration test and check if our REST Api methods do what we think.
```java
@MicronautTest
@ExtendWith(LocalDynamoDbExtension.class)
public class SimpleControllerTest {

    @Inject
    @Client("/event")
    RxStreamingHttpClient client;

    @Inject
    DynamoDBService dynamoDBService;

    @Test
    public void getEventsTest() {
        //add event to database so we can query it via http
        String eventBody = "testMessage";
        String eventId = dynamoDBService.saveEvent(eventBody).blockingGet();
        HttpRequest request = HttpRequest.GET(eventId);
        HttpResponse<List<Event>> rsp = client.toBlocking().exchange(request, Argument.listOf(Event.class));

        assertEquals(HttpStatus.OK, rsp.getStatus());
        List<Event> body = rsp.body();
        assertEquals(1, body.size());
        assertEquals(eventBody, body.get(0).getBody());
    }

    @Test
    public void saveEventTest() {
        HttpRequest request = HttpRequest.POST("/", "postBody");
        HttpResponse<String> rsp = client.toBlocking().exchange(request, Argument.of(String.class));
        Optional<String> id = rsp.getBody();
        assertTrue(id.isPresent());

        Event event = dynamoDBService.getEvent(id.get()).blockingGet();
        assertEquals(id.get(), event.getId());
        assertEquals("postBody", event.getBody());
    }
}
```
# 6. Native Image

Using [GraalVM](https://www.graalvm.org/docs/why-graal/) we can build ahead-of-time compiled native image which is very 
useful for small applications. Native image  includes an application classes, 
classes from its dependencies, classes from JDK. It does not run on the JVM. So in the end you will get standalone 
executable image which you can run without any JVM. 

Because image already compiled, linked and partly initialized it will start faster, and you will get 
lower memory footprint.
But keep in mind that there is a price for that - absence of JIT compiler, much simpler GC(SerialGC), platform dependent,
hard to use frameworks which heavily relies on reflection(Spring Framework).
       
You can build image in several ways:

###### 6.1 Building image with Docker multistage
One prons is that you need to know application's classpath or download all dependencies in folder and point to it during building image.
```dockerfile
FROM oracle/graalvm-ce:20.0.0-java11 as graalvm
RUN gu install native-image

COPY . /home/app/micronaut-dynamodb-client
WORKDIR /home/app/micronaut-dynamodb-client

RUN native-image --no-server -cp all-runtime-deps.jar

FROM frolvlad/alpine-glibc
RUN apk update && apk add libstdc++
EXPOSE 8080
COPY --from=graalvm /home/app/micronaut-dynamodb-client/micronaut-dynamodb-client /srv/micronaut-dynamodb-client
ENTRYPOINT ["/srv/micronaut-dynamodb-client", "-Xmx68m"]
```

###### 6.2 Building image with Maven

A bit harder than with a docker. You need to install GrallVM JDK, install native-image tool, set GraalVM as JDK for your project. 
After that you can add a plugin to maven and plugin will do the job.
```xml
<plugin>
    <groupId>org.graalvm.nativeimage</groupId>
    <artifactId>native-image-maven-plugin</artifactId>
    <version>20.0.0</version>
    <executions>
        <execution>
            <goals>
                <goal>native-image</goal>
            </goals>
            <phase>deploy</phase>
        </execution>
    </executions>
    <configuration>
        <mainClass>com.yegor.micronaut.dynamodb.App</mainClass>
        <buildArgs>-H:Name=dynamodb-client</buildArgs> <!--Image name-->
        <buildArgs>-H:IncludeResources="logback.xml|application.yml"</buildArgs> <!--Resources to add to image-->
    </configuration>
</plugin>
```

When native image is ready we can build Docker image with it.

```dockerfile
FROM frolvlad/alpine-glibc
RUN apk update && apk add libstdc++
COPY target/micronaut-dynamodb-client /srv/micronaut-dynamodb-client
EXPOSE 8080
ENTRYPOINT ["/srv/micronaut-dynamodb-client"]
```

# 7. Simple comparision between dockerized Native Image and dockerized Spring Boot app

As an example I'm gonna take spring boot application from [this post]({% link _posts/2020-02-21-spring-dynamodb-async.md %}) 
which is basically doing the same stuff but with a help of Spring.

###### 7.1 Image Sizes
First lets look at image sizes by running `docker images`
```
REPOSITORY                  TAG          SIZE
micronaut-dynamodb-native   latest       84.4MB
spring-boot-dynamodb        latest       364MB
```
Obviously, docker with native image uses less space, cause the native image removed everything that won't be in use, 
including JVM

###### 7.2 App startup

I run each image and just look at logs to get info when application completed startup.
Micronaut-Native-Image started in 54 ms. Pretty impressive :)
```
io.micronaut.runtime.Micronaut - Startup completed in 54ms. Server Running: http://5330567cbd7c:8080
```

Spring Boot application took much longer to start
```
com.example.dynamo_spring.App   : Started App in 4.093 seconds (JVM running for 4.736)
```

###### 7.3 Memory Footprint

To print Memory and CPU consumption run `docker stats`. But as I don't do any requests to images - CPU numbers are irrelevant.

```
NAME                            MEM USAGE                    
micronaut-dynamodb-native       12.63MiB    
spring-boot-dynamodb            152MiB                     
```
___

<br/><br/> 

Hurray, you made till the end!

Happy coding :)

