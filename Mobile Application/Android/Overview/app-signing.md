Application signing allows developers to identify the author of the application and to update their application without creating complicated interfaces and permissions. Every application that is run on the Android platform must be signed by the developer. Applications that attempt to install without being signed will be rejected by either Google Play or the package installer on the Android device.

# APK signing schemes

Android supports four application signing schemes:
- **v1 scheme** based on JAR signing.
- **v2 scheme** APK Signature Scheme v2, which was introduced in Android 7.0.
- **v3 scheme** APK Signature Scheme v3, which was introduced in Android 9.
- **v4 scheme** APK Signature Scheme v3, which was introduced in Android 11.

For backwards compatibility, an APK can be signed with multiple signature schemes in order to make the app run on both newer and older SDK versions.

## JAR signing (v1 scheme)

APK signing has been a part of Android from the beginning. It is based on [signed JAR](https://docs.oracle.com/javase/8/docs/technotes/guides/jar/jar.html#Signed_JAR_File).

v1 signatures do not protect some parts of the APK, such as ZIP metadata. The APK verifier needs to process lots of untrusted (not yet verified) data structures and then discard data not covered by the signatures. Moreover, the APK verifier must uncompress all compressed entries, consuming more time and memory.

## APK Signature Scheme v2

APK Signature Scheme v2 is a whole-file signature scheme that increases verification speed and strengthens integrity guarantees by detecting any changes to the protected parts of the APK.

Signing using APK Signature Scheme v2 inserts an APK Signing Block into the APK file immediately before the ZIP Central Directory section. Inside the APK Signing Block, v2 signatures and signer identity information are stored in an APK Signature Scheme v2 Block. See more in the [documentation](https://source.android.com/security/apksigning/v2).

## APK Signature Scheme v3

Android 9 supports APK key rotation, which gives apps the ability to change their signing key as part of an APK update. To make rotation practical, APKs must indicate levels of trust between the new and old signing key. v3 adds information about the supported SDK versions and a proof-of-rotation struct to the APK signing block. See more in the [documentation](https://source.android.com/security/apksigning/v3).

## APK Signature Scheme v4

APK Signature Scheme v4 is a streaming-compatible signing scheme. v4 is based on the Merkle hash tree calculated over all bytes of the APK. It follows the structure of the [fs-verity](https://source.android.com/security/features/apk-verity) hash tree exactly. Android 11 stores the signature in a separate file, `<apk name>.apk.idsig`. The v4 signature requires a complementary v2 or v3 signature. See more in the [documentation](https://source.android.com/security/apksigning/v4).

# References

- [Android Open Source Project: Application Signing](https://source.android.com/security/apksigning)
