# WebView overview

[WebView](https://developer.android.com/reference/android/webkit/WebView) is a View that displays web pages. WebView objects display web content as part of an activity layout, but lack some of the features of fully-developed browsers.

# Security issues

## addJavascriptInterface

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

## setWebContentsDebuggingEnabled

[setWebContentsDebuggingEnabled](https://developer.android.com/reference/android/webkit/WebView.html#setWebContentsDebuggingEnabled%28boolean%29) enables debugging of web contents (HTML/CSS/JavaScript) loaded into any WebViews of an application. This flag is used in order to facilitate debugging of web layouts and JavaScript code running inside WebViews.

{% hint style="info" %}
`WebContentsDebugging` is not affected by the state of the `debuggable` flag in the application's manifest. 
{% endhint %}

Enabling this flag allows you to execute arbitrary JavaScript code within any WebViews of an application.

References:
- [Remote debugging WebViews](https://developer.chrome.com/docs/devtools/remote-debugging/webviews/)
