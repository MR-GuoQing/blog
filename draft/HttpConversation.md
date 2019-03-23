# Developer Guide - HttpConversation

The flexible HTTP networking stack of the SAP Mobile Platform SDK for Android. In this document you'll find all the info necessary to start working with the `HttpConversation` and its close relative the `HttpConvAuthFlows` library. The guide gradually unveils the details as it goes from the most straightforward use cases to more complex ones. These two components are often referred to in the abbreviated form: _HttpC_.

It is assumed that you are familiar with Android application development and Java programming in general. If you have experience using the built-in `HttpURLConnection` API then its a plus as HttpC is built using it.

Jump to the section of interest if you already know this guide:

1. [Essentials](#essentials)
    1. [Sending basic GET and POST requests](#basic_send)
    1. [The flow listener](#flow_listener)
    1. [Method chaining](#method_chaining)
    1. [Summary](#sum1)
1. [Philosophy and features](#features)
    1. [The `HttpConversationManager` class](#hcm_class)
    1. [The concept of a _conversation_](#conv_concept)
    1. [Summary](#sum2)
1. [A closer look](#closer_look)
    1. [Streams](#streams)
    1. [Network Security Configuration considerations](#nsc)
    1. [Manager configurators](#manager_configurators)
    1. [Summary](#sum3)
1. [Common authentication/authorization flows](#hcaf)
    1. [Basic Authentication](#basic_auth)
    1. [Mutual SSL/TLS authentication](#x509)
    1. [SAML2](#saml2)
        1. [Overview](#saml2_overview)
        1. [Details of the SAML2 support](#saml2_details)
        1. [SAML2 configuration](#saml2_config)
        1. [Setting up a manager](#saml2_setup)
        1. [Insights and summary](#sum4)
    1. [OAuth2](#oauth2)
        1. [Supported grant types](#oauth2_details)
        1. [Extension points of the standard](#oauth2_ep)
        1. [The `OAuth2ServerSupport` interface](#oauth2_server_support)
        1. [Scope of access and refresh tokens](#oauth2_token_scope)
        1. [Token storages](#token_storages)
        1. [Enabling support for SAPcp](#oauth2_sapcp)
        1. [Authorization Code Grant & HTTP](#oauth2_web)
        1. [Summary](#sum5)
    1. [Two-Factor Authentication](#2fa)
        1. [Overview](#2fa_overview)
        1. [OTP configuration](#2fa_config)
        1. [Communication between **SAP Authenticator** and your app](#2fa_comm)
        1. [Summary](#sum6)
1. [Deep dive](#deep_dive)
    1. [The conversation context](#conv_context)
    1. [Writing custom filters](#custom_filters)
    1. [Executors](#executors)
    1. [OAuth2 token management](#oauth2_token_management)
    1. [Web strategies](#webstrategies)
    1. [Cookie management](#cookies)

## <a name="essentials"></a> Essentials

### <a name="basic_send"></a> Sending basic GET and POST requests

Let's begin with the simplest example: firing an HTTP GET request against `http://www.example.com`:

```java
import ...;

public class ExampleActivity extends Activity {
    @Override
    public void onStart() {
        // Create a new instance of a 'HttpConversationManager'. This is the start of everything.
        // Note that it requires an instance of 'android.content.Context' as argument.
        HttpConversationManager manager = new HttpConversationManager(this);

        // Create the URL and the conversation. The returned 'IHttpConversation' object can be
        // used to configure a lot more things, including request parameters, headers and
        // listeners to be called when the request is being sent and the response is being received.
        IHttpConversation conv = manager.create(Uri.parse("http://www.example.com"));

        // Set the response listener that gets called when the response is received. The below
        // example uses Java 8 lambdas but with earlier Java versions you'd be using an anonymous
        // nested class.
        conv.setResponseListener(event -> {
            // Process the response. The 'event' argument contains everything related to the
            // received HTTP response (status code, headers, payload, etc..)
            ...
        });

        // Send the request by simply starting the conversation. Request execution will take place
        // on a new thread.
        conv.start();
    }
}
```

> The above example code is placed in the `onStart()` method of an example activity. Of course, this is just to simplify the code snippet and help you concentrate on the important parts. The rest of the snippets in this guide follow this approach.

To get started, the very first thing to do is to acquire an instance of `com.sap.smp.client.httpc.HttpConversationManager`. In simple cases one can instantiate one for itself and then throw it away at the end of the request. However, the manager may be reused for multiple requests.

The details of the request can be configured via the `com.sap.smp.client.httpc.IHttpConversation` object returned by `create()`. The above example sets the response listener only which is called when the final HTTP response is received.

Initially, the conversation object is dormant and does nothing 'till `start()` is called. When that happens the library creates a new `java.net.HttpURLConnection` (or `javax.net.ssl.HttpsURLConnection`, depending on the URL scheme) in the background, passes it the configured parameters and then uses it to send the request and process the response.

Therefore, it is best to think of each `IHttpConversation` object as an intelligent wrapper around a `HttpURLConnection` extending the latter with additional features. Thus, `HttpConversationManager` is the factory of conversations.

Let's send now a POST request to an endpoint that expects HTML form input with data describing a user:

```java
import ...;

public class ExampleActivity extends Activity {
    @Override
    public void onStart() {
        // Obtaining a manager is the starting point.
        HttpConversationManager manager = new HttpConversationManager(this);

        // Create the conversation.
        IHttpConversation conv = manager.create(Uri.parse("http://www.example.com/userform"));

        // Set the content type and the method.
        conv.setMethod(HttpMethod.POST);
        conv.addHeader("Content-Type", "application/x-www-form-urlencoded; charset=utf-8");

        // Set the request listener using Java 8 lambdas.
        conv.setRequestListener(event -> {
            // Set the payload. Note how the fields are properly encoded as required by the content type.
            String userName = ...;
            String emailAddress = ...;
            String phoneNumber = ...;

            // HttpC is stream-based. The below writer is for convenience to send textual data.
            // It is available for all text-based content-types for which the charset is provided.
            OutputStreamWriter writer = event.getWriter();

            writer.write("user_name=");
            writer.write(Uri.encode(userName));
            writer.write("&");

            writer.write("email_address=");
            writer.write(Uri.encode(emailAddress));
            writer.write("&");

            writer.write("phone_number=");
            writer.write(Uri.encode(phoneNumber));
            return null;
        });

        // Set the response listener using Java 8 lamdas again.
        conv.setResponseListener(event -> {
            // Process the results.
            ...
        });

        // Send the request by simply starting the conversation. Request execution will take place
        // on a new thread.
        conv.start();
    }
}
```

This concludes the basic examples. This should be more than enough to cover the simplest use cases. Let's continue by looking at handling important events during the conversation flow, such as errors.

### <a name="flow_listener"></a> The flow listener

The introductory examples demonstrated how to handle the so-called _happy path_. However, several things might happen with a started conversation, like running into network errors or cancellation of the entire request sending process.

All these events can be handled by implementing the `com.sap.smp.client.httpc.listeners.IConversationFlowListener` interface. Let's extend the HTTP GET example with some basic error handling:

```java
import ...;

public class ExampleActivity extends Activity {
    @Override
    public void onStart() {
        // Create a new instance of a 'HttpConversationManager' again.
        HttpConversationManager manager = new HttpConversationManager(this);

        // Create the URL and the conversation.
        IHttpConversation conv = manager.create(Uri.parse("http://www.example.com"));

        // Set the response listener.
        conv.setResponseListener(event -> {
            // Process the response.
            ...
        });

        // Set the flow listener. This is what gets notified of several other actions.
        conv.setFlowListener(IConversationFlowListener
                                .prepareFor()
                                .communicationError(e -> {
                                    // Handle 'e' which is a 'java.io.IOException'.
                                    ...
                                }).build());

        // Send the request by simply starting the conversation. Request execution will take place
        // on a new thread.
        conv.start();
    }
}
```

> Look at how we made use of the convenience `prepareFor()` static method of the `IConversationFlowListener` interface which allows us to listen for only a selected few events of interest and use Java 8 lambdas for brevity. Of course, we could have implemented the interface in the traditional way.
>
> An alternative solution is to subclass the `com.sap.smp.client.httpc.utils.EmptyFlowListener` class, a no-op implementation of the flow listener, like so:
> ```java
> conv.setFlowListener(new EmptyFlowListener() {
>     @Override
>     void onCommunicationError(IOException e) {
>         // Handle the exception.
>         ...
>     }
> });
> ```
> This can be useful if you're not using Java 8 in your project.

In the above example we set the flow listener on the conversation directly. However, we could have set it on the `HttpConversationManager` instance instead, using the `setDefaultFlowListener()` method. The difference is that if a conversation has no flow listener configured then it will use the one it inherits from the conversation manager.

### <a name="method_chaining"></a> Method chaining

Most of the APIs of HttpC are prepared for method chaining. This allows for setting up an individual HTTP request and sending it in a quite concise form. Let's grab some examples from the previous sections and see how they'd look like with method chaining:

```java
import ...;

public class ExampleActivity extends Activity {
    @Override
    public void onStart() {
        // Create a new instance of a 'HttpConversationManager'.
        HttpConversationManager manager = new HttpConversationManager(this);

        // Define the entire conversation with one long method chain and kick-off the
        // request with 'start()'.
        manager.create(Uri.parse("http://www.example.com/userform"))
            .setMethod(HttpMethod.POST)
            .addHeader("Content-Type", "application/x-www-form-urlencoded; charset=utf-0")
            .setRequestListener(event -> {
                // Set the payload. Note how the fields are properly encoded as required by the content type.
                String userName = ...;
                String emailAddress = ...;
                String phoneNumber = ...;

                // HttpC is stream-based. The below writer is for convenience to send textual data.
                // It is available for all text-based content-types.
                OutputStreamWriter writer = event.getWriter();

                writer.write("user_name=");
                writer.write(Uri.encode(userName));
                writer.write("&");

                writer.write("email_address=");
                writer.write(Uri.encode(emailAddress));
                writer.write("&");

                writer.write("phone_number=");
                writer.write(Uri.encode(phoneNumber));
            }).setResponseListener(event -> {
                // Process the response. The 'event' argument contains everything related to the
                // received HTTP response (status code, headers, payload, etc..)
                ...
            }).setFlowListener(IConversationFlowListener
                                .prepareFor()
                                .communicationError(e -> {
                                    // Handle 'e' which is a 'java.io.IOException'.
                                    ...
                                }).build())
            .start();
    }
}
```

Looks better, doesn't it?

### <a name="sum1"></a> Summary

The following things are to be taken away from this introductory section:

* HttpConversation builds on top of the built-in URL connection APIs. Its main concept is the _conversation manager_ which creates _conversations_.
* The starting point is always a `HttpConversationManager` instance. For now, look at it as a factory using which conversations can be created which effectively is just another name for an object that represents a HTTP request.
* Preparing a request is about configuring the created `IHttpConversation` object with parameters and appropriate listeners. Then, the `start()` method is what starts the request sending process, by default, on a background thread.
* Flow listeners can be used to become notified of other important events in the lifecycle of a conversation, such as when an IO error occurs.
* For the greatest convenience, consider using the Java 8 helpers and the method chaining pattern that the HttpC API offers out of the box.

> If you would like to try out a working sample application sending simple HTTP GET and POST requests then go to `SDK_install/android/samples` directory and open `build.gradle` file with Android Studio. Sample applications will be imported and compiled, `HttpGetPost` is the sample application for sending simple HTTP GET and POST requests. You can debug the application source code and feel free to modify it and experiment.


## <a name="features"></a> Philosophy and features

Let's continue by taking a look at the main philosophy of the library and its features. This section is followed by a series of examples.

### <a name="hcm_class"></a> The `HttpConversationManager` class

In previous examples, the conversation manager has always been instantiated anew and has not been reused. The examples could have been modified to make use of a single `HttpConversationManager` object stored in a property of an activity or in a static variable of some class, for example. As far as the introductory code snippets are concerned, the results would have been the same.

What does a single instance of a `HttpConversationManager` represent then? Apart from being just a factory to create and send HTTP requests with, it also carries a series of user exits that can be utilized to customize the behaviour of and/or hook into the various stages of HTTP request/response processing.

Each of these user exits have corresponding setter or adder methods on this class which are categorized as follows:

* Filters are standardized hooks invoked at predefined points. The two kinds: request and response filters may be used for different purposes. There are methods available to add and remove filters with.
* Observers can be added the same way as filters can but they can only be used for getting notifications about certain events. It is not possible to alter the way how the HTTP request is being sent via observers.
* Listeners are there to hook into the various stages of request sending not covered by filters and observers. There are many kinds of listeners for many purposes. Namely:
    * The flow listener which has already been mentioned and which can be configured on a per conversation basis and not just per conversation manager.
    * Connection configuration listeners which are invoked just before request sending and can be used to further customize the operation from the following aspects:
        * Hostname verification (via `setHostnameVerifierListener()`): The configured listener may return a `javax.net.ssl.HostnameVerifier` that is then plugged into the SSL/TLS handshake mechanism to verify the server host.
        * Proxy configuration (via `setProxyProviderListener()`): This can be used to return a `java.net.Proxy` instance. It is seldom used as by default HttpC uses the proxy settings configured by the system.
    * SSL socket factory listener (via `setSSLSocketFactoryListener()`) which is useful if one wishes to configure the `javax.net.ssl.SSLSocketFactory` the conversation manager should be using to create SSL/TLS connections with. HttpC provides a default listener which should be enough for most use cases.
* Response detectors (configurable via `HttpConversationManager.setDefaultTextBasedResponseDetector()` or `IHttpConversation.setTextBasedResponseDetector()`) are user exits invoked when a response is received. Their main purpose is to analise the received response and produce a `java.nio.charset.Charset` object, if possible. As HttpC is stream-based, when a response is received the payload byte stream is always readable. However, if a response detector can deduce that the payload is textual then instead of having to consume raw bytes one can consume character data directly which will be decoded with the correct charset.

  Overriding response detectors is doable on the manager- or the conversation-level, however it's seldom necessary. HttpC has a built-in detector that can infer the charset of many kinds of HTTP responses. Check the API documentation of `com.sap.smp.client.httpc.ITextBasedResponseDetector` for additional info.
* Redirect whitelist (via `setRedirectWhitelist()`) that may be used to set a `com.sap.smp.client.httpc.RedirectWhitelist` implementation. It is one that can be asked whether a particular redirect may or may not be followed.

The above list is quite long. In fact, it's a rather comprehensive summary of what you can tweak on a `HttpConversationManager` instance. However, in the majority of the cases you'll leave most of these settings untouched. The most important things to concentrate on now are **request and response filters**.

A particular collection of user exits together are called the _configuration_ of a particular `HttpConversationManager` instance.

The most important implication of all this is that when thinking about how to manage manager instances within an application, one actually has to think about the configurations.

What can filters be used for? A lot of things but in the overwhelming majority of the cases the filters of a configuration are used to handle _authentication_ challenges transparently.

"Transparently" here refers to that with a properly configured `HttpConversationManager` one can separate the logic required to handle authentication and authorization with the server from the logic that sends HTTP requests to acquire actual data or perform changes on a server.

This might be shady at this point but reading on will help clear things up.

### <a name="conv_concept"></a> The concept of a _conversation_

The HttpC library introduces a new concept, called the HTTP _conversation_ (hence the name). A conversation might consist of multiple HTTP request-response exchanges with the server 'till the process settles down and produces a final response.

How many requests are sent depends heavily on how a `HttpConversationManager` is configured. But first, let's take a look at the lifecycle of a conversation:

1. A conversation starts by creating an `IHttpConversation` object, setting it up and then starting it using the `start()` method.
1. Before the actual request is executed, the manager calls all the configured _request filters_ in the order they were added. These filters get a chance to take a look into the request that is about to be sent. Some filters might even alter the request by adding request headers or additional parameters.
1. The HTTP request is then sent using the underlying `HttpURLConnection` object. At this point, various configured listeners might be called which might have a say in how the URL connection object is to be parametrized.
1. After the response has been received, each configured _response filter_ is called in order, one after the other. These can perform HTTP response post-processing of any sort. If one of these filters commands the executing conversation manager to restart the request then, just like in case of authentication challenges, the conversation jumps back to step no. 2.
1. At the very end of the conversation, the final response (i.e. the response of the last HTTP request fired) is handed over to the response listener configured on the conversation object. Certain conditions, such as network error or cancellation, might yield that the response listener is never called. In these cases the events will manifest in calls to the configured flow listener.

Distinguishing between a single HTTP request and the entire HTTP conversation is therefore crucial in understanding how the library works.

Let's take a look again at the HTTP GET introductory example but this time let's extend it with some configuration:

```java
import ...;

class HardCodedCredentialFilter implements IRequestFilter {
    private static final String TRIED_ALREADY_FLAG = "HardCodedCredentialsTried";

    @Override
    public Object filter(ISendEvent event, IRequestFilterChain chain) {
        // Get the context to read conversation-scoped information from.
        Map<String, Object> convContext = event.getConversationContext().getStateMap(this.getClass().getName(), false);

        // Check if the hard-coded credentials have been tried already.
        if (!convContext.containsKey(TRIED_ALREADY_FLAG)) {
            // Assemble the BasicAuth header.
            final String user = "john_doe";
            final String password = "p4Ssw0rD1Z3";
            String encodedUserPassword = Base64.encodeToString(
                (user + ":" + password).getBytes(Charset.forName("UTF-8")),
                Base64.NO_WRAP);

            // Set it.
            event.getRequestHeaders().put("Authorization", "Basic " + encodedUserPassword);

            // Set the flag so that the next time we'll not retry the credentials.
            convContext.put(TRIED_ALREADY_FLAG, true);
        }
    }

    @Override
    public Object getDescriptor() {
        return this.getClass();
    }
}

public class ExampleActivity extends Activity {
    private HttpConversationManager manager;

    @Override
    public void onStart() {
        configure();
        use();
    }

    private void configure() {
        // Create a new instance of a 'HttpConversationManager'.
        manager = new HttpConversationManager(this);

        // Configure it by adding our custom filter to it.
        manager.addFilter(new HardCodedCredentialFilter());
    }

    private void use() {
         manager.create(Uri.parse("http://www.example.com"))
                .setResponseListener(event -> {
                    // Process the response.
                    ...
                }).start();
    }
}
```

Note how the act of **configuring** the manager can be detached from actually **using** the manager: the `configure()` method should be called only once after which the `manager` member will point to the properly configured conversation manager. After that, `use()` can be called as many times we want. It will make use of the configured manager.

Observe that in the `use()` method we don't bother with authentication at all. The request is just assembled and the results are processed by the response listener. There's no need for this method to know the credentials or let alone be aware of how the authentication needs to be performed. This is all taken care of by the `configure()` method which applies our `HardCodedCredentialFilter` on the newly instantiated manager.

Also note that the `HardCodedCredentialFilter` prepares for that it might be called multiple times. Checking a flag indicating whether the credentials have been tried already ensures that the hard-coded credentials are sent only once, thereby preventing an infinite loop of retries if authentication fails.

> A lot of things are likely to be rather new at this point for you: what is the state map that the filter uses? Why are we not storing the "already tried" flag simply in a private member variable of the filter? All these questions are going to be answered by later chapters. Please read on to get the full picture.

### <a name="sum2"></a> Summary

Below is what you should remember from this part:

* An `HttpConversationManager` instance carries a configuration that affects how HTTP request and responses are handled.
* The configuration is made up various kinds of user exits: filters, observers, listeners, etc.. Out of these, request and response filters are the most important concepts to concentrate on.
* A conversation is actually a complex flow which invokes the configured user exits and produces a final response. During execution, multiple HTTP requests might get sent, for example, in case authentication challenges need to be responded to.
* The main philosophy therefore is that configuring and using a conversation manager can be separated.

## <a name="closer_look"></a> A closer look

In this section we begin to take a step further and examine the features of HttpC more closely. We'll discuss threading concepts, the stream-based nature of the API and the possible ways of configuring a `HttpConversationManager`.

### <a name="streams"></a> Streams

HttpC uses `java.io` output and input streams to send the request payload and to consume the response payload with, respectively. This allows it to process rather large payloads without causing extensive memory consumption.

Using streams, one can send/receive raw bytes which is quite useful when dealing with binary payloads. However, a character-based stream is much more effective if the payload is textual. One could manually wrap the streams into the corresponding `java.io` readers or writers (depending on the type of the stream) but for this a character encoding is required. The latter is often available in the `Content-Type` header, therefore HttpC provides the corresponding readers and writers for us, if possible.

In practice, when sending requests you have to implement your own `IRequestListener` and associate it with an `IHttpConversation`. The `onRequestBodySending()` method of the listener takes a `com.sap.smp.client.httpc.events.ITransmitEvent` parameter object as argument. Through this object you have access to both the output stream and the writer via getters. If the `Content-Type` of the request defines the character encoding then HttpC will initialize the writer and we can use it to send character data rather easily:

```java
...

IHttpConversation conv = ...;
conv.setMethod(HttpMethod.POST)
    .addHeader("Content-Type", "application/json; charset=utf-8")
    .setRequestListener(event -> {
        // Write some character data to the output. In this case it's a JSON.
        event.getWriter().write("{ \"key\" : \"value\" }");
    });

...
```

Similarly, you can use the same approach when processing the response:

```java
...

IHttpConversation conv = ...;
conv.setResponseListener(event -> {
        // Get the reader and use it to consume the response body.
        Reader r = event.getReader();
        ...
    });

...
```

If you'd like to collect the entire contents of the reader into a `java.lang.String` then use a helper method of the event interface:

```java
...

IHttpConversation conv = ...;
conv.setResponseListener(event -> {
        // Consume the entire character stream and put it into a string.
        String responseBody = IReceiveEvent.Util.getResponseBody(event.getReader());
        ...
    });

...
```

This approach is recommended only if you expect the payload to be relatively small and thus worth keeping it in memory.

The following things are to be kept in mind when using these binary streams and readers/writers:

* Input streams and readers available in the response listeners are connected to the same underlying data source. If you consume some data using one then it will affect the other. For example, if you read a handful of characters using the reader then it has the same effect on the underlying data source as if you read the corresponding raw bytes.

  Normally, when a reader is available one does not use the input stream as it's not needed. If you have a scenario where the input payload has a mixed content then make sure you don't mess with the character encoding. For example, if the response body is encoded in UTF-8 and the next character is represented by a multibyte sequence then reading only the first few bytes of said sequence will screw up the input underneath the reader. Take great care if you opt to use the stream and the reader together.

  As the response body is also available in response filters, the same rules apply.
* Similarly, output streams and writers available in the request listeners are connected to the same underlying data sink. Again it is recommended to use only the writer if one is available. Similar kind of problems may be caused if you intermix the usage of the output stream and the writer: if you write some raw bytes then you must ensure not to mess with the character encoding.
* Neither request nor response listeners require you to close the streams. In fact, trying to close these streams actually yields an exception. It is the responsibility of the library to take care of properly closing the streams and the readers/writers for you.

  Make sure that the code you write in these listeners does not pass these streams to any method that would try to close them.

### <a name="nsc"></a> Network Security Configuration considerations

Since Android API level 24 a new feature has been introduced which can be used to control the network security policies an application must adhere to. This is **[Network Security Configuration](https://developer.android.com/training/articles/security-config.html)** and may be used to customize things such as the set of trusted CAs (for HTTPS connections), certificate pinning and cleartext traffic, etc..

HttpC is affected by any potential settings. Most notably, since API level 23, the default set of trusted anchors **no longer contain user CAs**, that is, custom certificates that you manually install on your device. This yields `SSLException`s when you try to use HttpC with a server whose CA is not among the ones the system trusts by default. The solution is to provide a properly configured `<trust-anchors>` tag in the NSC XML. Check out the details in the Android docs.

### <a name="manager_configurators"></a> Manager configurators

What is considered to be the _configuration_ of a `HttpConversationManager` has been touched in the previous chapters briefly. To recap, this is the set of user exits (filters, listeners, observers, whatnot...) that is registered on a particualr conversation manager instance.

The `HttpConversation` library defines a special interface, called `com.sap.smp.client.httpc.IManagerConfigurator` which defines a single method: `configure()`. This interface is implemented by classes which have the responsibility of configuring `HttpConversationManager` instances by adding filters, listeners, etc. to it, as needed.

Classes implementing `IManagerConfigurator` are therefore called _manager configurators_. Using these classes is not strictly required but they may come in handy. The most notable (and probably the most frequently used) implementation is the `com.sap.smp.client.httpc.authflows.CommonAuthFlowsConfigurator` class found in the `HttpConvAuthFlows` library. However, applications themselves can define their own manager configurators if they frequently instantiate conversation managers which need to be configured the same way.

Let's look at an earier example and rewrite it using configurators:

```java
import ...;

class HardCodedCredentialFilter implements IRequestFilter {
    private static final String TRIED_ALREADY_FLAG = "HardCodedCredentialsTried";

    @Override
    public Object filter(ISendEvent event, IRequestFilterChain chain) {
        // Get the context to read conversation-scoped information from.
        Map<String, Object> convContext = event.getConversationContext().getStateMap(this.getClass().getName(), false);

        // Check if the hard-coded credentials have been tried already.
        if (!convContext.containsKey(TRIED_ALREADY_FLAG)) {
            // Assemble the BasicAuth header.
            final String user = "john_doe";
            final String password = "p4Ssw0rD1Z3";
            String encodedUserPassword = Base64.encodeToString(
                (user + ":" + password).getBytes(Charset.forName("UTF-8")),
                Base64.NO_WRAP);

            // Set it.
            event.getRequestHeaders().put("Authorization", "Basic " + encodedUserPassword);

            // Set the flag so that the next time we'll not retry the credentials.
            convContext.put(TRIED_ALREADY_FLAG, true);
        }
    }

    @Override
    public Object getDescriptor() {
        return this.getClass();
    }
}

enum FooAppConfigurator implements IManagerConfigurator {
    inst;

    @Override
    public HttpConversationManager configure(HttpConversationManager manager) {
        // Configure it by adding our custom filter to it.
        manager.addFilter(new HardCodedCredentialFilter());
        return manager;
    }
}

public class ExampleActivity extends Activity {
    @Override
    public void onStart() {
        IManagerConfigurator configurator = FooAppConfigurator.inst;
        configurator.configure(new HttpConversationManager(this))
                    .create(Uri.parse("http://www.example.com"))
                    .setResponseListener(event -> {
                        // Process the response.
                        ...
                    }).start();
    }
}
```

> Note how the `configure()` method returns its argument, allowing the user to make use of the already mentioned method chaining pattern.

With the use of a manager configurator we can clearly encapsulate the logic of configuring a `HttpConversationManager` into a single class. In the `onStart()` function, all we have to do is create a manager and then configure it using `FooAppConfigurator` (in our example we made it a singeton as it is stateless).

Using manager configurators is advisable because of clarity, even though the above code adding just a single filter could have been placed at one location in the app. It is up to you how you'd like to organize the related code. It's possible to simply use `CommonAuthFlowsConfigurator`, write your own configurator like in the above example or do both: create an app-specific configurator which under the hood makes use of the `HttpConvAuthFlows` library to actually configure the conversation manager. It all depends on what level of flexibility you need in your app.

However, overcomplicating your code with this optional component is not recommended. The vast majority of apps configure their conversation managers somewhere around application startup by using `CommonAuthFlowsConfigurator` to enable certain authentication types. That should be more than enough for most of the apps.

### <a name="sum3"></a> Summary

* The HttpC API is stream based but comes with helpers to make it easier to work with character payloads. There are some things to keep in mind though concerning how character encodings are handled and how streams are closed.
* Recent Android versions introduced **Network Security Configuration**, a new feature to control the networking policies of an application with. This needs to be set up correctly otherwise the system will not permit the communication that the application tries to initiate or the requests will run into unexpected errors. Becoming familiar with the related documentation is more than recommended.
* When configuring `HttpConversationManager`s, consider the use of manager configurators, especially `CommonAuthFlowsConfigurator` using which a whole bunch of well-known auth. flows can be configured easily.

## <a name="hcaf"></a> Common authentication/authorization flows

Commonly used authentication and authorization flows are implemented in the `HttpConvAuthFlows` library. It is built on top of `HttpConversation` and its main class is `CommonAuthFlowsConfigurator` which is a manager configurator implementation using which application developers can configure these flows with ease.

The configurator itself can be parametrized with various options and user exits. The next subsections describe the details about each supported flow.

It's recommended to take a look at the API documentation of `CommonAuthFlowsConfigurator` which also contains a lot of useful info. This class will simply be referred to as _the configurator_ or _common configurator_ hereafter.

### <a name="basic_auth"></a> Basic Authentication

> **IMPORTANT NOTE**: Before considering the use of the Basic Authentication scheme, be advised that it has its inherent limitations (arising either from its simplicity or from ambiguities caused by the standard). These are:
>
> * This scheme can work reliably only with usernames and passwords **which do not contain characters outside the `US-ASCII` range**. This is because certain platforms encode the BasicAuth header with a different encoding. Unfortunately, the standard is not very clear from this aspect and thus certain client and server solutions use different encodings. For example, iOS native applications encode the header using `ISO-8859-1` encoding whereas Android native applications use `UTF-8`.
>
>   Therefore, use the BasicAuth scheme only if you understand these limitations and **when you don't plan to let your end users pick credentials that might contain Unicode characters outside the `US-ASCII` range**.
>
>   If you insists on using non-ASCII characters in credentials then we urge you to **switch to another authentication method**, like SAML2. Otherwise, you will not be able to build a bullet-proof solution on top of BasicAuth. You have been warned!
> * Usernames are separated from passwords in the BasicAuth HTTP request header using the colon `:` character. It is highly recommended to disallow the use of this character in user credentials too to avoid parsing errors of the credentials on the server side.
> * As credentials are transferred effectively in plain-text, it is not recommended to use this scheme over unencrypted HTTP.

`HttpConvAuthFlows` supports BasicAuth via its `supportBasicAuthUsing()` method of the configurator. Use this method to add a so-called username/password provider (i.e. an implementation of the `com.sap.smp.client.httpc.authflows.UsernamePasswordProvider` interface). When a challenge occurs, the provider is going to be called which then can either produce the correct credentials or return nothing (which of course cancels the conversation).

As multiple providers may be added, the first one that succeeds in returning credentials wins and the iteration over providers is stopped.

Let's modify our hard-coded credential example with something more realistic:

```java
import ...;

/**
 * Fragment class that displays a username/password input with 'Login' and 'Cancel'
 * buttons in a dialog.
 */
class UsernamePasswordDialogFragment extends DialogFragment {
    /**
     * Returns a future that produces the credentials or null once one of the buttons is tapped
     * by the user.
     *
     * @return the future using which credentials may be returned, always non-null
     */
    Future<UsernamePasswordToken> getEnteredCredentials() {
        // As this fragment is displayed the user may start entering the credentials. Once
        // a button is pressed the result of the returned future should be set.
        ...;
    }
}

public class ExampleActivity extends Activity {
    @Override
    public void onStart() {
        new CommonAuthFlowsConfigurator(this)
            .supportBasicAuthUsing(event -> {
                // Initialize a dialog fragment.
                final UsernamePasswordDialogFragment fragment = new UsernamePasswordDialogFragment();

                // Get the future object through which the results are going to be retrieved.
                Future<UsernamePasswordToken> credentials = fragment.getEnteredCredentials();

                // Show the dialog on the UI thread.
                runOnUiThread(() ->) {
                    fragment.show(getActivity());
                });

                // Wait for the results.
                try {
                    return credentials.get();
                } catch (InterruptedException | ExecutionException e) {
                    return null;
                }
            }).configure(new HttpConversationManager(this))
            .create(Uri.parse("http://www.example.com"))
            .setResponseListener(event -> {
                // Process the response.
                ...
            }).start();
    }
}
```

The actual implementation of `UsernamePasswordDialogFragment` is omitted for brevity. What matters is how the `UsernamePasswordProvider` implementation is organized. The dialog is shown using the enclosing activity and then the credentials are acquired using a `java.util.concurrent.Future` object. The call to `get()` will block the thread running the conversation 'till the dialog produces valid credentials or a null value (if it was cancelled).

Blocking the thread running the conversation is a legal thing to do, provided that it is indeed a background thread. By default, this is the case.

> For the details concerning threading, consult [this section](#executors) explaining the details.

UI-relevant HttpC user exit implementations must manage the activity references properly to prevent leaks. The above code would be complete if `ExampleActivity` cancelled the running conversation in `onStop()`, for example. It is not the goal of this guide to teach you how to do proper background-thread management on Android in general. When writing user exits - like the username/password provider above - make sure to adhere to general threading guidelines and techniques of the Android platform.

Of course, we can easily integrate our previous hard-coded credentials into the flow. The solution would be to create another implementation of `UsernamePasswordProvider` which returns the credentials unconditionally:

```java
import ...;

public class ExampleActivity extends Activity {
    @Override
    public void onStart() {
        new CommonAuthFlowsConfigurator(this)
            .supportBasicAuthUsing(event -> {
                // Get the context to read conversation-scoped information from.
                Map<String, Object> convContext = event.getConversationContext().getStateMap(this.getClass().getName(), false);

                // Check if the hard-coded credentials have been tried already.
                if (!convContext.containsKey(TRIED_ALREADY_FLAG)) {
                    // Set the flag so that the next time we'll not retry the credentials.
                    convContext.put(TRIED_ALREADY_FLAG, true);

                    // Return the credentials.
                    final String user = "john_doe";
                    final String password = "p4Ssw0rD1Z3";
                    return new UsernamePasswordToken(username, password);
                }

                return null;
            }).supportBasicAuthUsing(event -> {
                // Initialize a dialog fragment.
                final UsernamePasswordDialogFragment fragment = new UsernamePasswordDialogFragment();

                // Get the future object through which the results are going to be retrieved.
                Future<UsernamePasswordToken> credentials = fragment.getEnteredCredentials();

                // Show the dialog on the UI thread.
                runOnUiThread(() ->) {
                    fragment.show(getActivity());
                });

                // Wait for the results.
                try {
                    return credentials.get();
                } catch (InterruptedException | ExecutionException e) {
                    return null;
                }
            }).configure(new HttpConversationManager(this))
            .create(Uri.parse("http://www.example.com"))
            .setResponseListener(event -> {
                // Process the response.
                ...
            }).start();
    }
}
```

Note that the hard-coded credential provider implementation still has to account for whether the credentials have been tried already, just like our `HardCodedCredentialFilter` implementation had to in one of our previous examples. The user exits of `HttpConvAuthFlows` are not exempt from having to do proper state management. Never forget that these user exits run within the scope of a conversation.

Also note that the order in which the providers are added to the configurator does in fact matter. The above example will try to use the hard-coded credentials first and only fall back to showing the dialog if those fail.

> If you would like to try out a working sample application using Basic Authentication then go to `SDK_install/android/samples` directory and open `build.gradle` file with Android Studio. Sample applications will be imported and compiled, `BasicAuth` is the sample application for Basic Authentication. You can debug the application source code and feel free to modify it and experiment. 

### <a name="x509"></a> Mutual SSL/TLS authentication

In other words: X.509 client certificate authentication. This is when during the SSL/TLS handshake the server asks the client to present a suitable client certificate.

As HttpC builds on top of `HttpsURLConnection`, it makes use of the platform-default mechanism to hook into the handshake. Consequently, this authentication type is supported by the `HttpConversationManager` class directly and there's no need to use `HttpConvAuthFlows` for this purpose.

The conversation manager defines an `addKeyManager()` method using which a `javax.net.ssl.X509KeyManager` implementation may be registered. If you are not familiar with this interface then now is the time to read the guides and the API documentations on the official Android site.

> Likewise, `HttpConversationManager` defines an `addTrustManager()` method for hooking into the server certificate validation mechanism via `javax.net.ssl.X509TrustManager`s.

In short, it is a user exit which gets called to return the client certificate and the associated private key at the right time. Implementing a key manager properly can ensure that the certificate and the private key spend the least amount of time in memory as possible and that they are used only for the handshake after which they are thrown away immediately.

Just like BasicAuth, multiple key managers may be added in sequence. The order of addition counts: the first one to produce a proper certificate/private key pair wins and the rest are not invoked.

> Although possible, it is not recommended to undertake lengthy actions from within a key manager. This is because unlike BasicAuth, the conversation is waiting for it **in the middle of the SSL/TLS handshake**. If a timeout occurs then the request fails, even though the key manager might eventually produce a valid certificate.

There are a dozen of examples out there how to write a proper key manager. Below is an example that reads a .P12 file which is assumed to contain a single certificate. Only the relevant method implementations are shown, the rest are omitted :

```java

import ...;

public class FileBasedX509KeyManager implements X509KeyManager {
    private final String file;
    private final String password;
    private final Context context;

    private final String alias;
    private final String keyType;
    private final X500Principal issuer;

    public FileBasedX509KeyManager(File file, String password, Context context) {
        // Put the key inputs into private members. 'load()' will need them to
        // fetch the contents of the PKCS12 file.
        this.file = file;
        this.password = password;
        this.context = context.getApplicationContext();

        // Initialize the key store and load the file.
        KeyStore keyStore = null;
        try {
            // Expecting a single certificate in the file.
            keyStore = load();
            Enumeration<String> aliases = keyStore.aliases();
            alias = aliases.nextElement();

            // Get the certificate to extract the issuer. This info is required when selecting
            // the client aliases.
            X509Certificate certificate = (X509Certificate) keyStore.getCertificate(alias);
            keyType = certificate.getPublicKey().getAlgorithm();
            issuer = certificate.getIssuerX500Principal();
        } catch (KeyStoreException | IOException | CertificateException | NoSuchAlgorithmException e) {
            throw new IllegalStateException("Unable to initialize key manager", e);
        }
    }

    ...

    @Override
    public String[] getClientAliases(String keyType, Principal[] principals) {
        if (this.keyType.equals(keyType)) {
            boolean okay;
            if (principals != null) {
                okay = false;
                for (Principal principal : principals)
                    if (issuer.equals(principal)) {
                        okay = true;
                        break;
                    }
            } else okay = true;

            if (okay) return new String[]{alias};
        }
        return null;
    }

    @Override
    public String chooseClientAlias(String[] keyTypes, Principal[] principals, Socket socket) {
        for (String keyType : keyTypes) {
            String[] aliases = getClientAliases(keyType, principals);
            if (aliases != null) return aliases[0];
        }

        return null;
    }

    @Override
    public X509Certificate[] getCertificateChain(String alias) {
        if (this.alias.equals(alias)) {
            try {
                KeyStore keyStore = load();
                Certificate[] certificateChain = keyStore.getCertificateChain(alias);
                X509Certificate[] x509CertificateChain = new X509Certificate[certificateChain.length];
                for (int i = 0; i < certificateChain.length; i++) x509CertificateChain[i] = (X509Certificate) certificateChain[i];
                return x509CertificateChain;
            } catch (KeyStoreException | IOException | CertificateException | NoSuchAlgorithmException e) {
                return null;
            }
        } else
            return null;
    }

    @Override
    public PrivateKey getPrivateKey(String alias) {
        if (this.alias.equals(alias)) {
            try {
                KeyStore keyStore = load();
                return (PrivateKey) keyStore.getKey(alias, x509Settings.getPassword().toCharArray());
            } catch (KeyStoreException | IOException | CertificateException | NoSuchAlgorithmException | UnrecoverableKeyException e) {
                return null;
            }
        } else
            return null;
    }

    private KeyStore load() throws KeyStoreException, IOException, CertificateException, NoSuchAlgorithmException {
        KeyStore keyStore = KeyStore.getInstance("pkcs12");
        keyStore.load(new FileInputStream(file), password.toCharArray());
        return keyStore;
    }
}

```
> If you would like to try out a working sample application using Mutual SSL/TLS authentication then go to `SDK_install/android/samples` directory and open `build.gradle` file with Android Studio. Sample applications will be imported and compiled, `ClientCert` is the sample application for Mutual SSL/TLS authentication. You can debug the application source code and feel free to modify it and experiment.

### <a name="saml2"></a> SAML2

SAML2 is a widespread and rather comprehensive standard for various purposes, including single sign-on, identity federation, access management, etc.. An overview of the standard can be found at the official [Oasis documentation site](http://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0-cd-02.html) which is a must-read before reading this section as the nomenclature used below makes heavy use of the concepts layed down by the standard.

#### <a name="saml2_overview"></a> Overview

The SAML2 support in HttpC allows a mobile application developer to integrate the application with a SAML2-based web single sign-on infrastructure. This is the use case what the standard commonly refers to as the _Web Browser SSO Profile_. This actually boils down to displaying a web-based login form of some sort using which the user can authenticate and then access the protected resource.

HttpC is a native networking solution. If a SAML2-protected resource needs to be accessed on the _Service Provider (SP)_ then an authenticated session must be established beforehand. The goal of SAML2 support is therefore to detect the need for the authentication, orchestrate the web-based login and then resume the original request that targeted the protected resource.

Originally, the SAML2 web SSO profile has been designed purely for browser-based applications. There, accessing a protected resource is most often done by opening a link. If an authentication is required then the SP redirects the browser to an _Identity Provider (IdP)_ which handles the authentication and then redirects back to the SP. At the end, the browser is redirected to the original resource but this time an authenticated session is already present.

With a little modification, the above flow is reusable for native requests (i.e. those which are initiated by a HTTP client that is not a web browser). Here's how all this takes place using HttpC using a properly configured `HttpConversationManager`:

1. The `create()` method is used to create a conversation and send a request. The request targets a protected endpoint hosted by the SP.
1. The SP responds with an authentication challenge, telling the client that authentication is required before accessing the endpoint.
1. The SAML2 support in HttpC detects the challenge in the received HTTP response and prepares for performing the SAML2 authentication.
1. An embedded web browser is displayed by HttpC in which the IdP login form is displayed.
1. The user authenticates itself to the IdP. Note that this step is completely in the hands of the IdP and is not configurable from within HttpC. Different IdPs might employ vastly different authentication mechanisms, ranging from simple username/password input forms to more complex use cases involving biometric identifications or hardware keys. All this depends on what the IdP server supports and how it is configured. This part is completely out of the hands of HttpC.
1. As the SAML2 authentication completes, the series of redirections within the displayed embedded browser (called the _web view_, shortly) eventually lead back to the SP with the SAML assertion (i.e. the _SAML Response_) in hand. The latter is the piece of information what represents the logged-in user.
1. The SP then establishes the authenticated session and sends a final response that the mobile client is able to detect. This will then cause the web view to disappear.
1. With the authenticated session at hand (which is represented by HTTP cookies), HttpC can restart the conversation and retry the original request. If successful, the resource is going to be returned to the client.

The most important points of integration with a SAML2-based infrastructure can be found in steps 3, 7 and 8, namely:

1. The Service Provider must be able to signal the SAML2 challenge in a predefined way that the client can easily detect.
1. When the flow completes successfully, the Service Provider is required to send a final response signaling the end of the flow. This allows the mobile client to dismiss the embedded web browser and resume the original request.
1. The Service Provider must use HTTP cookies to establish the authenticated session.

Requirements 1 and 3 follow from the SAML2 standard. However, no. 2 is something that must be implemented by the Service Provider **on top of** the compliance with the standard. The _final response_ in question is actually expected to be a redirect which can be easily detected in the web view by observing the opened URL. The endpoint producing the final response is what HttpC calls the _finish endpoint_. This term is going to be important later.

#### <a name="saml2_details"></a> Details of the SAML2 support

As said before, HttpC supports SAML2 Web SSO. This profile is configurable from the aspect of what bindings are used for the SAML2 request and the SAML2 response. HttpC supports the following setups:

* HTTP POST and HTTP Redirect Binding for the SAML2 request
* HTTP POST and HTTP Artifact Binding for the SAML2 response

> HTTP Artifact Binding for the SAML2 request is not supported.

> The SAML2 standard does not allow HTTP Redirect Binding for the SAML2 response.

HttpC expects a challenge when a protected resource is accessed without an authenticated session. The challenge itself is a SAML2 request that the SP sends back in the response with one of the supported bindings. Out of these, HTTP POST binding is used more frequently. As this consists of a `text/html` response that contains a plain-old HTML form, its detection requires HttpC to parse the output HTML and search its contents for the form to see if it contains the SAML request.

Doing this can be slow, therefore the SP can implement support for the _identifying header_. This is an extra HTTP response header that can be sent as part of the SAML2 challenge which makes detection a lot easier. If it's present, the client concludes that a challenge has been received. This eliminates the need for doing expensive HTML parsing.

> This header is not part of the SAML2 standard. It is an SAP-specific extension to make things easier for the mobile client. SAP Mobile Platform Server and the Mobile Services on SAP Cloud Platform support this header.

#### <a name="saml2_config"></a> SAML2 configuration

To enable SAML2 support in HttpC, a configuration must be provided that tells a `HttpConversationManager` how to detect the challenge and what endpoints to use in the SAML2 flow.

Parts of this configuration have already been mentioned in the previous sections. Below is the summary:

* _Identifying Header_: This is the header that the SP is recommended to send along with the HTTP response containing the SAML2 request if it's configured to use HTTP POST Binding. Recall that the presence of the SAML2 request itself represents the authentication challenge. This header is optional.
* _Finish Endpoint_: This is the endpoint **hosted by the Service Provider** which must be opened when the challenge is detected. Simply put, when the embedded browser is displayed, this endpoint is the first thing opened by it. The SP is then expected to respond with a SAML2 challenge **again** which starts the entire SAML2 flow inside a browser environment.
* _Finish Endpoint Parameter_: This is the request parameter that is used to signal the end of the SAML2 flow. If the embedded browser opens the finish endpoint again but with this extra request parameter in the URL then the web view is dismissed and the conversation is restarted.

One might wonder why does the flow **start** by opening the **finish** endpoint? The reason is that the above described SAML2 flow is a reactive authentication mechanism: one must open the protected resource to provoke the challenge. From the aspect of the mobile client, the finish endpoint therefore acts like a special protected resource that is expected to require the same kind of authenticated session as the original resource.

A compliant Service Provider therefore should:

* ...support the identifying header (when using HTTP POST Binding) to make it easier for the mobile client to detect a challenge,
* ...provide a finish endpoint that can produce the same kind of SAML2 challenge that the mobile client would get when accessing other protected resources,
* ...make sure that when the authentication to the finish endpoint succeeds, it is reopened with the extra finish endpoint parameter. The latter is usually implemented using a simple HTTP redirect.

All these features are required from the SP to make it work with the SAML2 support of HttpC.

> The previously mentioned SAP server-side solutions (SAP Mobile Plaform server, Mobile Services on SAPcp) already support this mechanism. When a mobile application registered in SMP or Mobile Services on SAPcp accesses a protected OData resource, the server ensures that the established session grant access only to that resource. All this happens transparently to the client.

### <a name="saml2_setup"></a> Setting up a manager

To configure a `HttpConversationManager` for SAML2, again the `CommonAuthFlowsConfigurator` class should be used. The `supportSaml2AuthUsing()` method must be called with an implementation of `com.sap.smp.client.httpc.authflows.SAML2ConfigProvider`. Then, all conversation managers configured with this configurator will have SAML2 enabled.

The SAML2 configuration provider is a similar asynchronous user exit like the one used for Basic Authentication or Mutual SSL/TLS. The difference is that meanwhile these two providers actually produce credentials to authenticate with, the SAML2 configuration provider - as its name implies - produces only configuration.

The typical configuration for the above mentioned SAP servers is as follows:

* Identifying Header: `com.sap.cloud.security.login`
* Finish Endpoint: `<server scheme, host and port>/SAMLAuthLauncher`
* Finish Endpoint Parameter: `finishEndpointParam`

> Note that the actual values of the identifying header and the finish endpoint parameter are not relevant. Their sheer presence is enough for the SAML2 functionality to work.

If you'd like to integrate SAML2 support in your app, implement a new SAML2 configuration provider and return a new configuration from the `onSAML2Challenge()` method.

> Similarly to Mutual SSL/TLS, a configuration provider is recommended to avoid lengthy computations or too many UI interactions. This is because a configuration provider is invoked before __every__ request.

No matter what configuration you use, the following entry needs to be added to the `AndroidManifest.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" ...>

    ...

    <application ...>

        ...

        <activity android:name="com.sap.smp.client.httpc.authflows.SAML2AuthActivity"></activity>
    </application>
</manifest>
```

> Provided that you are using the default SAML2 web strategy which is indeed the case when you don't customize the web strategies in any way. Read [this part](#webstrategies) if you're interested in the details.

### <a name="sum4"></a> Insights and summary

> Read this section if you'd like to better understand the motives behind the SAML2 support and why the SAML2 flow is implemented the way it is described above. This might shed some light on the challenges supporting SAML2 on the mobile with a native solution. Furthermore, the insights contained herein might help you understand what can and what cannot be expected from this solution. Considering that SAML2 is one of the most complex auth. standards in the industry, this should prove useful.

The SAML2 support in HttpC expects that the Service Provider meet additional requirements on top of being a fully compliant SAML2 SP. This is caused by that the Web Browser SSO Profile has originally been designed for plain-old HTML form-based web applications. There, posting a HTML form leads to a full page reload. This mechanism is what SAML2 "hijacks" for the purpose of authentication which if succeeds will eventually produce the results of the original form post.

A native HTTP client solution operating inside a mobile application however has much more limitations. It is unable to participate in web-based authentication flows transparently. It needs to explicitly open an embedded browser to allow the user to log in. Consequently, knowing when to start and when to end the flow is critical.

On the other hand, embedding web views in native mobile applications on Android has its own limitations too. Due to security considerations, a native application cannot access the actual HTML contents the web view is displaying. It can only see what is the URL of the page that the embedded web view is about to open. Hence the need for the finish endpoint parameter which can be used to easily detect the end of the flow.

These mobile client-specific peculiarities imply the need for the above extensions to a normal SAML2 authentication flow. However, the SAP Mobile Platform server and Mobile Services on SAP Cloud Platform both properly implement the necessary requirements.

> It should be noted that the SAML2 IdP does not have to be _customized_  (i.e. augmented with extra, mobile-specific features) to take part in this flow. This is because all communication with the IdP takes place inside the embedded web view which implies the following:
>
> * The SAML2 assertions are not intended to be exposed to the client and as such are not retrievable using HttpC.
> * The IdP login screen cannot be altered in any way. Any customizations required there must be implemented on the IdP itself.
> * The embedded web view is opened only for the sole purpose of logging the user in. An IdP login screen might offer other functions beyond simple authentication (for ex. links to _Forgot Your Password?_ or _Registration_ pages) but it must be ensured that eventually the flow ends in a successful login. Without it, the user might be "stuck" in the embedded web view, surfing around meanwhile a running HTTP conversation awaits for the authenticated session. To prevent this, avoid placing such distractions on the IdP screen.

-
 
> If you would like to try out a working sample application using SAML2 authentication then go to `SDK_install/android/samples` directory and open `build.gradle` file with Android Studio. Sample applications will be imported and compiled, `Saml2` is the sample application for SAML2 authentication. You can debug the application source code and feel free to modify it and experiment. You can configure your SAP Mobile Platform server and Mobile Services on SAP Cloud Platform in the settings menu of the application.

## <a name="oauth2"></a> OAuth2

OAuth2 is one of the most complex authorization flows supported by the `HttpConvAuthFlows` library. However, setting it up is relatively easy. If the mobile application uses services hosted by Mobile Services on SAPcp, getting things up and running is even easier.

> SAP Mobile Platform on-premise server does not support OAuth2 at the time of writing this guide.

This section provides all the info to help you to set up a `HttpConversationManager` to communicate with OAuth2 protected endpoints.

It is assumed that you have a solid understanding of how OAuth2 works in general. Should there be any black spots for you, please read through some of the introductory materials on the Internet before proceeding to read this part of the guide.

### <a name="oauth2_details"></a> Supported grant types

The OAuth2 standard defines various _grant types_ which differ in the method how the access token is acquired. HttpC supports the following types:

* Authorization Code Grant
* Password Grant
* Client Credentials Grant

> Although the standard mentions it, the so-called _Implicit Grant_ type is not supported by HttpC. A variation of _Authorization Code Grant_ is supported instead where the `client_secret` parameter is optional. This follows well-known best practices.

Unquestionably, the most widely used grant type is _Authorization Code Grant_ which is often used to implement an OAuth2-based SSO solution. Think about the various apps and services that let you register and log in using your Facebook or Google Account credentials, for example.

### <a name="oauth2_ep"></a> Extension points of the standard

Before discussing what the OAuth2 features of `HttpConvAuthFlows` are, let's take a closer look at the standard in terms of extension points. The below emphasized points are what the OAuth2 support in HttpC is built on top of.

The no. 1 goal of HttpC in supporting OAuth2 is to offer a reactive mechanism that can respond to a challenge presented by the server and trigger an OAuth2 authorization flow. However, the OAuth2 standard does not define what an _OAuth2 challenge_ is. In fact, every Resource Server (in OAuth2 terms) can have its own way of asking the client for an access token. The first question to answer is therefore how to detect a challenge?

Similarly to SAML2, when a challenge is detected the OAuth2 flow must be started. The similarity does not end here: like SAML2, the OAuth2 support in HttpC also requires a configuration to know which endpoints to talk to, where to acquire the access and refresh tokens, etc..

The third thing is the scope of the access token. Remember that OAuth2 is an **authorization** standard and the access token is something that implicitly represents a set of resources, i.e. those that can be accessed with it. Again, the standard does not define any means to derive what endpoints the access token might be good for.

The above questions arise for all supported grant types independently. The next section provides the anwers HttpC gives to all these.

### <a name="oauth2_server_support"></a> The `OAuth2ServerSupport` interface

Enabling OAuth2 support is done following the same logic as other auth. types. The `supportOAuth2Using()` method of `CommonAuthFlowsConfigurator` should be called with an implementation of `com.sap.smp.client.httpc.authflows.oauth2.OAuth2ServerSupport`. Such objects are called _server support objects_, hereafter.

This interface is what enriches the generic OAuth2 mechanism in HttpC with the specifics related to a particular server. Effectively, this is what answers the questions introduced in the previous section.

Here's how a server support object is integrated into the overall OAuth2 flow:

1. The `isSupportedEndpoint()` method is called with the URL of the currently executing request whenever an HTTP response with status code between 400 (inclusive) and 500 (exclusive) is received. This method then can check if the URL points to a server whose OAuth2-specifics are known. If this method reports false then the current server support object is skipped entirely.

   For example, a server support object tailored to SAP Cloud Platform can return true only if the URL is pointing to the expected SAPcp service endpoint. If the request is fired for any other endpoint, the SAPcp-specific server support object will not be considered competent and HttpC will look for another object.
1. A HTTP 4xx response means that the client sent an incorrect request. However, this is not necessarily an indication that an OAuth2 access token would have been required. To decide this, the `canRecognizeChallengeInResponse()` method is called on the server support object.

   > Remember: this method is called only if the server support object passed the previous step, that is, it supports the endpoint.

   This method has access to the entire response through its `IReceiveEvent` argument. It can be used to analise the original request URL, response headers and even the payload. If the server implements a clear way of signaling the need of an access token, here this can be checked. The result of this method is an assessment about the response. If this states that the HTTP response at hand is indeed a challenge then the flow continues with the current server support, otherwise the response is not processed any further from the aspect of OAuth2.
1. At this point the HTTP 4xx response has been analised and found that it is indeed a challenge presented by a supported endpoint. The `configForChallenge()` method is then called to get the configuration in the form of an `com.sap.smp.client.httpc.authflows.oauth2.OAuth2Config` object.

   For each grant type there exists a corresponding subclass of `OAuth2Config`. The `configForChallenge()` method therefore has to return an object belonging to the appropriate subclass. This is what decides what grant type is to be used.

   For example, `OAuth2AuthCodeGrantConfig` pertains to the _Authorization Code Grant_ type and contains properties such as the _authorization endpoint_, _client ID_ and _client secret_, etc..
1. The next step depends on the actual grant type. The most complex from this aspect is the _Authorization Code Grant_ with an authorization endpoint that uses the HTTP URL scheme. This is the most common use case and causes an embedded web view to be displayed. As the web view appears, the authorization endpoint URL is loaded which triggers the first part of the authorization flow. The end of this step is the authorization code that concludes the user's direct involvement and instructs HttpC to dismiss the web view.

   > If you are already familiar with how the SAML2 flow works you might see the similarity here. Again, an embedded webview has to be used, however this fits in with the flow much more smoothly. This is because the OAuth2 standard explicitly mandates the use of a HTTP redirect to the _redirect URI_ with the authorization code included as query parameter. That makes detecting the end of the flow possible without the need for additional requirements from the server side.
1. Independently of the grant type, at a certain point acquiring the access and refresh tokens becomes possible. This is all done transparently and without the involvement of the server support object.
1. Once the tokens are at hand, two things are performed:
    * The tokens are placed in a so-called OAuth2 token storage represented by an instance of `OAuth2ServerSupport`. Prior to saving the tokens, the `transformRequestUrlToStorageUrl()` method of the server support object in use is called. This is what determines the scope of said tokens. Read the next section concerning how this works exactly.
    * The original request is retried (i.e. the conversation gets restarted) but this time the access token is placed in the HTTP request inside the `Authorization: Bearer ***` header.

The built-in OAuth2 support is intelligent enough to retry previously acquired access tokens if they are found in the storage. Also, if a refresh token is also present then HttpC will try to use it first to renew the access token, should it expire. The above flow unfolds in its entirety only if none of these attempts are successful in acquiring a valid access token locally.

### <a name="oauth2_token_scope"></a> Scope of access and refresh tokens

As previously mentioned, the OAuth2 standard does not define in any way what kind of resources might an access token be used for. For example, the OAuth2 token acquired by a particular mobile application on SAPcp will grant access only to the OData resources exposed by the corresponding OData service hosted by Mobile Services. For other kinds of services, you'd need a different token.

A mechanism therefore is required which associates a URL with the corresponding access and refresh tokens. If this association is in place, HttpC can easily find the right token for a particular HTTP endpoint.

The association is implemented using the `transformRequestUrlToStorageUrl()` method of the server support object. It is recommended to become familiar with the API documentations of this method later on.

Here's what happens: when a freshly acquired access token/refresh token pair is to be saved, this method is called to transform the concrete request URL into a symbolic one for identification purposes. The scope of the tokens is therefore going to be represented by the transformation itself. Imagine the below transformation:

```
http://foo-services.com/bar/sales/** -> http://foo-services.com/bar
http://foo-services.com/bar/users/** -> http://foo-services.com/bar
http://foo-services.com/bar/products/** -> http://foo-services.com/bar

http://foo-services.com/bar/admin/** -> http://foo-services.com/bar/admin
```

> The `**` sequence denotes a subpath of arbitrary depth.

On the left are concrete URLs, on the right are symbolic ones returned by `transformRequestUrlToStorageUrl()`.

Let's say that a native request is sent to `http://foo-services.com/users/johndoe?detail=name,email,phone`. If this is OAuth2-protected then the corresponding server support object is called to help acquire the OAuth2 tokens. After successful authorization, this request URL is then transformed in the above way, hence producing `http://foo-services.com/bar`. This URL is then used as a key into a key-value storage represented by the `com.sap.smp.client.httpc.authflows.oauth2.OAuth2TokenStorage` in use. Later on, when another request is fired against the same endpoint (or for another endpoint that transforms to the same symbolic URL) then the tokens can be found and used.

Note that the above example implies that an access token acquired for the `users` sub-service of `bar` is usable for the `sales` and `products` sub-services as well. However, these access tokens would not be suitable for the `admin` sub-service as it transforms to a different symbolic URL.

This mechanism is especially effective when sending a request **after** the tokens have been acquired. There, HttpC can efficiently read the tokens from the storage and try them when sending a request without waiting for an authentication challenge. This proactive behaviour is built into the library and requires no further coding.

### <a name="token_storages"></a> Token storages

An OAuth2 token storage is represented by an instance of `OAuth2TokenStorage`. The library by default uses an in-memory storage that stores tokens in a `java.util.Map`. If you'd like to keep tokens around for a longer period (for ex. by storing them in some kind of encrypted storage) you can implement your own token storage class.

To use a custom implementation, use the `setTokenStorage()` setter of `CommonAuthFlowsConfigurator` **prior to** configuring a `HttpConversationManager` object.

> Note how token storages are explicitly separated from server support objects. This is because the two things do actually represent different aspects of the OAuth2 mechanism. An app can easily make use of its own storage mechanism meanwhile using server support implementations from other parties.

The storage is used at the end of a successful OAuth2 flow to store the tokens and during request sending to retrieve the access token to be sent along. For the details on how to implement a storage, read the API documentation of `OAuth2TokenStorage`.

### <a name="oauth2_sapcp"></a> Enabling support for SAPcp

The library ships the `SAPOAuth2ServerSupport` class that is an implementation of `OAuth2ServerSupport` tailored to SAP Cloud Platform. To make use of it, instantiate an object of this type and pass it to the `supportOAuth2Using()` method of the common configurator.

This object requires that the server URL, client ID and secret, scope and redirect URI be specified during initialization. This will drive its internal logic to know which endpoint is to be supported, how to detect the challenge and what the scope of the acquired access/refresh tokens is.

At the time of writing this guide, SAP Cloud Platform supports the _Authorization Code Grant_ and _Client Credential Grant_ types but `SAPOAuth2ServerSupport` produces OAuth2 configuration only for the former.

The above parameters can be taken directly from the admin UI of the Mobile Services cockpit. The minimum configuration requires only the server URL and the client ID. If the client secret is left empty and the OAuth2 endpoints are not customized in any way then `SAPOAuth2ServerSupport` can make use of the default values it derives internally for the rest of its parameters. In the majority of the cases this should be enough.

### <a name="oauth2_web"> Authorization Code Grant & HTTP

If you're using OAuth2 Authorization Code Grant - which is the case if you rely on the built-in `SAPOAuth2ServerSupport` - then the following entry needs to be added to the `AndroidManifest.xml` of your app:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" ...>

    ...

    <application ...>

        ...

        <activity android:name="com.sap.smp.client.httpc.authflows.oauth2.OAuth2AuthActivity"></activity>
    </application>
</manifest>
```

> Provided that you are using the default OAuth2 web strategy which is indeed the case when you don't customize the web strategies in any way. Read [this part](#webstrategies) if you're interested in the details.

As this grant type with an HTTP-based authorization URL causes an embedded browser to be displayed, the above entry is necessary for this to be possible. `OAuth2AuthActivity` is what coordinates the web view and displays the login and authorization screens.

### <a name="sum5"></a> Summary

The below things are worth remembering about the OAuth2 support of HttpC:

* What server we are trying to communicate with? How does it handle OAuth2? The answer is provided by an `OAuth2ServerSupport` object that must be added to the common configurator. This knows what endpoints it supports, how to detect a challenge, what parameters are required to initiate the authorization flow and what the scope of the acquired tokens is going to be.
* How should we store the tokens? To configure this, an `OAuth2TokenStorage` must be implemented and set via the corresponding setter of the common configurator. Remember, this means that you're going to be in charge of storing the tokens HttpC acquires for you.
* For SAP Cloud Platform servers, the `SAPOAuth2ServerSupport` is the most elegant solution. It's a class that should be instantiated with the proper parameters taken from the Mobile Services Cockpit. This should be enough to allow the mobile application to work with our OData endpoints hosted in the cloud.

> If you would like to try out a working sample application using OAuth2 authentication then go to `SDK_install/android/samples` directory and open `build.gradle` file with Android Studio. Sample applications will be imported and compiled, `OAuth2` is the sample application for OAuth2 authentication. You can debug the application source code and feel free to modify it and experiment. You can configure your Mobile Services on SAP Cloud Platform in the settings menu of the application.


## <a name="2fa"></a> Two-Factor Authentication

Two-Factor Authentication (also known as: One-Time Pin Authentication) is an additional authentication step supported by both SAP Mobile Platform server and Mobile Services on SAP Cloud Platform. It can be enabled on the administration cockpit at the security settings page. Consult the documentation of these products for the related info.

Enabling 2FA on the admin cockpit is not enough. The end user must install the **SAP Authenticator** mobile application and enable second factor authentication on his/her account. This must be done on the user administration page of the IdP.

> The IdP must be the same that the SMP server or the SAP Cloud Platform account is integrated with, of course.

This guide does not intend to go into the details of setting up the infrastructure. Please find the related materials on the SAP Help Portal.

### <a name="2fa_overview"></a> Overview

2FA usually takes place after successfully passing the first authentication stage. This can be anything that HttpC supports: Basic Authentication, Mutual SSL/TLS, etc.. When the second stage begins, an embedded web view is opened where a one-time passcode is asked from the user. This passcode can be obtained by opening the **SAP Authenticator** application preinstalled on the same device. Once the second factor authentication succeeds, the user is logged in.

> At this point it is strongly recommended to have a solid understanding of how SAML2 authentication works with HttpC. Please jump to [this section](#saml2) if you are not yet familiar with the topic. 2FA shares a lot of common traits with how SAML2 is implemented.

Just like SAML2 and (in most cases) OAuth2, Two-Factor Authentication also involves opening a web view. Here's how the flow takes place after the first authentication stage:

1. The need for the second-factor authentication is signaled by the server using an HTTP redirect. Along with the redirect response a special response header is sent. The client can use this information to realize that this is a special redirect and that 2FA is to be commenced.
1. The redirect points to the _One-Time Passcode form_. HttpC - instead of following this redirect with a native request as part of the running conversation - loads this URL in a newly opened embedded web view.
1. The OTP form loads and presents the user with the one-time pin challenge.
1. The user opens the **SAP Authenticator** app to obtain the pin code.
   > At the time of writing this guide, this is possible in two ways:
   > 1. The user taps on the link that navigates to the appropriate screen of **SAP Authenticator**.
   > 1. The user opens **SAP Authenticator** manually, navigates to the proper screen that generates the One-Time Pin for the current application and then copy-pastes the passcode into the OTP form.
1. Once the challenge is answered successfully, the server redirects to a URL that contains the _finish endpoint parameters_. This is very similar to the same-named concept of SAML2 and is used by the mobile client to detect the end of the flow in the web view.

At the end of the above process the running conversation is restarted and the request is retried. The authenticated state is captured by the HTTP cookies the server issues at the end.

### <a name="2fa_config"></a> OTP configuration

> Don't be confused: the terms _2FA_ and _OTP_ are often used interchangeably. They refer to the same thing.

The OTP configuration is supplied via an instance of `com.sap.smp.client.httpc.authflows.OTPConfigProvider`. Like other authentication methods, multiple instances of this provider may be registered with a `CommonAuthFlowsConfigurator` by calling the `supportOTPUsing()` method.

The configuration consists of the following attributes:

* _Response Header Name_: The name of the response header that must be looked for when receiving a HTTP 3xx response.
* _Response Header Value_: If a HTTP 3xx redirect is received which has a response header whose name equals to the previous attribute and its value equals to this attribute then HttpC considers this response a 2FA challenge.
* _Finish Endpoint Parameters_: The list of URL parameters to look for when the web view opens a new page. Once the 2FA process completes successfully, the server appends these parameters to the URL it redirects to. The client uses it to detect the end of the flow.
* _OTP Additional URL Parameters_: Additional URL parameters to append to the OTP form URL before opening it in a web view. This attribute is optional.

The default configuration for SAP Mobile Platform server and Mobile Services on SAPcp is as follows:

* Response Header Name: `x-smp-authentication`
* Response Header Value: `otp-challenge`
* Finish Endpoint Parameters: `finished=true`
* OTP Additional URL Parameters: `redirecttooriginalurl=false`

### <a name="2fa_comm"> Communication between **SAP Authenticator** and your app

The OTP form contains a link that can be used to open the **SAP Authenticator** application conveniently. If the user taps it, Authenticator is opened which generates the pin code and then calls back to your application.

The mechanism behind this communication is based on custom URI schemes. The above mentioned link has `sapauthenticator` as its URI scheme. The opened URL contains several parameters which helps the Authenticator identify which application is calling it.

As part of setting up **SAP Authenticator**, your application has to be registered with it which means adding the callback URL pointing to your application to the user's account. By convention, this URL should have a custom URI scheme that ends with `.xcallbackurl`.

> Such URLs are usually quite lengthy. To facilitate registration, **SAP Authenticator** provides a QR code reader using which one can easily add the URL of your app.

To make this work, you have to register this callback URL in your `AndroidManifest.xml` file, as follows:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" ...>

    ...

    <application ...>

        ...

        <activity android:name="com.sap.smp.client.httpc.authflows.OTPAuthActivity">
            <intent-filter>
                <data android:scheme="<your application identifier>.xcallbackurl" />
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

Now, when **SAP Authenticator** is called, it will be able to call back to your registered application with the passcode in the URL.

> How to properly set up **SAP Authenticator** and the entire 2FA landscape is out of the scope of this guide. The above discussion relies on that you have a solid understanding of how it works. You can find useful material about this topic on the SAP Help Portal.

Registering the `OTPAuthActivity` is either ways a must as this is what displays the web view. Without the `<data android:scheme="..." />` setting, the link on the OTP form will not work properly: **SAP Authenticator** will not be able to reopen your app and if you return to it manually the OTP form in the web view will not be informed of the passcode and will just wait there as if nothing happened.

> Of course, in the above case the user is still able to open **SAP Authenticator** and copy-paste the passcode to the OTP form manually.

### <a name="sum6"></a> Summary

* Two-Factor Authentication is an additional authentication step taking place after the first-stage authentication is successful. It presents an additional challenge to the user which can be answered using a one-time passcode generated by a properly configured **SAP Authenticator** application running on the same device.
* The mechanism 2FA uses is rather similar to how SAML2 support works in HttpC. It's also relying on an embedded web browser that opens the OTP form presenting the challenge.
* To enable, an OTP configuration provider must be implemented and registered with the common configurator. This will then enrich the configured `HttpConversationManager` with the ability to detect and orchestrate the 2FA flow.
* For the communication between **SAP Authenticator** and your app to work nicely, the custom URI scheme have to be set up for the `OTPAuthActivity` in your `AndroidManifest.xml`.
* The configuration is pretty much static for all kinds of SAP Mobile Platform server and Mobile Services on SAPcp endpoints.

## <a name="deep_dive"></a> Deep dive

This section is dedicated to readers wishing to explore the more complex features of the HttpConversation library. Reading this part is not crucial to get started. However, if you'd like to use more advanced features and/or extend the library in a way not covered by the previous sections, read on...

### <a name="conv_context"></a> The conversation context

Before jumping into writing your own (request, response, etc..) filters an important concept called the _conversation context_ should be understood first.

Technically, an HTTP conversation may be composed of multiple HTTP requests. This is especially true if any of the authentication methods are configured. If a server responds with a challenge, HttpC needs to handle it and then retry the request. Sometimes there might be multiple retries before the correct credentials are found and/or acquired.

This implies that a conversation itself has some internal state which spans over multiple requests. This is what the OAuth2 support uses internally to keep track of whether new or existing tokens were tried. Again, this is what the SAML2 support uses to prevent infinite loops if the server responds with a challenge even though the user has already logged in. The latter can happen if the server is not configured properly.

The conversation context is the object that maintains this state. Its lifecycle starts when the first request of a particular conversation is started and ends when the last request finishes.

The context is represented by `com.sap.smp.client.httpc.IContext` that is part of the `HttpConversation` library. This interface is accessible via `com.sap.smp.client.httpc.events.IBaseEvent` interface which has a couple of subinterfaces. Namely:

* `ISendEvent`
* `ITransmitEvent`
* `ICancellationEvent`
* `IReceiveEvent`

To access the context, call the `getConversationContext()` method.

> A similar getter, `getManagerContext()` is accessible which also returns an `IContext` but its lifecycle is tied to the enclosing `HttpConversationManager` instead.

These objects appear as arguments in many places in the HttpC API in filters, observers, listeners, etc..

The most important elements of the conversation context are accessible via its getters:

* `getName`: A unique name of a conversation (or of a manager). It is a rarely used feature and allows one to access the name of the `IHttpConversation` or the `HttpConversationManager` object that was set using `setName()`. Serves only identification purposes.
* `getStateMap`: This is a map of arbitrary key-value pairs. Whatever value you set in it will survive conversation restarts but will not live past the end of the conversation. For the manager context, it of course lives together with the `HttpConversationManager` instance it is bound to.

  This is what `HttpConvAuthFlows` itself makes use of internally to keep track of the internal state of various authentication methods.

It's recommended to read the API documentations of the above mentioned interfaces and methods. These cover the remaining details that should prove valuable before you start using this part of the API.

### <a name="custom_filters"></a> Writing custom filters

If you want to write your own filters then you must implement the appropriate interface in your own class, instantiate the filter and then register it with a `HttpConversationManager` instance using the corresponding `addFilter()` method.

The API documentations of the filter interfaces already clearly describe what contracts should be honored by the implementations. Nonetheless, some of the most important contracts are emphasized below:

* Make sure filter implementations are stateless, that is, they do not store **conversation-specific** information in member variables. The stateless nature is required to ensure overall thread-safety. Remember, a conversation manager might run multiple conversations in parallel, each of which will use your filter. If you keep internal state at an inappropriate location (i.e. in member vars) then you're going to have a problem real soon which is rather difficult to debug.
* If you are in need of associating some temporary information with the running conversation, use the conversation context. Via the various event objects, the context is available through one of the arguments of your filter method. Read [this section](#conv_context) if you have not done already.
* Avoid maintaining a reference to the `HttpConversationManager` to which your filter is registered. This is in general bad practice. However, you are free to instantiate and even configure **new** conversation managers **inside** your filters and use them to fire additional requests, if need be. This technique is called firing **orthogonal conversations** as part of another one. For this purpose however, you must ensure that your filter implementation waits 'till the orthogonal conversation completes.

Implementing a filter is the right thing to do if you'd like to enrich many different kinds of requests in your application with a certain functionality. It's kind of like a component that always takes part of request sending no matter the targeted endpoint.

Filters are used quite extensively by `HttpConvAuthFlows` to support the above documented auth. methods. In fact, all these methods are based purely on filters. There's no hidden API that `HttpConvAuthFlows` makes use of. You could implement your own SAML2 or OAuth2 request and response filters, if you want.

A typical example for a use case that fits filters is, for example, CSRF protection. Servers that protect against cross-site request forgery attacks usually ask for an extra CSRF token as part of every data modification (i.e. POST/PUT/DELETE/etc..) request. Depending on how this is implemented by the server, acquiring and sending this token is something that can be condensed into a filter implementation of your own. Then, all the requests you send using HttpC can make use of this mechanism transparently.

### <a name="executors"></a> Executors

A `HttpConversationManager` executes conversations in a separate thread by default. If you call the `start()` method of an `IHttpConversation` object then the internal threading settings of the manager determines how to spawn threads.

Under the hood, Java's built-in Executor framework is used. To override this, use the `setExecutor()` method of `HttpConversationManager` to configure a new executor. Via the platform-provided `java.util.concurrent.Executors` class, for example, you can make use of thread pools, if you like.

In rare cases one might want to execute conversations synchronously. To this end, HttpC provides the `com.sap.smp.client.httpc.utils.SynchronousExecutor` class that you can set as follows:

```java
HttpConversationManager manager = ...;
manager.setExecutor(SynchronousExecutor.inst);
```

> Note that `SynchronousExecutor` is actually an enum-based singleton.

This is handy if you don't want the `start()` method of an `IHttpConversation` object to return before the conversation completes.

Using the synchronous executor is not recommended if you start conversations directly on the UI thread. However, in the rare cases when you are implementing your own filters and are in need of orthogonal conversations (see [this section](#custom_filters) again for what these are) then synchronous executor can be a good choice. It will simplify your code a lot.

Imagine a request filter like this:

```java
...
import ...;

public class OrthogonalConversationRequestFilter implements IRequestFilter {
    private Context context;
    
    ...

    @Override
	public Object filter(ISendEvent event, IRequestFilterChain chain) {
        // Create a new conversation manager for the orthogonal request.
        HttpConversationManager manager = new HttpConversationManager(context);

        // Make it use a synchronous executor.
        manager.setExecutor(SynchronousExecutor.inst);

        // Fire an orthogonal request.
        manager.create(Uri.parse("http://example.com"));
            .addHeader(...)
            ...
            .start();

        return null;
    }
}
```

The above code will not return from the `OrthogonalConversationRequestFilter` 'till the orthogonal request send to `http://example.com` finishes. This is quite effective in case your custom authentication mechanism, for example, would need to access another HTTP endpoint **meanwhile another conversation is running**.

Don't forget that in this case the executor only applies to the internal conversation manager created within the filter. The conversation in scope of which the above code runs is orchestrated by another manager and consequently by another executor.

### <a name="oauth2_token_management"></a> OAuth2 token management

During OAuth2 authorization, `OAuth2ServerSupport` objects are used to orchestrate the flow and a `OAuth2TokenStorage` object is used to store the tokens.

However, one might need to manage the tokens **independently** of the running conversations. This is where the OAuth2 token manager, represented by the `com.sap.smp.client.httpc.authflows.oauth2.OAuth2TokenManager` class, comes into play.

The `CommonAuthFlowsConfigurator` has a getter called `getTokenManager()` that can be used to access the manager. By nature, the common configurator instance is a short-lived object that is used only for the time of configuring some `HttpConversationManager` instances. However, the token manager is something that can and should be kept around for a longer period.

The key thing is this: the token manager provides another entrance into dealing with OAuth2 access and refresh tokens. It can be used to delete some or all tokens, retrieve tokens for a particular URL, etc.. Here's how you use it:

1. Configure OAuth2, as instructed by [this section](#oauth2), on a common configurator instance.
1. Optionally, set a token storage on the same configurator instance.
1. Configure one or more `HttpConversationManager`s with it. These will have OAuth2 support enabled.
1. Get the token manager from the common configurator and put it away for later use.
1. At this point, the common configurator instance can be thrown away.

The token manager put away in step no. 4 will have access to the same server support objects and the same token storage as the conversations started by the managers configured in step no. 3.

Suppose you're accessing SAP Cloud Platform with our built-in server support `SAPOAuth2ServerSupport` and your application-specific token storage. If you then put away the token manager you can use it to drive a logout logic, for example. Simply call the appropriate method on the manager to remove the tokens for a particular URL. Internally, the manager will then make sure the same tokens are wiped which were acquired using `SAPOAuth2ServerSupport` and your storage implementation.

Most authentication and authorization methods are based on HTTP cookies. However, OAuth2 is using access and refresh tokens. The OAuth2 token manager therefore implements a similar access to such tokens as a cookie storage provides access to actual cookie values.

It is highly recommended to read the API documentation of the `getTokenManager()` getter of `CommonAuthFlowsConfigurator` and the `OAuth2TokenManager` class. It contains some additional details and insights you might find useful.

### <a name="webstrategies"> Web strategies

SAML2, OAuth2 and Two-Factor Authentication are features of HttpC which often rely on opening an embedded web browser. Normally, HttpC does this by displaying an instance of `android.webkit.WebView` in a separate activity.

The `HttpConvAuthFlows` library however allows for customizing these web-based flows. Customization here means that you can replace `WebView` with your own web view implementation.

All web-based flows are implemented via web strategies. Each of the three listed features have a corresponding web strategy interface in the `com.sap.smp.client.httpc.authflows.webstrategies` package. These are:

* `SAML2WebStrategy`
* `OAuth2WebStrategy`
* `OTPWebStrategy`

Albeit these are different interfaces, all of them follow the same philosophy. Whenever HttpC needs to display a web view it calls the appropriate strategy implementation. Each of these interfaces define a single main method which gets everything that's needed to display a web view as arguments and returns the results via a callback typically.

For example, `OAuth2WebStrategy` defines the `startOAuth2Authorization()` method that gets the OAuth2 authorization code grant configuration as argument and produces the authorization code via the callback.

Consequently, if you want to integrate your own web view solution you must implement the corresponding strategy and register it with the `WebStrategies` singleton via a resolver object. For each strategy interface a corresponding resolver interface belongs whose sole responsibility is to create and return a strategy for certain arguments. Example:

```java
import ...;

/**
 The custom OAuth2 web strategy that displays a custom web view.
 */
public class MyOAuth2Strategy implements OAuth2WebStrategy {
    @Override
    public void startOAuth2Authorization(Context context, OAuth2AuthCodeGrantConfig config, OAuth2AuthorizationCompleteCallback callback) {
        // Display the custom web view and make sure to call the callback
        // once the web view loads the redirect URI containing the authorization code.
        ...
    }
}

/**
 The resolver pertaining to 'MyOAuth2Strategy' making sure the custom strategy is used
 only with certain OAuth2 configurations and hosts.
 */
public class MyOAuth2StrategyResolver implements OAuth2WebStrategyResolver {
    @Override
    OAuth2WebStrategy resolve(String authorizationEndpointUrl) {
        if (authorizationEndpointUrl.indexOf("myservice.com") >=0) {
            return new MyOAuth2Strategy();
        }

        return null;
    }
}

...

public class UseItSomewhere {
    public void configure() {
        WebStrategies.inst.registerOAuth2Resolver(new MyOAuth2StrategyResolver());

        ...
    }
}

```

When calling the `configure()` method, our custom OAuth2 strategy resolver gets registered with the `WebStrategies` singleton. Looking at the implementation of `MyOAuth2StrategyResolver` we can see that this resolver will return a new instance of `MyOAuth2Strategy` if the authorization endpoint is hosted by `myservice.com`. In any other case null is returned.

> Of course, the above example is quite simplistic. Your implementation should be much more sophisticated, especially concerning the part that uses `indexOf()` in the example.

If the registered resolvers (no matter the type) are not able to produce custom strategy objects then `WebStrategies` always falls back to using the defaults. Therefore, the above code will use the custom web view implementation integrated via `MyOAuth2Strategy` only when an OAuth2 authorization code flow is to be started using an authorization endpoint that's hosted by `myservice.com`. For any other configuration the built-in `android.webkit.WebView`-based solution will be used.

> All of the default web strategy implementations use activities: SAML2 uses the `SAML2AuthActivity`, OAuth2 the `OAuth2AuthActivity` and OTP the `OTPAuthActivity`. The previous sections required that these activities be properly included in the `AndroidManifest.xml` of the application.
>
> If you customize the web strategies then you will need to define your own activities to handle the web-based flows. These built-in activities are not extendable.

Customizing web views is seldom required. In most cases the `WebView`-based solution works just fine. However, there are alternatives to the built-in web view on the Android platform. Furthermore, there are hybrid frameworks (like Apache Cordova) which are running most of the code in embedded browsers. Being able to plug in these solutions via custom web strategies makes HttpC even more flexible.

### <a name="cookies"></a> Cookie management

Proper cookie management is one of the trickiest things to get right on the Android platform. This is because `HttpURLConnection` (the class underneath each conversation) is using `java.net.CookieManager` whereas web-based flows (like SAML2, OTP, etc..) which use the default strategies and as such the `android.webkit.WebView` component use `android.webkit.CookieManager`.

These two cookie managers are separate. If you execute requests natively and acquire some authenticated cookies you will not be able to share them with the built-in web views and vice versa.

To get around this problem, HttpC ships the `com.sap.smp.client.httpc.SAPCookieManager` class which bridges the gap. Internally, this class is an extension of `java.net.CookieManager` which stores all its cookies in `android.webkit.CookieManager` by proxying all calls to it.

This mechanism is essential for SAML2 and OTP to work. If the cookie manager is replaced with the built-in default cookie manager then these authentication types will **not** work properly. In case of SAML2, for example, you'll see the authentication window pop up after which the original request will still fail even though you entered the correct credentials.

The reason is simple: the SAML2 web view acquires the cookies for the authenticated session but the native request underneath HttpC will not use them as it resides in a separate cookie storage.

If you run into problems with cookie management make sure to debug what the default cookie manager is. If it's an object other than `SAPCookieManager` then the web-based authentication flows with the default web strategies are not going to function.

> If you implement your own web strategies for SAML2 for example, you must keep the cookie problem foremost on your mind. The new web view implementation you are planning to use must be able to come to terms with the `HttpURLConnection`-based networking stack as to where the cookies are to be stored. Getting this right is can be quite complex.

By default, `HttpConversationManager` ensures that the cookie manager is indeed `SAPCookieManager`. Don't fiddle with this setting if you don't have to. If you do, make sure you understand the implications. It's best to read the API docs of these classes before touching anything.