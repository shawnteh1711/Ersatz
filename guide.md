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

## Request Decoders

- Request decoders are used to convert the bytes of incoming request content into a specified object type.
- This conversion allows the server to match the incoming request content against the expected content defined in the server's expectations.
- They are implemented as a `BiFunction<byte[], DecodingContext, Object>`, where `byte[]` is the request content and the `Object` is the result of transforming the content. The `Decoding Context` is used to provide additional information about the request being decoded (e.g. content-length, content type, character-encoding), along with a reference to the decoder chain.
- Decoders are defined at various levels with the same method signature:

```
ServerConfig decoder(String contentType, BiFunction<byte[], DecodingContext, Object> decoder)
```

- Example:

```
// Define a request decoder for JSON content
def jsonDecoder = { content, context ->
  // Convert the request content bytes to a Groovy data structure
  new JsonSlurper().parse(content ?: '{}'.bytes)
}

// Create an instance of ErsatzServer
def server = new ErsatzServer { cfg ->
  // Register the JSON decoder for JSON content using ServerConfig
  cfg.decoder(ContentType.APPLICATION_JSON, jsonDecoder)
}
```

## Provided Decoders

- The provided decoder in the `io.github.cjstehno.ersatz.encdec.Decoders` class offer convenient way to handle common senarios when decoding request content in the server.

1. Pass-through

- This decoder simply passes the content byte array through as an array of bytes.
- It's useful when you don't need to decode the request content and want to handle it as raw bytes
- Example: `cfg.decoder(ContentType.APPLICATION_JSON, Decoder.passThrough()`)

2. Strings

- convert the request content bytes into a String object, with optional Charset specification
- `cfg.decoder(ContentType.TEXT_PLAIN, Decoder.strings(StandardCharsets.UTF_8))`

3. URL-Encoded

- convert request content butes in a URL-encoded format into a map of name/value pairs
- It's useful when dealing with form submissions where data is sent in the URL-encoded format
- `cfg.decoder(ContentType.APPLICATION_FORM_URLENCODED, Decoders.urlEncoded())`

4. Multipart

- This decoder converts request content bytes into a `MultipartRequestContent` object populated with the multipart request content
- It's useful when handling multipart form data uploads
- `cfg.decoder(ContentType.MULTIPART_FORM_DATA, Decoders.multipart())`

## JSON Decoders

- JSON decoders are configured using the `decoder` method on a `ServerConfig` instance. This method takes the content type of the expected request content (in this case, `ContentType.APPLICATION_JSON`) and a function that performs the decoding.

1. Groovy Decoder

- Groovy `JsonSlurper` takes the content and context as parameters to parse the content into a Groovy object. If the content is null, it defaults to an empty JSON object(`{}`). This decoder is suitable for use in Groovy

```
decoder(ContentType.APPLICATION_JSON) { content, context ->
  new JsonSlurper().parse(content ?: '{}'.bytes)
}
```

2. Jackson Decoder

- Jackson `ObjectMapper`, a popular Java library for JSON processing. The decoder function takes the content and context as parameters. It uses the `ObjectMapper` to deserialize the JSON content into a `Map` object. This decoder is suitable for use in Java applications.

```
decoder(ContentType.APPLICATION_JSON, (content, context) -> {
  return new ObjectMapper().readValue(content, Map.class);
});
```

## Response Encoders

- to convert response configuration data types into the outbound request content string.
- They are implemented as a `Function<object, byte[]`> with the input `Object` being the configuration object being converted, and the `byte[]` is the return type.

- The various configuration levels have the same method signature:

```
ServerConfig encoder(String contentType, class objectType, Function<Object, byte[]> encoder)
```

- The `contentType` is the response content type to be encoded and the `objectType` is the type of configuration object to be encoded - this allows for the same content-type to have different encoders for different configuration object types

- Example of default "text" encoder

```
static function<Object, byte[]> text(final Charset charset) {
  return obj -> (obj != null ? obj.toString() : "").getBytes(charset);
}
```

- To use this encoder, you can configure it in a similar manner in both the server configuration block or in the response configuration block itself, which is shown below:

```
server.expectations(expect -> {
  expect.POST('/submit', req -> {
    req.responder(res -> {
      res.encoder(TEXT_PLAIN, String.class, Encoders.text(UTF_8));
      req.body("This is a string response!", TEXT_PLAIN);
    });
  });
});
```

## Provided Encoders

- `io.github.cjestehno.ersatz.encdec.Encoders` provides a handful of commonly used encoders

1. Text

- convert the object into a String of text
- It has an optional character set specification

```
serverConfig.encoder(ContentType.TEXT_PLAIN, String.class, Encoders.text(StandardCharsets.UTF_8));
```

2. Content

- loads the content at the specified destination and returns it as a byte array
- The content destination may be specified as an InputStream, String, Path, File, URI or URL.

```
serverConfig.encoder(ContentType.IMAGE_PNG, byte[].class, Encoders.content("path/to/image.png"));
```

3. Binary Base64

- encodes a byte array, InputStream, or other object with a "getBytes()" method into a base-64 string

```
serverConfig.encoder(ContentType.APPLICATION_OCTET_STREAM, byte[].class, Encoders.binaryBase64());
```

4. Multipart

- encodes a MultipartResponseContent object to its multipart string representation

```
serverConfig.encoder(ContentType.MULTIPART_FORM_DATA, MultipartResponseContent.class, Encoders.multipart());
```

## JSON Encoders

1. Groovy JSON Encoder

- serialize an object into a JSON string using Groovy's JsonOutput

```
public static final Function<Object, byte[]> json = obj -> {
  (obj != null ? toJson(obj) : "{}").getbytes(UTF_8);
}

encoder(Contenttype.APPLICATION_JSON, MyType.class, obj ->
  JsonOutput.toJson(obj).getBytes(UTF_8)
);
```

2. Jackson JSON Encoder

- serializes an object into a JSON byte array using the Jackson ObjectMapper

```
encoder(ContentType.APPLICATION_JSON, MyType.class, obj -> {
  return new ObjectMapper().writeValueAsBytes(obj);
});
```

## Request Requirements

- Global request requirements allow for common configuration of expectation amcthing requirements in the `ServerConfig` rather than in the individual expectations.

- Consider the case where every request to a server must have some security token configured as a header (say "Security-token") and that all requests must use HTTPS.

```
var server = new ErsatzServer(cfg -> {
  cfg.requirements(require -> {
    require.that(ANY, anyPath(), and -> {
      and.secure();
      and.header("Security-Token", expectedToken);
    });
  });
});
```

## Request Expectations

- The expectation definition methods take four common forms:

1. Using a string path

- This is the simplest form where you specify the request path as a tring

```
server.expectations(expect -> {
  expect.GET("/api/users").responder(res -> {
    res.body("{\"name\": \"John\"}", ContentType.APPLICATION_JSON);
  });
});
```

2. String path with a Consumer<Request>

- This allows for additional customization using a consumer function

```
server.expectations(expect -> {
  expect.GET("/api/users", req -> {
    req.header("Authorization", "Bearer token123");
  }).responder(res -> {
    res.body("{\"name\": \"John\"}", ContentType.APPLICATION_JSON);
  });
});
```

3. String path with a Groovy Closure

- Similar to second method but uses a Groovy Closure for customization

```
server.expectations(expect -> {
  expect.GET("/api/users") {
    header("Authorization", "Bearer roken123");
  }.responser {
    body("{\"name\": \"John\"}", ContentType.APPLICATION_JSON);
  });
});
```

4. Using a Hamcrest Matcher<String> for matching the path

- This allows for flexile path matching based on various request properties

```
import static org.hamcrest.Matchers.*;

server.expectations(expect -> {
  expect.GET(startsWith("/api")) { req ->
    req.header("Authorization", "Bearer token123");
  }.responder(res -> {
    res.body("{\"name\": \"John\"}", ContentType.APPLICATION_JSON);
  });
});
```

- ANY Request Method Matcher: support for an ANY request method matcher configuration

```
server.expectations(expect -> {
  expect.ANY("/api/**").responder(res -> {
    res.status(HttpStatus.NOT_FOUND);
  });
});
```

- Configuration interfaces support three main approaches to configuration

1. Chained builder approach

```
HEAD('/foo')
  .query('a', '42)
  .cookie('stamp', '1234')
  .respond().header('ok', 'true')
```

2. Groovy DSL approach

```
HEAD('/foo')
  query 'a', '42'
  cookie 'stamp', '1234'
  responder {
    header 'ok', 'true'
  }
```

3. Java-based approach using the `Consumer<?>`

```
HEAD('/foo', req -> {
  req.query("a", "42")
  req.cookie("stamp", "1234")
  req.responder( res -> {
    res.header("ok", "true")
  })
})
```

- Request expectations mau be configured to respond differently based on how many times a request is matched.

```
GET('/something') {
  responder {
    content 'Hello'
  }
  responder {
    content "Goodbye'
  }
  /* Addding the called configurations ad the extra safety of ensuring that if the request is called more than our expected two times, the verification will fail
  */
  called 2
}
```

## Request Methods

1. HEAD

- A `HEAD` request is used to retrieve the headers for a URL, basically a `GET` request without any reponse body.

```
ersatzServer.expectations {
  HEAD('/something').responds().header('X-Alpha', 'Interesting-data').code(200)
}
```

2. GET

- The `GET` request is a common HTTP request, and what browsers do by default. It has no request body, but it does have response content. You mock `GET` requests using the `get()` mehtods as follows:

```
ersatzServer.expectations {
  GET('/something').responds().body('this is INTERESTING!', 'text/plain').code(200)
}
```

3. OPTIONS

- `OPTIONS` HTTP request method is similar to an `HEAD`request, having no request or response body.
- The primary response value is an `OPTIONS` request is the content of the `Allow` response header, which will contain a comma-seperated list of the request methods supported by the server
- In order to mock out an `OPTIONS` request, you will want to respond with a provided `Allow` header. This may be done using the `Response.allows(HttpMethod...)` method in the responder

```
ersatzServer.expectations {
  OPTIONS('/options').responds().allows(GET, POST).code(200)
  OPTIONS('/*').responds().allows(DELETE, GET, OPTIONS).code(200)
}
```

4. POST

- The `POST` request is often used to send browser form data to a backend server. It can have both request and response content

```
ersatzServer.expectations{
  POST('/form') {
    body([first:'Shawn', last:'Teh'], APPLICATION_URLENCODED)
    responder {
      body('{status: "saved"}', APPLICATION_JSON)
    }
  }
}
```

- In a RESTful interface,the `POST` method is generally ised to "create" new resources

5. PUT

- A `PUT` request is similar to `POST` except that while there is request content, there is no response body content.

```
ersatzServer.expectations {
  PUT('/form') {
    query('id', '1234')
    body([middle: 'Q'], APPLICATION_URLENCODED)
    responder {
      code(200)
    }
  }
}
```

- In a RESTful interface, a `PUT` request is most often used as an "update" operation.

6. DELETE

- A `DELETE` request has not request or response content

```
ersatzServer.expectations {
  DELETE('/user').query('id', '1234').responds().code(200)
}
```

- In a RESTful interface, a `DELETE` request may be used as a "delete" operation.

7. PATCH

- `PATCH` request method creates a request that can have a body content; however, the response will have no content

```
ersatzServer.expectations {
  PATCH('/user') {
    query('id', '1234')
    body('{"middle": "Q"}', APPLICATION_JSON)
    responder {
      code(200)
    }
  }
}
```

- In a RESTful interface, a `PATCH` request may be used as a "modify" operation for an existing resource.

8. ANY

- The `ANY` request method creates a request expectation that can match any HTTP method

```
server.expectations(expect -> {
  expect.ANY("/something", req -> {
    req.secure();
    req.called(1);
    req.responder(res -> res.body(responseText, TEXT_PLAIN));
  });
});
```

9. Generic Method

- In versinon 3.2 a generic set of request expectation methods were added to allow the definition of request expectations based on variable HTTP request methods:

* `request(final HttpMethod method, final String path`)

```
server.expectations(expect -> {
  expect.request(HttpMethod.GET, "/api/users");
});
```

- `request(final HttpMethod method, final Matcher<String> matcher)`

```
import static org.hamcrest.Matchers.*;

server.expectations(expect -> {
  expect.request(HttpMethod.POST, matchesPattern("/api/users/\\d+"));
});
```

- `request(final HttpMethod method, final String path, Consumer<Request>consumer)`

```
server.expectations(expect -> {
  expect.request(HttpMethod.PUT, "/api/users/123", req -> {
    req.header("Authorization", "Bearer token123");
    req.body("{\"name\": \"John\"}", ContentType.APPLICATION_JSON)
  })
})
```

- `request(final HttpMethod method, final Matcher<String> matcher, final Consumer<Request> consumer)`

```
import static org.hamcrest.Matchers.*;

server.expectations(expect -> {
  expect.request(HttpMethod.DELETE, matchesPattern("/api/users/\\d+"). req -> {
    req.query("id", "1234");
    req.responder(res -> {
      res.code(200);
    });
  });
});
```

- `request(final HttpMethod method, final PathMatcher pathMatcher)`

```
import static io.github.artsok.Repeatedly.*;

server.expectations(expect -> {
  eexpect.request(HttpMethod.PATCH, any());
});
```

- `request(final HttpMethod method, final PathMatcher pathMatcher, Consumer<Request> consumer)`

```
import static io.github.artsok.Repeatedly.*;

server.expectations(expect -> {
  expect.request(HttpMethod.PATCH, any(), req -> {
    req.body("{}", ContentType.APPLICATION_JSON);
    req.responder(res -> {
      res.code(204);
    });
  });
});
```

10. TRACE

- THE `TRACE` method is generally meant for debugging and diagnostics. The request will have no request content; however, if the request is valid, the response will contain the entire request message in the entity-body, with a Content-Type of `message/http`.

- Making a `TRACE` request to Ersatz looks like the following:

```
ersatzServer.start()

URL url = new URL("${ersatzServer.httpUrl}/info?data=foo+bar")

HttpURLConnection connection = url.openConnection() as HttpURLConnection
connection.requestMethod = 'TRACE'

assert connection.contentType == MESSAGE_HTTP.value
assert connection.responseCode == 200

assert connection.inputStream.text.readLines()*.trim() == """TRACE /info?data=foo+barHTTP/1.1
    Accept: text/html, image/gif, image/jpeg, *; q=.2, */*; q=.2
    Connection: keep-alive
    User-Agent: Java/1.9.0.1_121
    Host: localhost:${ersatzServer.httpPort}
""".readLines()*.trim()
```

- The explicit `start()` call is required since there are no expectations specified (auto-start won't fire). The `HtpUrlConnection` is used to make the request, and it can be seen that the response content is the same as the original request content.

# Verification

- A timeout parameter is available on the `verify` method so that a failed verification can fail-out in a timely manner, while still waiting for messages that are not coming

```
// Verifying expectations with a timeout of 5 seconds
boolean verificationResult = server.verify(5000, TimeUnit.MILLISECONDS);

if (verificationResult) {
  System.out.println("All expected requests were received within the timeout period.");
} else {
  System.out.println("Some expected requests were not received within the timeout period or not at all.")
}
```

# Hamcrest Matchers

- allow for a morerich and expressive matching configuration/

```
server.expectations{
  GET(startsWith('/foo)) {
    called greaterThanOrRqualto(2)
    query 'user-key', notNullVAlue()
    responder {
      body 'ok', TEXT_PLAIN
    }  
  }
}
```

# Specialized Matchers

- found in the `io.github.cjstehno.ersatz.match` package
- Examples
  - BodyMatcher
  - BodyParamMatcher
  - PathMatcher
  - QueryParamMatcher
  - HeaderMatcher
  - RequestCookiesMatcher
  - PredicateMatcher (predicateis a condition to test whether condition is true or false. Example: isEven(), isPrime, startsWith(prefix, str))

# Matching Cookies
- There are four methods for matching cookies associated with a requst

1. By Name and Matcher
- The `cookie(String name, Matcher<Cookie>matcher)` configures matcher for the cookie with the given name

```
server.expectations{
  GET('/somewhere') {
    cookie 'user-key', CookieMatcher.cookieMatcher {
      value startsWith('key-')
      domain 'mydomain.com'
    }
    responds().code(200)
  }
}
```

2. By Name and Value
- method where cookie value must be qual to the specified value
```
server.expectations {
  GET('/somewhere') {
    cookie 'user-key', CookieMatcher.cookieMatcher {
      value equalTo('key-2345HJKSDGF86)
    }
    responds().code(200)
  }
}
```

3. Multiple Cookies
- the cookies (`Map<String, Object>`) provides a mean of configuration multiple cookie matchers
```
server.expectations {
  GET('/something') {
    cookies([
      'user-key': cookieMatcher {
        value startsWith('key-')
      },
      'appid': 'user-manager',
      'timeout': nullValue()
    ])
    responds().code(200)
  }
}
```


Overall Matcher
- the `the cookes(Matcher<Map<String, Cookie>)` is used to specify a Matcher for the map of cookie names to `io.github.cjstehno.ersatz.cfg.Cookie` objects
or the `io.github.cjstehno.ersatz.match.NoCookiesMatcher` to match case where no cookies should be defined n the requests

```
server.expectations{
  GET('/something'){
    cookies NoCookiesMatcher.noCookies()
    responds().code(200)
  }
}
```

# Multipart Request Content
- Ersatz server supports multipart file upload request (`multipart/form-data` content-type) using Apache File Upload libary libary on the "server" side.

```
ersatz.expectations {
  POST('/upload') {
    decoder MULTIPART_MIXED, Decoders.multipart
    decoder IMAGE_PNG, Decpders.passthrough
    body multipart {
      part 'something', 'interesting'
      part 'infoFile', 'info.txt', TEXT_PLAIN, infoText
      part 'imageFile', 'image.png', IMAGE_PNG, imageBytes
    }, MULTIPART_MIXED
    responder {
      body 'ok
    }
  }
}
```

# Response Building
- `responds(...), responder(...) and forward(...)` method allow for customization of the response to the request.
- Basic response properties such as headers, status code, and content body are available, as well as more afvanced configuration options, described below:

1. Request/Response Compression
- Ersatz support GZip compression seamlessly as long as the `Accept-Encoding` header is specified as gzip. If the response is compressed, a Content-Encoding header will be added to the response with the appropriate compression type as value.

```
ersatz.expectations {
  GET('/api/data') {
    requestHeader 'Accept-Encoding', 'gzip'
    responds {
      status 200
      header 'Content-Type', 'application/json'
      header 'Content-Encoding', 'gzip'
      body gzipCompress('{"name": "John", "age": "30"})
    }
  }
}
```

2. Chunked Response
- a response may be configured as a "chunked" response, wherein the response data is dent to clinet in small bits along with additional response header, the `Transfer-encoding: chunked`.
- for testing purpose, a fixed or randomized range of time of delay may be configured so that the chunks may be sent slowly, to more accurately simulate a real environment.

```
ersatzServer.expectations {
  GET('/chunky').responder {
    body 'This is chunked content' TEXT_PLAIN
    chunked {
      chunks 3
      delay 100..500
    }
  }
}
```

# Multipart Response Content
- the response content will have standard `multipart/form-data` content type and format.
- the response content parts are provided using an instance of the `MultipartResponseContent` class along with the `Encoders.multipart` multipart response content encoder
- an example multipart response with a field and an image file would be like:


```
ersatz.expectations {
  GET('/data') {
    responder {
      encoder ContentType.MULTIPART_MIXED, MultipartResponseContent, Encoders.multipart
      body(multipart {
        encoder TEXT_PLAIN, CharSequence { o -> (o as String).bytes}
        encoder IMAGE_JPG, File, {o -> ((File)o).bytes }

        field 'comments', 'This is a cool image.'
        part 'image', 'test-image.jpg', IMAGE_JPG, new File('/test-image.jpg'), 'base64'
      })
    }
  }
}
```


- The resulting response body would look like following as a String

```
--WyAJDTEVlYgGjdI13o
Content-Disposition: form-data; name="comments"
Content-Type: text/plain

This is a cool image.
--WyAJDTEVlYgGjdI13o
Content-Disposition: form-data; name="image"; filename="test-image.jpg"
Content-Transfer-Encoding: base64
Content-Type: image/jpeg

... more content follows ...
```

# Request Forwarding
- the `forward(String)` response configuration method causes the incoming reuqest to be forwarded to another server.
- the `String` parameter is the scheme, host, and port of the target server.

```
ersatz.expectations(expect -> {
  expect.GET("/api/widgets/list", req -> {
    req.called()
    req.query("partial", "true")
    req.forward("http://somehost:1234");
  });
});
```

- This will expect that a GET request `/api/widgets/list?partial==true` will be called once and that its response will be the response from making the same request against `http://somehost:1234/api/widgets/list?partial=true`.

# Web socket
- websocket support is provided in the "expectations" configuration. You can expect that a websocket is connected-to, and that it-receives a specified message. You can also "react" to the connection or inbound message by sending a message back to the client.

```
def server = ersatz.expectations(expect -> {
  expect.webSocket("/game", ws -> {
    ws.receives(pingBytes).reaction(pongBytes, BINARY)
  })
})

server.verify(WaitFor.FOREVER)'
```

- the server expects that a websocket connection occurs for "/game", when the connection occurs, the server will exwhen it receibvpect to receive a message of `pingBytes`. When it receives the message, it will respond with a message `pongBytes` in `BINARY` format

- verification timeouts: sometimes when running asynchronous network tests, you can run into issues on different environments. The `verify` method accept a `WaitFor` parameter which allows you to configure a wait time. One useful value here is `FOREVER` which causes the test verification to wait for the expected condition

