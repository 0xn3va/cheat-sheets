# Deep linking overview

Deep linking is a mechanism that processes specific types of links and sends users directly to an app, for example, to a particular activity. Android allows developers to create two types of links:
- Deep link
- Android App Link

## Deep link

Deep link is a URL that takes users directly to specific content in an app. For example, the `example://myapp` deep link can be used to start `MainActivity`.

Deep links are set up by adding [intent filters](/Mobile%20Application/Android/Intent%20Vulnerabilities/README.md#intent-filter) and users are driven to the right activity based on the data extracted from incoming intents. Therefore, several apps are able to handle the same deep links (intents). In this case, users might not go directly to a particular app and they need to select an app, see the [Intent resolution](/Mobile%20Application/Android/Intent%20Vulnerabilities/README.md#intent-resolution) section.

The following XML snippet shows an example of an intent filter in a manifest for deep linking, where the `example://myapp` URI is resolved to the `MainActivity`:

```xml
<activity android:name="MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <!-- Accepts URIs that begin with "example://myapp -->
        <data android:scheme="example"
              android:host="myapp" />
    </intent-filter>
</activity>
```

## Android app link

Android App Links is a special type of deep link that allows website URLs to immediately open the corresponding content in an app (without requiring the user to select the app). If the user doesn't want the app to be the default handler, they can override this behavior from their device's system settings.

Android App Links are set up by adding [intent filters](/Mobile%20Application/Android/Intent%20Vulnerabilities/README.md#intent-filter) that open an app content using `http/https` URLs and verifying that an app is allowed to open these website URLs. The verifying is required the following steps:
- Request [automatic app link verification](https://developer.android.com/training/app-links/verify-site-associations#request-verify) in a manifest. This signals to the Android system that it should verify an app belongs to the URL domain used in intent filters.
- Declare the relationship between a website and intent filters by hosting a [Digital Asset Links](https://developers.google.com/digital-asset-links/v1/getting-started) JSON file at the following location:

    ```http
    https://domain.name/.well-known/assetlinks.json
    ```

If the system successfully verifies that an app is allowed to open a URL, the system automatically routes this URL intents to the app.

The following XML snippet shows an example of an intent filter in a manifest for app linking, where the `https://example.com` URI is resolved to the `MainActivity`:

```xml
<activity android:name="MainActivity">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <!-- Accepts URIs that begin with "https://example.com/ -->
        <data android:scheme="https" 
              android:host="example.com" />
    </intent-filter>
</activity>
```

## The difference between deep links and app links

| # | Deep links | App links |
| --- | --- | --- |
| Intent URL scheme | `http`, `https`, or a custom scheme | Requires `http` or `https` |
| Intent action | Any action | Requires `android.intent.action.VIEW` |
| Intent category | Any category | Requires `android.intent.category.BROWSABLE` and `android.intent.category.DEFAULT` |
| Link verification | None | Requires a [Digital Asset Links](https://developers.google.com/digital-asset-links/v1/getting-started) file served on a website with HTTPS |
| User experience | May show a disambiguation dialog for the user to select which app to open the link | No dialog; an app opens to handle website links |
| Compatibility | All Android versions | Android 6.0 and higher |

# Security issues

## Access arbitrary components

An app can implement its own intent parser to handle deeplinks using JSON objects, strings or byte arrays that may expand Serializable and Parcelable objects and allow to set insecure flags.

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/android-application/intent-vulnerabilities#access-arbitrary-components" %}

For example, the following deeplink parser converts a byte array to Parcel and read an intent from it:

```java
Uri deeplinkUri = getIntent().getData();
if (deeplinkUri.toString().startsWith("deeplink://handle/")) {
    byte[] handle = Base64.decode(deeplinkUri.getQueryParameter("param"), 0);
    Parcel parcel = Parcel.obtain();
    parcel.unmarshall(handle, 0, handle.length);
    startActivity((Intent) parcel.readParcelable(getClassLoader()));
}
```

## App link misconfiguration

Deeplinks as well as app links can use the `https` scheme, and developers can configure an intent filter for deeplinks instead of app links. As a result, you can create an app that will handle the same deeplinks and intercept intents:

```xml
<intent-filter android:priority="999">
	<action android:name="android.intent.action.VIEW" />
	<category android:name="android.intent.category.DEFAULT" />
	<category android:name="android.intent.category.BROWSABLE" />
	<data android:scheme="https" />
	<data android:host="myapp.link" />
</intent-filter>
```

References:
- [Report: Account takeover intercepting magic link for Arrive app](https://hackerone.com/reports/855618)

## Arbitrary URL opening in WebView

If an app opens URL in WebView based on parameters from a deeplink you can try to bypass a URL validation and open arbitrary URLs. This can be used to execute arbitrary JavaScript, steal sensitive data, access arbitrary components and chain with other weaknesses.

References:
- [WebView Vulnerabilities: Bypass URL validation](/Mobile%20Application/Android/WebView%20Vulnerabilities/README.md#bypass-url-validation)
- [Report: [Grab Android/iOS] Insecure deeplink leads to sensitive information disclosure](https://hackerone.com/reports/401793)
- [Writeup: Breaking The Facebook For Android Application](https://ash-king.co.uk/blog/facebook-bug-bounty-09-18)
- [Writeup: When Equal is Not, Another WebView Takeover Story](https://valsamaras.medium.com/when-equal-is-not-another-webview-takeover-story-730be8d6e202)

## Bypass local authentication

Apps can handle deeplinks before local authentication (passcode/biometrics) and sometimes this can lead to a direct user being pushed into an activity without local authentication. This may require you to simply follow a deeplink, or abuse parameters / functionality, trying to get exceptional conditions, such as failing validation or interrupting a flow in the middle.

References:
- [Report: Bypass of biometrics security functionality is possible in Android application (com.shopify.mobile)](https://hackerone.com/reports/637194)

## Insecure parameter handling

Deeplinks allows users to provide parameters to an application that can be used as parameters when performing local actions, requests to API, etc. Therefore, if these parameters are not properly validated, an attacker can use them for profit (up to RCE).

For instance, suppose an application opens local files based on http/https URLs by the next flow:
1. A user send the link `https://website.com/file.pdf`
2. An application parses the URL and retrieves the URL path: `file.pdf`
3. An application joins a hard-coded temp folder with `file.pdf`: `/data/data/com.vulnerable-app/temp-files/file.pdf`
4. An application downloads the PDF file from `https://website.com/file.pdf` and save them to `/data/data/com.vulnerable-app/temp-files/file.pdf`
5. An application opens downloaded file for a user

In such case, an attacker is able to rewrite an arbitrary file within the package using path traversal: `https://website.com/x/..%2F..%2Fdatabases/secret.db`.

References:
- [Writeup: RCE IN ADOBE ACROBAT READER FOR ANDROID(CVE-2021-40724)](https://hulkvision.github.io/blog/post1/)

## Perform unsafe actions without confirmation

Sometimes apps allow users to perform unsafe actions through deeplinks, such as modifying data, doing a call, buying a subscription and etc. If these actions do not require additional confirmation from a user, you can perform a CSRF-like attack.

For example, if an app allows authenticated users to change their email through the `myapp://user?email=<email>` deeplink, you can change the victim's email to your own by making them visit the following page:

```html
<!DOCTYPE html>
<html>
    <script>location.href = "myapp://user?email=attacker@attacker-website.com";</script>
</html>
```

References:
- [Report: Periscope android app deeplink leads to CSRF in follow action](https://hackerone.com/reports/583987)
- [Report: CSRF when unlocking lenses leads to lenses being forcefully installed without user interaction](https://hackerone.com/reports/1085336)

# References

- [Android Developers: Handling Android App Links](https://developer.android.com/training/app-links)
- [Oversecured: Android Access to app protected components](https://blog.oversecured.com/Android-Access-to-app-protected-components/)
