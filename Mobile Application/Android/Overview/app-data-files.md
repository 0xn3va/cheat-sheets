Android uses a file system that's similar to disk-based file systems on other platforms. The system provides several options for saving app data:
- **App-specific storage** - internal storage to saving sensitive information that other apps should not be able to access.
- **Shared storage** - storage for files that the app shares with other apps, including media, documents, and other files.
- **Preferences** - storage for private, primitive data in key-value pairs.
- **Databases** - storage for structured data in a private database using the Room persistence library.

|  | Type of content | Access method | Permissions needed |
| --- | --- | --- | --- |
| [App-specific files](https://developer.android.com/training/data-storage/app-specific) | App-specific files | <p>From internal storage, [getFilesDir()](https://developer.android.com/reference/android/content/Context#getFilesDir%28%29) and [getCacheDir()](https://developer.android.com/reference/android/content/Context#getCacheDir%28%29)</p><p>From external storage, [getExternalFilesDir()](https://developer.android.com/reference/android/content/Context#getExternalFilesDir%28java.lang.String%29) and [getExternalCacheDir()](https://developer.android.com/reference/android/content/Context#getExternalCacheDir%28%29)</p> | <p>Never needed for internal storage</p><p>Not needed for external storage when your app is used on devices that run Android 4.4 (API level 19) or higher</p> |
| [Media](https://developer.android.com/training/data-storage/shared/media) | Shareable media files (images, audio files, videos) | [MediaStore](https://developer.android.com/reference/android/provider/MediaStore) API | <p>**READ_EXTERNAL_STORAGE** when accessing other apps' files on Android 11 (API level 30) or higher</p><p>**READ_EXTERNAL_STORAGE** or **WRITE_EXTERNAL_STORAGE** when accessing other apps' files on Android 10 (API level 29)</p><p>Permissions are required for all files on Android 9 (API level 28) or lower</p> |
| [Documents and other files](https://developer.android.com/training/data-storage/shared/documents-files) | Other types of shareable content, including downloaded files | Storage Access Framework | None |
| [App preferences](https://developer.android.com/training/data-storage/shared-preferences) | Key-value pairs | [Jetpack Preferences](https://developer.android.com/guide/topics/ui/settings/use-saved-values) library | None |
| Database | Structured data | [Room](https://developer.android.com/training/data-storage/room) persistence library | None |

# References

- [Android Developers: Data and file storage overview](https://developer.android.com/training/data-storage)
