# Gradle

```
testImplementation 'io.github.cjstehno.ersatz:ersatz:4.0.1'
testImplementation 'io.github.cjstehno.ersatz:ersatz-groovy:4.0.1'
```

# Test

## ErsatzServer

```
@ExtendWith(ErsatzServerExtension.class)
class HelloTest{
    @Test
    void sayHello(final ErsatzServer server) throws Exception {
        server.expectations(expect -> {
            expect.GET("/say/hello", req -> {
                req.called(1);
                req.query("name", "Ersatz");
                req.responder(res -> {
                    res.body("Hello, Ersatz", TEXT_PLAIN);
                });
            });
        });

    final var request = HttpRequest
        .newBuilder(new URI(server.httpUrl("/say/hello?name=Ersatz")))
        .GET()
        .build();

    final var response = newHttpClient().send(request, ofString());

    assertEquals(200, response.statusCode());
    assertEquals("Hello, Ersatz", response.body());
    assertTrue(server.verify());

    }
}
```

## Ersatz-groovy server

- configuration syntax a bit cleaner and provide additional Groovy DSL support

```
@ExtendWith(ErsatzServerExtension)
class HelloGroovyTest {
    @Test
    void 'say hello'(final GroovyErsatzServer server) {
        server.expectation{
            GET('/say/hello') {
                called 1
                query 'name', 'Ersatz'
                responder {
                    body 'Hello, Ersatz', TEXT_PLAIN
                }
            }
        }

         final var request = HttpRequest
        .newBuilder(new URI(server.httpUrl('/say/hello?name=Ersatz')))
        .GET()
        .build()

        final var response = newHttpClient().send(request, ofString())

        assertEquals 200, response.statusCode()
        assertEquals 'Hello, Ersatz', response.body()
        assertTrue server.verfiy()
    }
}
```

# JUnit Extension

## ErsatzServerExtension

- a configurable server management extension that creates and destroys the server after each test method

- main two test lifecycle hooks:

  - beforeEach: before each test method is executed, the server will be created, configured and started.
  - afterEach: After each test method is done executing, the server will be stopped, and the expectations cleared.

- configuratio of the test server can take one of the following forms (in order of precedence)

  - Annotated Method
    This option involves annotating a test method with `@ApplyServerConfig(method)` where `method` refers to the method that will populate a `ServerConfig` instance.

  ```
  @Test
  @ApplyServerConfig("configureServer")
  void testMethod(ErsatzServer server){}
  void configureServer(ServerConfig config){
    config.https(8443)
  }
  ```

  - Annotated Class
    Similar to the annotated method approach, but here the test class itseld is annotated with `@ApplyServerConfig(method)`

  ```
  @ApplyServerConfig("configureServer")
  class MyTestClass {
    @Test
    void testMethod(ErsatzServer server){}
    void configureServer(ServerConfig config) {
        config.https(8443);
    }
  }
  ```

  - Type Field without value
    This optiono involves adding a field of type `ErsatzServer` to the test class without providing an explicit value

  ```
  class MyTestClass {
    ErsatzServer server;

    @Test
    void testMethod() {
        // Test logic using 'server` field
    }
  }
  ```

  - Typed Field with value
    Similar to the previous option, but here the field is initialized with a specific server instance

  ```
  class MyTestClass {
    ErsatzServer server = new ErsatzServer();
    @Test
    void testMethod(){
        // Test logic using 'server field
    }
  }
  ```

  - No configuration
    If no explicit configuration is provided, the test framework will create an instance of `ErsatzServer` automatically

  ```
  class MyTestClass {
    @Test
    void testMethod(ErsatzServer server) {
        // Test logic using automatically created 'server' instance
    }
  }
  ```

  ### Example

  ```
  @ExtendWith(ErsatzServerExtension.class)
  @ApplyServerConfig("defaultConfig")
  class SomeTesting {
    @Test void defaultTest(final ErsatzServer server){
        // do some testing with the server - defaultConfig
    }

    @Test @ApplyServerConfig("anotherConfig)
    void anotherTest(final ErsatzServer server) {
        // do some more testing with the server - anotherConfig
    }

    private void defaultConfig(final ServerConfig conf) {
        // apply some config
    }

    private void anotherConfig(final ServerConfig conf) {
        // apply some other config
    }
  }
  ```

## SharedErsatzServerExtension

- a server management extension that creates the server at the start of the test class, and destroys it when all tests are done
- this extension differ from the other in that it creates the server instane in `beforeAll` and destroys it in `afterAll`, so that there is one server running and available for all of the tests in the class. This cuts down on the startup and shutdown time, at the cost of being able to configure the server for each test method.
- The server expectations are cleared after each test method completes (e.g. `afterEach`)
- configuration of server instance may be done using a class-level `@ApplyServerConfig` annotation, or, if none is provided, the default configuration will ve used tocreate the server.
- configuration method must be static
- test methods should add an `ErsatzServer` or `GroovyErsatzServer` typed parameter to the test methods that require access to the server instance
- A simple example:

```
@ExtendWith(SharedErsatzServerExtension.class)
@ApplyServerConfig
class SomeTesting {
    @Test void defaultTest(final ErsatzServer server) {
        // do some testing with the server - defaultConfig
    }

    private static void serverConfig(final ServerConfig conf) {
        // apply some config
    }
}
```

# Server lifecycle

## configuration

- first lifecycle state where server is instantiated, request expectations are configured and the server is started

- global decoders and encoders may also be configured with the server, as such they will be used as defaults acress all configured expectations

- at this point, there is no HTTP server running, and it is ready for further configuration, as well specifying the request expectations (using the `expectations(...)` and `expects()` methods)

- Once the request expectations are configured, if auto-start is enabled (the default), the server will automatically start. If auto-start is disabled (using autoStart(false)), the server will need to be started using the `start()` method. If the server is not started, you will receive connection errors during testing.

```
public class ErsatzServerConfigurationExample {
    @Test
    void testErsatzServerConfiguration() {
        // Configure the server with a consumer for server configuration
        ErsatzServer server = new ErsatzServer(config -> {
            // Configure server settings
            config.https(8443);
            config.autoStart(true);
            config.defaultReponseContentType(ContentType.TEXT_PLAIN);
        });

        // Configure global decoders and encoders if needed
        server.globalDecoders(decoders -> {

        });

        server.globalEncoders(encoders -> {

        });

        // Configure request expectations
        server.expectations(expect -> {
            expect.GET("/api/resource")
                .respond()
                .status(200)
                .body("Hello, World!", ContentType.TEXT_PLAIN);
        });

        assertTrue(server.isConfigured());
    }
}
```

## Matching

- second state of the server where request/ response interactions are made against the server

```
final var request = HttpRequest
      .newBuilder(new URI(server.httpUrl("/say/hello?name=Ersatz")))
      .GET()
      .build();

final var response = newHttpClient().send(request, ofString());
```

## Verification

- Once the testing has performed, it may be desirable to verify whether the expected requests were matched the expected number of times (using the `Request::called(...)` methods) rather than just that they were called at all.

- to execute verification, one the `ErsatzServer::verify(...)` must be called, which will return a boolean value of true if the verification passed.

```
{
  server.expectation(expect -> {
    expect.GET("/api/resource")
      .respond()
      .status(200)
      .body("Hello, World!". ContentType.TEXT_PLAIN)
      .called(2); // Expect this request to be valled twice
  });

  server.start();

  HttpClient client = HttpClient.newHttpClient();

  // Make two requests to the server
  makeRequest(client, server);
  makeRequest(client, server);

  // Verify that the server has been called as exptected
  assertTrue(server.verify());

  server.stop();
}

private void makeRequest(HttpClient client, ErsatzServer server) throws Exception {
  HttpRequest request = HttpRequest.newBuilder()
        .uri(new URI(server.httpUrl("/api/resources")))
        .GET()
        .build();

  HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
}

```

## Cleanup

- When all test interactions have completed, the server must be stopped in order to free up resources and close connections.
- this is done by calling `ErsatzServer::stop()` method or its alias `close()`.
- For Spock, you can use `@AutoCleanup` annotation on the ErsatzServer to perform cleanup automatically.
- A stopped server may be restarted, though if you want to clean out expectations, you may want to call the `ErsatzServer::clearExpectations()` method before starting it again.

```
@ExtendWith(ErsatzServerExtension)
class Test {
  @AutoCleanup
  GroovyErsatzServer server
}
```

# Server configuration

- `ServerConfig` interface provides the configuration methods for the server and requests, at the global level.

## Ports

- It is recommended o let the server find the best available port for your run. It starts on an ephemeral port by default.
- There are some cases when you need to explicitly specify the HTTP or HTTPS server port and you can do so in the following manner:

```
final var server = new ErsatzServer(cfg -> {
  cfg.httpPort(1111);
  cfg.httpsPort(2222);
})
```

## Auto-Start

- the auto-start flag is enabled by defaut to allow server to start automatically once the expectations have been applied (e.g. after the `expectations(...)` method has been called.) This removes the need to explicityly call the `start()` method in your tests
- you can disable the auto-start feature by following:

```
final var server = new ErsatzServer(cfg -> {
  cfg.autoStart(false);
});
```

## HTTPS

- The server supports HTTPS requests when the `https()` feature flag is enabled. When enabled, the server will setup both an HTTP and HTTPS listener which will have access to all configured expectations.

```
final var server = new ErsatzServer(cfg -> {
  cfg.https();
});

```

- In order to limit a specific request expectation to HTTP or HTTPS, apply the `secure(boolean)` requeest matcher with a value of `true` for HTTPS and `false` for HTTP, similar to the following:

```
server.expectations(expect -> {
  expect.GET("/something").secure(true).responding("stuff");
});
```

## Keystore

- A keystore is a file that contains keys and certificates used for secure communication over HTTPS. The keys and certificates stored in the keystore are used to authenticate the server to clients and establish a secure connection.

- A supported keystore file may be created usign the following command:

```
./keytool -genkey -alias <NAME> -keyalg RSA -keystore
<FILE_LOCATION>
```

- Keystore then needs to be provided during the server configuration as follows:

```
final var server = new ErsatzSerer(cfg -> {
  cfg.https();
  cfg.keystore(KEYSTORE_URL, KEYSTORE_PASS);
});
```

## Request Timeout

- The server request timeout configuration may be specified using the `timeout(...)` configuration methods.

```
final var server = new ErsatzServer(cfg -> {
  cfg.timeout(15, TimeUnit.SECONDS);
});
```

- This will allow some wiggle room in test with high volumes of data or having complex matching logic to be resolved.

## Repost-to-Console

- If the report-to-console flag is enabled (disabled by default), additional details will be written to the console when request matching fails (in addition to writing it in the logs, as it always does).

```
final var server = new ErsatzServer(cfg -> {
  cfg.reportToConsole();
});
```

- The rendered report would be written to the console similar to following:

```
# Expectations

Expectation 0 (3 matchers):
+ HTTP method matches <GET>
+ Path matches "/say/hello"
X Query string name matches a collection containing "Ersatz"
(3 matchers: 2 matched, 1 failed)
```

## Logging Response Content

- By default, the content of a response is only logged as its length(in bytes).
- If the log-response-content feature flag is enabled, the entire content of the response will be written to the logs. This is helpful when debugging issues with tests.

```
final var server = new ErsatzServer(cfg -> {
  cfg.logResponseContent();
});
```

## Server Threads

- By default (as of 3.1), the underlying server has 2 IO threads and 16 Worker threads configured (based on the recommended configuration for the underlying Undertow server). If you need to configure these values, you can use one of the `serverThreads` methods:

```
final var server = new ErsatzServer(cfg -> {
  cfg.serverThreads(3);
})
```

## Content Transformation

- Content transformation involves converting the body content of HTTP requests amd responses from one format to another. This conversion is achieved using request decoders and response encoders.

- Request Decoders: responsible for converting the incoming request body content into desired format that can be easily compared or processed. For example, if the incoming request body is in JSON format, a request decoder can convert it into a Java object for easier handling in tests.

- Response Encoders: used to convert outgoing response objects into byte array data that can be sent over the HTTP connection. For instance, if the application generates response objects in Java, response encoders can serialize these objects into JSON format before sending them as HTTP responses.

- decoders and encoders can be configured at different levels

- Global Configuration: Decoders and encoders configured at the `ServerConfig` level are considered global. They are applied to all request/response interactions unless overidden by local configurations.

- Local Configuration: Decoders and encoders configured in specific request/response expectations override global configurations for the same content. This allows for fine-grained control over the transformation process for individual interactions.

```
// Global configuration of request decoder
ServerConfig globalConfig = new ServerConfig();
globalConfig.addRequestDecoder(new JsonDecoder());

// Local configuration for response encoder in a specific expectation
ErsatzServer server = new ErsatzServer(cfg -> {
  cfg.expectations(expect -> {
    expect.GET("/api/resource")
      .responder(res -> {
        res.body(new Resource("example"), ContentType.APPLICATION_JSON);
      })
      .encoder(new JsonEncoder());
  });
});

public void handleRequest(Request request) {
  Object requestData = request.decodeBody();
  Resource resource = new Resource("example");
  byte[] responseData = server.encodeResponse(resource);
}
```

## Expectations

- When you setting up a Ersatz server to hande incoming HTTP requests during testing, the server needs to know how to respond to different types of requests. These instructions for how the server should respond to requests are called "expectations".

- Expectations: instructions or rules you set for the server. Each expectatio tells the server that to do when it receives a specific type of HTTP request.

- Configuring Expectations: you set up expectations using `Expectations` interface. This interface provides method for configuring expectations for different HTTP request methods like GET, POST, PUT, etc.

```
ErsatzServer server = new ErsatzServer(cfg -> {
  cfg.expectations(expect -> {
    expect.GET("/api/resources") // Expect a GET request to "/api/resouce"
      .responder(res -> { // Define how to respond to this request
        res.body("Hello, Ersatz!"); // respond with "Hello, Ersatz!"
      });
  });
});
```

## Requirements

- These are conditions or rules that a request must meet to be considered valid.

- They are similar to expectations but serve only to verify requests rather than define how the server should respond.

- Configuring Requirements: you set up requirements using `Requirements` interface. This interface provides method for configuring requirements based on request method and path.

```
ErsatzServer server = new ErsatzServer(cfg -> {
  cfg.requirements(reqs -> {
    reqs.GET("/api/resources") // Specify the request method and path
      .requireHeader("Authorization") // Ensure that requests must contain the Authorization header
  })
})
```
