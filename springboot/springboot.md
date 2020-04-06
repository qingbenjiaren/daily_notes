# Spring Boot Features

## Lazy Initialization

SpringApplication allows an application to be initialized lazily. When lazy initialization is enabled,
beans are created as they are needed rather than during application startup. As a result, enabling
lazy initialization can reduce the time that it takes your application to start. In a web application,
enabling lazy initialization will result in many web-related beans not being initialized until an
HTTP request is received.

A downside of lazy initialization is that it can delay the discovery of a problem with the application.
If a misconfigured bean is initialized lazily, a failure will no longer occur during startup and the
problem will only become apparent when the bean is initialized. Care must also be taken to ensure
that the JVM has sufficient memory to accommodate all of the application’s beans and not just those
that are initialized during startup. For these reasons, lazy initialization is not enabled by default
and it is recommended that fine-tuning of the JVM’s heap size is done before enabling lazy
initialization.

Lazy initialization can be enabled programatically using the lazyInitialization method on
SpringApplicationBuilder or the setLazyInitialization method on SpringApplication. Alternatively,
it can be enabled using the spring.main.lazy-initialization property as shown in the following
example:

```properties
spring.main.lazy-initialization=true
```

If you want to disable lazy initialization for certain beans while using lazy initialization for the rest of the application, you can explicitly set their lazy attribute to false using the **@Lazy(false)** annotation.

## Customizing the Banner

The banner that is printed on start up can be changed by adding a banner.txt file to your classpath
or by setting the **spring.banner.location** property to the location of such a file. If the file has an
encoding other than UTF-8, you can set spring.banner.charset. In addition to a text file, you can also
add a **banner.gif, banner.jpg, or banner.png** image file to your classpath or set the
**spring.banner.image.location** property. Images are converted into an ASCII art representation and
printed above any text banner.

Inside your banner.txt file, you can use any of the following placeholders:

| Variable               | Description                                                  |
| ---------------------- | ------------------------------------------------------------ |
| ${application.version} | The version number of your application, as<br/>declared in MANIFEST.MF. For example,<br/>Implementation-Version: 1.0 is printed as 1.0. |
|                        |                                                              |

