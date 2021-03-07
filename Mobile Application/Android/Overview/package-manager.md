Package Manager is the API that actually manages the installation, removal, and updating of applications.

# APK installation process

Android app installation process is as follows:

![](img/apk-installation.png)

1. When you install an APK file, Package Manager parses the APK and displays confirmation. When you press the "OK" button, Package Manager calls `PackageInstaller` for interactively installing the package.
2. PackageInstaller provides a user interface to manage the package. PackageInstaller calls `InstallAppProgress` activity to receive instructions from the user.
3. InstallAppProgress asks `Package Manager Service` to install the package.
4. Package Manager Service install the package using the `installd` daemon, which receives a request via `/dev/socket/installd` socket and executes a series of steps to install APK with root permission. The whole installation process looks like this:
    - Add a package to the installation queue.
    - Determine the appropriate location of the package installation (`/system/app/` for system apps, `/data/app/` for third-part apps).
    - Copy the APK to a given directory.
    - Determine the UID of the application.
    - Request the `installd` daemon process. 
    - Create the application directory in `/data/data/` and set permissions.
    - Extracting the DEX code to the cache directory `/data/dalvik-cache/`.
    - Update the `packages.list` and `packages.xml` files.
    - Broadcast to the system with `Intent.ACTION_PACKAGE_ADDED` or `Intent.ACTION_PACKAGE_REPLACED`
5. The application was installed successfully.

# Data storage

Package Manager stores information about the application in three files located in `/data/system`:
- `packages.xml` contains list of permissions and packages:
- `packages.list` is a simple text file containing package name, user id, flag and data directory.
- `packages-stoped.xml` contains the package list which has stopped state (stopped state applications cannot receive any broadcast).

## packages.list

The `packages.list` file contains something like this for all applications installed on the system:

```
com.android.packageinstaller 10025 0 /data/data/com.android.packageinstaller platform 1028,3003,2001
```

The row is divided into 6 columns, which contain information about the application:
- `com.android.packageinstaller` is the package name of the application, which is the content of the `package` attribute of the `<manifest>` element in the `AndroidManifest.xml`.
- `10025` is the `userId` used by the application; if the `android:sharedUserId` attribute is not specified in `AndroidManifext.xml`, the system will automatically assign a `userId` to the application.
- `0` is whether the application is in debug mode, specified by `android:debuggable` in `AndroidManifext.xml`.
- `/data/data/com.android.packageinstaller` is the data storage path of the application, which is usually a folder like `/data/data/<package_name>`.
- `platform` is the `seinfo` information of the application; this should be related to the SEAndroid mechanism (it is not too clear, its value seems to have `platform` and `default`).
- `1028,3003,2001` is the user group to which the application belongs; if the application does not belong to any group, the value here is `None`.

## packages.xml

The `packages.xml` file has the following structure:

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<packages>
    <version ... />
    <version ... />
 
    <permissions>
        <item name="xxxx" package="xxx" protection="xx" />
        ... ...
    </permissions>
 
    <package xxx>
        ...
    </package>
    ...
 
    <shared-user xxx>
        ...
    </shared-user>
    ...
 
    <keyset-settings version="1">
        ...
    </keyset-settings>
</packages>
```

The main information in the `packages.xml` file is divided into the following sections:
- **Permission block** contains information about all defined permissions in the system.
- **Package block** contains details of all installed applications in the system.
- **Shared-user block** contains all system-defined shared user information.
- **Keyset-settings block** contains the public key information of the installed app signature.

### Permission block

```xml
<permissions>
    <item name="android.permission.REAL_GET_TASKS" package="android" protection="18" />
    <item name="android.permission.REMOTE_AUDIO_PLAYBACK" package="android" protection="2" />
    ...
</permissions>
```

It defines all the declared permission information in the system, and each item block represents a permission:
- **Name** indicates the name of the permission.
- **Package** indicates the package that declares the permission
- **Protection** indicates the level of the permission, such as normal, dangerous, etc.

### Package block

The package block contains the details of each application.

```xml
<package name="com.tencent.qqmusictv" codePath="/data/app/qqmusictv" nativeLibraryPath="/data/app/qqmusictv/lib" primaryCpuAbi="armeabi" publicFlags="941112933" privateFlags="0" ft="15f00a383c8" it="15f00a383c8" ut="15f00a383c8" version="134" userId="10044">
    <sigs count="1">
        <cert index="6" key="30820247308201b0a003020..." />
    </sigs>
    <perms>
        <item name="android.permission.WRITE_SETTINGS" granted="true" flags="0" />
        ...
    </perms>
    <proper-signing-keyset identifier="7" />
</package>
```

The attributes of the `<package>` element:
- **name** indicates the package name.
- **codePath** indicates the location where this APK file is stored.
- **nativeLibraryPath** represents the location of the `xxx.so` inventory used by the application.
- **primaryCpuAbi** indicates which abi architecture the application runs on.
- **publicFlags and privateFlags** are generated based on the settings in `AndroidManifest.xml`, for example: `android:multiarch`.
- **ft** indicates when the APK file was last changed.
- **it** indicates when the application was first installed.
- **ut** indicates when the application was last updated.
- **version** is information about the version number of the application, which is the `android:versionCode` configured in `AndroidManifest.xml`.
- **userId** is the user id assigned to the application; if there is a `shareUserId`, the `SharedUserId` appears here.

The attributes of the `<sigs>` element:
- **count** indicates how many signatures the application has (some applications may be signed by multiple certificates).

The attributes of the `<cert>` element:
- **index** indicates the serial number of the certificate used by the application; when the system finds a new certificate, the number will be incremented by 1.
- **key** is the ASCII code value of the certificate content used by the application; if the Package Manager Service, when scanning the APK, detects that a previously known certificate is used, then the `key` value will be absent; packages with the same index, indicating that they are using the same signature.

The `<perms>` block is the permissions that the application has. For normal applications, these permissions are written in `AndroidManifest.xml`. For those applications that use the same `userId`, the permissions here are the sum of the permissions of all applications that use the same `userId`. `granted` indicates whether this permission has been allowed.

The `identifier` attribute of the `<proper-signing-keyset>` is the value of the identifier in the [keysets](#keyset-settings-block). It is used to indicate which public key the application uses.

### Shared-user block

```xml
<shared-user name="android.uid.system" userId="1000">
    <sigs count="1">
        <cert index="1" />
    </sigs>
    <perms>
        <item name="android.permission.REAL_GET_TASKS" granted="true" flags="0" />
        <item name="android.permission.ACCESS_CACHE_FILESYSTEM" granted="true" flags="0" />
        ... ...
        <item name="android.permission.DELETE_PACKAGES" granted="true" flags="0" />
    </perms>
</shared-user>
```

The attributes of the `<shared-user>` element:
- **name** indicates the name of this shared-user.
- **userId** indicates the number of the user in the system.

The meaning of the `<sigs>` element is the same here and in the package block.

`<perms>` indicates the permissions this user has. When the APK file is scanned at startup, the permissions of all applications using the same `userId` are collected and placed here. And finally, these permissions will be sent to those applications that use the same `userId`. The end result is that the application with the same `userId` in the system has the same permissions.

### Keyset-settings block

```xml
<keyset-settings version="1">
    <keys>
        <public-key identifier="1" value="MIIBIjANBgkqhki..." />
        ...
    </keys>
    <keysets>
        <keyset identifier="1">
            <key-id identifier="1" />
        </keyset>
        ...
    </keysets>
    <lastIssuedKeyId value="9" />
    <lastIssuedKeySetId value="9" />
</keyset-settings>
```

- The `<keyset-settings>` block collects public key information for all application signatures and is associated with the information in the package block.
- The value of the `value` attribute of the `<public-key>` in the `<keys>` block is the public key extracted from the signature file in the APK package.
- The `<keysets>` block contains a lot of keysets. Each keyset has a number with an identifier. The `identifier` attribute of the `<key-id>` corresponds to the value of the `<public-key>` identifier in the `<keys>` above.
- `lastIssuedKeyId` and `lastIssuedKeySetId` represent the set number to which the most recent public key was taken.

# References

- [In Depth: Android Package Manager and Package Installer](https://dzone.com/articles/depth-android-package-manager)
- [Android development system packages file parsing](https://www.programmersought.com/article/3817615344/)
