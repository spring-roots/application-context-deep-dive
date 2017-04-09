# Application Context Deep Dive

## Audience

You've worked on at least a couple of Spring applications.

## Objectives

Achieve fluency in constructing, configuring, and executing a Spring Application Context:

- essential of the parts of an Application Context;
- the role of each part;
- which parts are contributed by which flavor of Application Context;
- how Spring Beans are added to the Bean Factory;
- how properties are loaded (and how/why various sources are ordered).


## How to use this guide

1. **Enjoy!**.  Think of Spring Framework as a tinker toy: let poke and explore just for the fun of it!  You'll be amazed at how much more you'll retain when you're actually enjoying the experience.
1. **GO SLOW**.  The clearer your understanding of these fundamental bits, the stronger your foundation.  The stronger your foundation, the more solidly you'll be able to grow your knowledge of more advanced concepts.
1. **Use Your Tools**.  The IDE's debugger and code exploration features are powerful ways to bring shape to the code you are reading; it's worth taking the time to learn them.
   - IntelliJ:
     - **"Type Hierarchy" (Ctrl+H)** — pick a core interface (e.g. `ApplicationContext`) and explore the tree of implementors.
     - **"Download sources..."** when opening a class from a dependency, it's always a good idea to grab the source code.  Spring source code is pleasant to read.
     - **"Quick Documentation" (F1)** — when viewing a class or method, hit F1 to see a rendering of its JavaDoc.
       - out of the box, IntelliJ displays this view as a kind of "pop-up".  You might it useful to: make it a tool (click the pin in the top-right corner of the window), and and dock it on the bottom (or side).


## Part 1: Taking Stock

**Objective:** explore the innards of the simplest Application Context, breaking it down into its constituent parts and looking for how they work together.

1. Create a Java program that instantiates the simplest concrete type of Application Context available (i.e. `StaticApplicationContext`); 
   
   ```java
    StaticApplicationContext simplestAppCtx = new StaticApplicationContext();
    simplestAppCtx.refresh();
    simplestAppCtx.close();
   ```
   If you're using Gradle, here's your minimal `build.gradle`:
   
   ```groovy
   apply plugin: 'java'

   repositories {
       mavenCentral()
   }

   dependencies {
       compile 'org.springframework:spring-context:4.3.7.RELEASE'
   }
   ```
1. Run it in the debugger, set a breakpoint _after_ the `refresh()` call and explore the contents of the ApplicationContext object.

   Among all the many objects within the Application Context, let's find and examine each:
   - there are six (6) singleton beans in this plain-vanilla application
     context.  Can you find them within the Application Context?
     - _hint:_ you're looking for a Map between the bean names and their 
       actual instances.
   - what are they?
   - what does each of them do?
     - Does that square with the JavaDoc for [`ApplicationContext`](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/ApplicationContext.html)?
1. Turn up JDK logging to `FINEST` for `org.springframework` ([enabling logging](https://github.com/jtigger/memorex/tree/master/java/logging#enabling-jdk-logging)).
   - You might want to decrease the font size of the console and/or increase the line spacing to make the output more readable.
1. Let's use the logging output to tell the story of how an Application Context initializes.
   - Run the application again.
   - Start from the top of the log output, for each line attempt to translate to "english".
1. Let's add an Application Context for one that knows how to detail with annotation-based config, `AnnotationConfigApplicationContext`:

   ```java
    StaticApplicationContext simplestAppCtx = new StaticApplicationContext();
    simplestAppCtx.refresh();
 
    AnnotationConfigApplicationContext annotationAppCtx = new AnnotationConfigApplicationContext(SimpleHelloWorld.class);
    annotationAppCtx.refresh();
 
    simplestAppCtx.close();
    annotationAppCtx.close();
   ```
   Let's see what _more_ the "Annotation Config" part of the `AnnotationConfigApplicationContext` brings in:
   - now there are 12 singleton beans (twice as many!)... what are the new ones?
     - does their bean name match their type?
   - of each of these new singletons, what do they do?
   - of the beans you saw in the prior step (`environment`, `messageSource`, ...), do any change in their type or configuration?
   - what additional steps does _this_ flavor of Application Context show up in the logs?

## Part 2: Registering Beans

1. Either revert back to the first step of Part 1, above, or create a new Java program that instantiates a `StaticApplicationContext`, calls some lifecycle events and closes:
   
   ```java
    StaticApplicationContext appCtx = new StaticApplicationContext();
    appCtx.refresh();
    appCtx.close();
   ```
1. **Explicit special bean.** 

   Define a class that implements `ApplicationListener` and outputs events that are triggered:
   
   ```java
    class Peeker implements ApplicationListener<ApplicationEvent> {
      private static final Logger log = Logger.getLogger(Peeker.class.getName());

      @Override
      public void onApplicationEvent(ApplicationEvent event) {
        log.info("Received Event: " + event);
      }
    }
   ```
   Explicitly add `Peeker` as an application listener using `ApplicationContext#addApplicationListener()`.
   
   ```java
    appCtx.addApplicationListener(new Peeker());
   ```
   Re-run the app and see the listener's output.
1. **Implicit special bean.** 

   Remove the explicit add from the previous step.  Instead, add `Peeker` as a singleton:

   ```java
    appCtx.registerSingleton("peeker", Peeker.class);
   ```
   Compare the output from the previous step.  What _other_ kinds of beans are specially handled?  (hint: look for invocations of `AbstractApplicationContext#getBeanNamesForType()`).
1. **By default beans are singleton.** 

   Replace the `StaticApplicationContext` with `AnnotationConfigApplicationContext` and register `Peeker` with `register()` instead of `registerSingleton()`.
   
   ```java
    AnnotationConfigApplicationContext appCtx = new AnnotationConfigApplicationContext();
    appCtx.register(Peeker.class);
   ```
   Re-run the application and note what changes (and what doesn't).
1. **Explicit configuration (declaring beans).** 

   Add another class to house the configuration:
   
   ```java
    @Configuration
    class HelloWorldConfig {
      @Bean
      public static Peeker peeker() {
        return new Peeker();
      }
    }
   ```
   ... and instead of registering `Peeker` explicitly, register this configuration class, instead:
   
   ```java
    appCtx.register(HelloWorldConfig.class);
   ```
   Run the application in the debugger again.
   - note what new singletons appear in the BeanFactory.
   - is a class annotated with `@Configuration` a Spring bean?
   - looking at the logging output, trace through how the `Peeker` bean was added the Application Context:
     - which component contributes the BeanDefinition for `Peeker`?
     - which component actually creates an instance of `Peeker`?
1. **Scanned configuration.**

   Remove the explicit registration of the configuration class, `HelloWorldConfig`.  Instead, use the scanning feature of the `AnnotationConfigApplicationContext`:
   
   ```java
    appCtx.scan("hello.simple");
   ```
   ... re-run the application and examine the logging output:
   - was `Peeker` included or ignored up by `ClassPathBeanDefinitionScanner`?
   - which bean was included by `ClassPathBeanDefinitionScanner`?
1. **Component scan.**
   1. move the `@Bean` method from `HelloWorldConfig` into `SimpleHelloWorld`
     and delete the `HelloWorldConfig` class.
   1. specify `SimpleHelloWorld` as the root configuration class by specifying
     it as the parameter the constructor for `AnnotationConfigApplicationContext`.
   1. annotate `SimpleHelloWorld` as a `@Configuration` class.
   1. annotate `Peeker` as a `@Component` class.
   
   `SimpleHelloWorld` should look something like this:

   ```java
    @Configuration
    @ComponentScan("hello.simple")
    public class SimpleHelloWorld {
      public static void main(String[] args) {
        AnnotationConfigApplicationContext appCtx = new AnnotationConfigApplicationContext(SimpleHelloWorld.class);
        appCtx.refresh();
        appCtx.close();
      }
    }
   ```
   
   `Peeker` should look something like this:
   
   ```java
    @Component
    class Peeker implements ApplicationListener<ApplicationEvent> {
      private static final Logger log = Logger.getLogger(Peeker.class.getName());

      @Override
      public void onApplicationEvent(ApplicationEvent event) {
        log.info("Received Event: " + event);
      }
    }
   ```
   ... and rerun the application.  The intent, here, is that this is equivalent to the previous state.
   - is `@Configuration` on `SimpleHelloWorld` strictly required?
   - is the `"hello.simple"` value of `@ComponentScan` strictly required?
   

## Part 3. Spring Boot-enhanced 


1. **Spring Boot-supplied Application Context (minimally-activated)**.
  
   Replace the direct dependency on `spring-context` with the Spring Boot base artifact (notice: not the starter).
   
   ```groovy
    dependencies {
      compile 'org.springframework.boot:spring-boot:1.5.2.RELEASE'
    }
   ```
   
   Either explore the dependencies using the Gradle plugin or run gradle on the command-line to view them:
   
   ```bash
   $ gradle dependencies
   ```
   
   Notice that this artifact depends on `spring-context` as we have been thus far and adds the minimal set of Spring Boot.
   
   Replace instantiating the Application Context directly with the convenience method from Spring Boot:
   
   ```java
    ConfigurableApplicationContext appCtx = SpringApplication.run(SimpleHelloWorld.class);
   ```
   
   and re-run the application and/or debug it.
   
   - what exact concrete type of Application Context does Spring Boot use?
   - does `SpringApplication.run()` refresh the context?
     - how does that square with the behaviors described in the [`SpringApplication` JavaDoc](http://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/SpringApplication.html)?
   - there are now 18 singletons in the bean factory
     - what are the three (3) new ones?
     - what does each contribute?
   - Spring Boot [registers](http://docs.spring.io/autorepo/docs/spring-boot/current/reference/htmlsingle/#howto-customize-the-environment-or-application-context) a `ConfigFileApplicationListener`.
     - what does this component (as an `ApplicationListener`) do?
       - how does that square with the behavior described in [Spring Boot Reference — 24.3 Application property files](http://docs.spring.io/autorepo/docs/spring-boot/current/reference/htmlsingle/#boot-features-external-config-application-property-files)?
     - it defines an inner class of type `BeanFactoryPostProcessor` that it registers when the `ApplicationPreparedEvent` is triggered (see [Spring Boot Reference — 23.5 Application events and listeners](http://docs.spring.io/autorepo/docs/spring-boot/current/reference/htmlsingle/#boot-features-application-events-and-listeners)).
       - what does that inner class do?
       - How does that square with the order of precedence of config sources described in the [Spring Boot docs](http://docs.spring.io/autorepo/docs/spring-boot/current/reference/html/boot-features-external-config.html)?
   - Spring Boot also includes a number of other `ApplicationListener`s; what do these do?
     - [`LoggingApplicationListener`](http://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/logging/LoggingApplicationListener.html) (see also [Spring Boot Reference — 26. Logging](http://docs.spring.io/autorepo/docs/spring-boot/current/reference/htmlsingle/#boot-features-logging))
     - [`ClasspathLoggingApplicationListener`](https://github.com/spring-projects/spring-boot/blob/master/spring-boot/src/main/java/org/springframework/boot/context/logging/ClasspathLoggingApplicationListener.java)

1. **Fully Activated Spring Boot Application Context**.

   Replace the direct dependency on `spring-boot` with the Spring Boot Starter.
   
   ```groovy
    dependencies {
      compile 'org.springframework.boot:spring-boot-starter:1.5.2.RELEASE'
    }
   ```
   
   Either explore the dependencies using the Gradle plugin or run gradle on the command-line to view them:
   
   ```bash
   $ gradle dependencies
   ```
   
   - What new dependencies are included?  Here is some helpful background to determine what these do:
     - [Spring Boot Reference — 24.6. Using YAML instead of Properties](http://docs.spring.io/autorepo/docs/spring-boot/current/reference/htmlsingle/#boot-features-external-config-yaml)
     - [Spring Boot Reference — 26. Logging](http://docs.spring.io/autorepo/docs/spring-boot/current/reference/htmlsingle/#boot-features-logging)
     - Auto-configuration:
        - [Spring Boot Reference — 16. Auto-configuration](http://docs.spring.io/autorepo/docs/spring-boot/current/reference/htmlsingle/#using-boot-auto-configuration)
        - [It's a kind of magic: under the covers of Spring Boot by Stéphane Nicoll & Andy Wilkinson — Basic auto-configuration (video)](https://www.youtube.com/watch?v=uof5h-j0IeE&feature=youtu.be&t=758)
   
   Add a couple of arguments to the initialization:
   
   ```java
   ConfigurableApplicationContext appCtx = SpringApplication.run(SimpleHelloWorld.class, "--some-opt-arg=happy", "some-non-opt-arg");
   ```
   
   re-run the application and/or debug it:
   
   - did you notice the logging output change?
     - What mechanism in Spring Boot was turned on by merely including the starter?
     - Which logging system is active now?
   - there are now 20 singletons in the bean factory
     - what are the two (2) new ones?
     - what does each contribute?  Here are some references for background:
       - Application arguments:
         - [Spring Boot Reference — 23.7. Accessing application arguments](http://docs.spring.io/autorepo/docs/spring-boot/current/reference/htmlsingle/#boot-features-application-arguments)
          - [SimpleCommandLinePropertySource](http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/env/SimpleCommandLinePropertySource.html)
        - Auto-configuration report:
          - [It's a kind of magic: under the covers of Spring Boot by Stéphane Nicoll & Andy Wilkinson — Auto-configuration report (video)](https://www.youtube.com/watch?v=uof5h-j0IeE&feature=youtu.be&t=2052)

   
## Quiz

1. Draw a diagram (boxes with names) of the Application Context.
   1. Include all the things!
      - can you come up with a 30 second "Elevator Pitch" for each?
   1. Color code where all the parts come from:
      - Spring Framework
      - Spring Boot
1. What's the difference between a "Spring Bean" and a "Component"?
1. What are the sources of properties within the Environment?
