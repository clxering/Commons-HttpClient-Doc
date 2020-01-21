# Chapter 1. Fundamentals

## 1.1. HTTP messages

### 1.1.1. Structure

A HTTP message consists of a header and an optional body. The message header of an HTTP request consists of a request line and a collection of header fields. The message header of an HTTP response consists of a status line and a collection of header fields. All HTTP messages must include the protocol version. Some HTTP messages can optionally enclose a content body.

HttpCore defines the HTTP message object model to follow this definition closely, and provides extensive support for serialization (formatting) and deserialization (parsing) of HTTP message elements.

### 1.1.2. Basic operations

#### 1.1.2.1. HTTP request message

HTTP request is a message sent from the client to the server. The first line of that message includes the method to apply to the resource, the identifier of the resource, and the protocol version in use.

```
HttpRequest request = new BasicHttpRequest("GET", "/",
    HttpVersion.HTTP_1_1);

System.out.println(request.getRequestLine().getMethod());
System.out.println(request.getRequestLine().getUri());
System.out.println(request.getProtocolVersion());
System.out.println(request.getRequestLine().toString());
```

stdout >

```
GET
/
HTTP/1.1
GET / HTTP/1.1
```

#### 1.1.2.2. HTTP response message

HTTP response is a message sent by the server back to the client after having received and interpreted a request message. The first line of that message consists of the protocol version followed by a numeric status code and its associated textual phrase.

```
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK, "OK");

System.out.println(response.getProtocolVersion());
System.out.println(response.getStatusLine().getStatusCode());
System.out.println(response.getStatusLine().getReasonPhrase());
System.out.println(response.getStatusLine().toString());
```

stdout >

```
HTTP/1.1
200
OK
HTTP/1.1 200 OK
```

#### 1.1.2.3. HTTP message common properties and methods

An HTTP message can contain a number of headers describing properties of the message such as the content length, content type, and so on. HttpCore provides methods to retrieve, add, remove, and enumerate such headers.

```
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie",
    "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie",
    "c2=b; path=\"/\", c3=c; domain=\"localhost\"");
Header h1 = response.getFirstHeader("Set-Cookie");
System.out.println(h1);
Header h2 = response.getLastHeader("Set-Cookie");
System.out.println(h2);
Header[] hs = response.getHeaders("Set-Cookie");
System.out.println(hs.length);
```

stdout >

```
Set-Cookie: c1=a; path=/; domain=localhost
Set-Cookie: c2=b; path="/", c3=c; domain="localhost"
2
```

There is an efficient way to obtain all headers of a given type using the **_HeaderIterator_** interface.

```
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie",
    "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie",
    "c2=b; path=\"/\", c3=c; domain=\"localhost\"");

HeaderIterator it = response.headerIterator("Set-Cookie");

while (it.hasNext()) {
    System.out.println(it.next());
}
```

stdout >

```
Set-Cookie: c1=a; path=/; domain=localhost
Set-Cookie: c2=b; path="/", c3=c; domain="localhost"
```

It also provides convenience methods to parse HTTP messages into individual header elements.

```
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1,
    HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie",
    "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie",
    "c2=b; path=\"/\", c3=c; domain=\"localhost\"");

HeaderElementIterator it = new BasicHeaderElementIterator(
        response.headerIterator("Set-Cookie"));

while (it.hasNext()) {
    HeaderElement elem = it.nextElement();
    System.out.println(elem.getName() + " = " + elem.getValue());
    NameValuePair[] params = elem.getParameters();
    for (int i = 0; i < params.length; i++) {
        System.out.println(" " + params[i]);
    }
}
```

stdout >

```
c1 = a
 path=/
 domain=localhost
c2 = b
 path=/
c3 = c
 domain=localhost
```

HTTP headers are tokenized into individual header elements only on demand. HTTP headers received over an HTTP connection are stored internally as an array of characters and parsed lazily only when you access their properties.

### 1.1.3. HTTP entity

HTTP messages can carry a content entity associated with the request or response. Entities can be found in some requests and in some responses, as they are optional. Requests that use entities are referred to as entity-enclosing requests. The HTTP specification defines two entity-enclosing methods: POST and PUT. Responses are usually expected to enclose a content entity. There are exceptions to this rule such as responses to HEAD method and 204 No Content, 304 Not Modified, 205 Reset Content responses.

HttpCore distinguishes three kinds of entities, depending on where their content originates:

- **streamed**: The content is received from a stream, or generated on the fly. In particular, this category includes entities being received from a connection. Streamed entities are generally not repeatable.

- **self-contained**: The content is in memory or obtained by means that are independent from a connection or other entity. Self-contained entities are generally repeatable.

- **wrapping**: The content is obtained from another entity.

#### 1.1.3.1. Repeatable entities

An entity can be repeatable, meaning its content can be read more than once. This is only possible with self-contained entities (like **_ByteArrayEntity_** or **_StringEntity_**).

#### 1.1.3.2. Using HTTP entities

Since an entity can represent both binary and character content, it has support for character encodings (to support the latter, i.e. character content).

The entity is created when executing a request with enclosed content or when the request was successful and the response body is used to send the result back to the client.

To read the content from the entity, one can either retrieve the input stream via the HttpEntity#getContent() method, which returns an **_java.io.InputStream_**, or one can supply an output stream to the **_HttpEntity#writeTo(OutputStream)_** method, which will return once all content has been written to the given stream. Please note that some non-streaming (self-contained) entities may be unable to represent their content as a java.io.InputStream efficiently. It is legal for such entities to implement **_HttpEntity#writeTo(OutputStream)_** method only and to throw UnsupportedOperationException from **_HttpEntity#getContent()_** method.

The **_EntityUtils_** class exposes several static methods to simplify extracting the content or information from an entity. Instead of reading the **_java.io.InputStream_** directly, one can retrieve the complete content body in a string or byte array by using the methods from this class.

When the entity has been received with an incoming message, the methods **_HttpEntity#getContentType()_** and **_HttpEntity#getContentLength()_** methods can be used for reading the common metadata such as **_Content-Type_** and **_Content-Length_** headers (if they are available). Since the **_Content-Type_** header can contain a character encoding for text mime-types like text/plain or text/html, the HttpEntity#getContentEncoding() method is used to read this information. If the headers aren't available, a length of -1 will be returned, and NULL for the content type. If the Content-Type header is available, a Header object will be returned.

When creating an entity for a outgoing message, this meta data has to be supplied by the creator of the entity.

```
StringEntity myEntity = new StringEntity("important message",
    Consts.UTF_8);

System.out.println(myEntity.getContentType());
System.out.println(myEntity.getContentLength());
System.out.println(EntityUtils.toString(myEntity));
System.out.println(EntityUtils.toByteArray(myEntity).length);
```

stdout >

```
Content-Type: text/plain; charset=UTF-8
17
important message
17
```

#### 1.1.3.3. Ensuring release of system resources

In order to ensure proper release of system resources one must close the content stream associated with the entity.

```
HttpResponse response;
HttpEntity entity = response.getEntity();
if (entity != null) {
    InputStream instream = entity.getContent();
    try {
        // do something useful
    } finally {
        instream.close();
    }
}
```

When working with streaming entities, one can use the **_EntityUtils#consume(HttpEntity)_** method to ensure that the entity content has been fully consumed and the underlying stream has been closed.

### 1.1.4. Creating entities

There are a few ways to create entities. HttpCore provides the following implementations:

- BasicHttpEntity
- ByteArrayEntity
- StringEntity
- InputStreamEntity
- FileEntity
- EntityTemplate
- HttpEntityWrapper
- BufferedHttpEntity

#### 1.1.4.1. BasicHttpEntity

Exactly as the name implies, this basic entity represents an underlying stream. In general, use this class for entities received from HTTP messages.

This entity has an empty constructor. After construction, it represents no content, and has a negative content length.

One needs to set the content stream, and optionally the length. This can be done with the **_BasicHttpEntity#setContent(InputStream)_** and **_BasicHttpEntity#setContentLength(long)_** methods respectively.

```
BasicHttpEntity myEntity = new BasicHttpEntity();
myEntity.setContent(someInputStream);
myEntity.setContentLength(340); // sets the length to 340
```

#### 1.1.4.2. ByteArrayEntity

**_ByteArrayEntity_** is a self-contained, repeatable entity that obtains its content from a given byte array. Supply the byte array to the constructor.

ByteArrayEntity myEntity = new ByteArrayEntity(new byte[] {1,2,3},
ContentType.APPLICATION_OCTET_STREAM);

1.1.4.3. StringEntity

**_StringEntity_** is a self-contained, repeatable entity that obtains its content from a java.lang.String object. It has three constructors, one simply constructs with a given java.lang.String object; the second also takes a character encoding for the data in the string; the third allows the mime type to be specified.

```
StringBuilder sb = new StringBuilder();
Map<String, String> env = System.getenv();
for (Map.Entry<String, String> envEntry : env.entrySet()) {
    sb.append(envEntry.getKey())
            .append(": ").append(envEntry.getValue())
            .append("\r\n");
}

// construct without a character encoding (defaults to ISO-8859-1)
HttpEntity myEntity1 = new StringEntity(sb.toString());

// alternatively construct with an encoding (mime type defaults to "text/plain")
HttpEntity myEntity2 = new StringEntity(sb.toString(), Consts.UTF_8);

// alternatively construct with an encoding and a mime type
HttpEntity myEntity3 = new StringEntity(sb.toString(),
        ContentType.create("text/plain", Consts.UTF_8));
```

#### 1.1.4.4. InputStreamEntity

**_InputStreamEntity_** is a streamed, non-repeatable entity that obtains its content from an input stream. Construct it by supplying the input stream and the content length. Use the content length to limit the amount of data read from the **_java.io.InputStream_**. If the length matches the content length available on the input stream, then all data will be sent. Alternatively, a negative content length will read all data from the input stream, which is the same as supplying the exact content length, so use the length to limit the amount of data to read.

```
InputStream instream = getSomeInputStream();
InputStreamEntity myEntity = new InputStreamEntity(instream, 16);
```

#### 1.1.4.5. FileEntity

FileEntity is a self-contained, repeatable entity that obtains its content from a file. Use this mostly to stream large files of different types, where you need to supply the content type of the file, for instance, sending a zip file would require the content type **_application/zip_**, for XML **_application/xml_**.

```
HttpEntity entity = new FileEntity(staticFile,
        ContentType.create("application/java-archive"));
```

#### 1.1.4.6. HttpEntityWrapper

This is the base class for creating wrapped entities. The wrapping entity holds a reference to a wrapped entity and delegates all calls to it. Implementations of wrapping entities can derive from this class and need to override only those methods that should not be delegated to the wrapped entity.

#### 1.1.4.7. BufferedHttpEntity

**_BufferedHttpEntity_** is a subclass of **_HttpEntityWrapper_**. Construct it by supplying another entity. It reads the content from the supplied entity, and buffers it in memory.

This makes it possible to make a repeatable entity, from a non-repeatable entity. If the supplied entity is already repeatable, it simply passes calls through to the underlying entity.

```
myNonRepeatableEntity.setContent(someInputStream);
BufferedHttpEntity myBufferedEntity = new BufferedHttpEntity(
  myNonRepeatableEntity);
```

## 1.2. HTTP protocol processors

HTTP protocol interceptor is a routine that implements a specific aspect of the HTTP protocol. Usually protocol interceptors are expected to act upon one specific header or a group of related headers of the incoming message or populate the outgoing message with one specific header or a group of related headers. Protocol interceptors can also manipulate content entities enclosed with messages; transparent content compression / decompression being a good example. Usually this is accomplished by using the 'Decorator' pattern where a wrapper entity class is used to decorate the original entity. Several protocol interceptors can be combined to form one logical unit.

HTTP protocol processor is a collection of protocol interceptors that implements the 'Chain of Responsibility' pattern, where each individual protocol interceptor is expected to work on the particular aspect of the HTTP protocol it is responsible for.

Usually the order in which interceptors are executed should not matter as long as they do not depend on a particular state of the execution context. If protocol interceptors have interdependencies and therefore must be executed in a particular order, they should be added to the protocol processor in the same sequence as their expected execution order.

Protocol interceptors must be implemented as thread-safe. Similarly to servlets, protocol interceptors should not use instance variables unless access to those variables is synchronized.

### 1.2.1. Standard protocol interceptors

HttpCore comes with a number of most essential protocol interceptors for client and server HTTP processing.

#### 1.2.1.1. RequestContent

RequestContent is the most important interceptor for outgoing requests. It is responsible for delimiting content length by adding the Content-Length or Transfer-Content headers based on the properties of the enclosed entity and the protocol version. This interceptor is required for correct functioning of client side protocol processors.

#### 1.2.1.2. ResponseContent

ResponseContent is the most important interceptor for outgoing responses. It is responsible for delimiting content length by adding Content-Length or Transfer-Content headers based on the properties of the enclosed entity and the protocol version. This interceptor is required for correct functioning of server side protocol processors.

#### 1.2.1.3. RequestConnControl

RequestConnControl is responsible for adding the Connection header to the outgoing requests, which is essential for managing persistence of HTTP/1.0 connections. This interceptor is recommended for client side protocol processors.

#### 1.2.1.4. ResponseConnControl

ResponseConnControl is responsible for adding the Connection header to the outgoing responses, which is essential for managing persistence of HTTP/1.0 connections. This interceptor is recommended for server side protocol processors.

#### 1.2.1.5. RequestDate

RequestDate is responsible for adding the Date header to the outgoing requests. This interceptor is optional for client side protocol processors.

#### 1.2.1.6. ResponseDate

ResponseDate is responsible for adding the Date header to the outgoing responses. This interceptor is recommended for server side protocol processors.

#### 1.2.1.7. RequestExpectContinue

RequestExpectContinue is responsible for enabling the 'expect-continue' handshake by adding the Expect header. This interceptor is recommended for client side protocol processors.

#### 1.2.1.8. RequestTargetHost

RequestTargetHost is responsible for adding the Host header. This interceptor is required for client side protocol processors.

#### 1.2.1.9. RequestUserAgent

RequestUserAgent is responsible for adding the User-Agent header. This interceptor is recommended for client side protocol processors.

#### 1.2.1.10. ResponseServer

ResponseServer is responsible for adding the Server header. This interceptor is recommended for server side protocol processors.

### 1.2.2. Working with protocol processors

Usually HTTP protocol processors are used to pre-process incoming messages prior to executing application specific processing logic and to post-process outgoing messages.

```
HttpProcessor httpproc = HttpProcessorBuilder.create()
        // Required protocol interceptors
        .add(new RequestContent())
        .add(new RequestTargetHost())
        // Recommended protocol interceptors
        .add(new RequestConnControl())
        .add(new RequestUserAgent("MyAgent-HTTP/1.1"))
        // Optional protocol interceptors
        .add(new RequestExpectContinue(true))
        .build();

HttpCoreContext context = HttpCoreContext.create();
HttpRequest request = new BasicHttpRequest("GET", "/");
httpproc.process(request, context);
```

Send the request to the target host and get a response.

```
HttpResponse = <...>
httpproc.process(response, context);
```

Please note the BasicHttpProcessor class does not synchronize access to its internal structures and therefore may not be thread-safe.

## 1.3. HTTP execution context

Originally HTTP has been designed as a stateless, response-request oriented protocol. However, real world applications often need to be able to persist state information through several logically related request-response exchanges. In order to enable applications to maintain a processing state HttpCpre allows HTTP messages to be executed within a particular execution context, referred to as HTTP context. Multiple logically related messages can participate in a logical session if the same context is reused between consecutive requests. HTTP context functions similarly to a java.util.Map<String, Object>. It is simply a collection of logically related named values.

Please nore HttpContext can contain arbitrary objects and therefore may be unsafe to share between multiple threads. Care must be taken to ensure that HttpContext instances can be accessed by one thread at a time.

### 1.3.1. Context sharing

Protocol interceptors can collaborate by sharing information - such as a processing state - through an HTTP execution context. HTTP context is a structure that can be used to map an attribute name to an attribute value. Internally HTTP context implementations are usually backed by a HashMap. The primary purpose of the HTTP context is to facilitate information sharing among various logically related components. HTTP context can be used to store a processing state for one message or several consecutive messages. Multiple logically related messages can participate in a logical session if the same context is reused between consecutive messages.

```
HttpProcessor httpproc = HttpProcessorBuilder.create()
        .add(new HttpRequestInterceptor() {
            public void process(
                    HttpRequest request,
                    HttpContext context) throws HttpException, IOException {
                String id = (String) context.getAttribute("session-id");
                if (id != null) {
                    request.addHeader("Session-ID", id);
                }
            }
        })
        .build();

HttpCoreContext context = HttpCoreContext.create();
HttpRequest request = new BasicHttpRequest("GET", "/");
httpproc.process(request, context);
```