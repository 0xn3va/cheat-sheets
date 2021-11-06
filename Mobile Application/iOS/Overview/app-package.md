iOS apps are distributed in IPA (iOS App Store Package) archives. The IPA file is a ZIP-compressed archive that contains all the code and resources required to execute the app.

At a high level, the IPA file has the following structure:

| Name | Type | Description |
| --- | --- | --- |
| Payload | Directory | This directory contains all the application data. |
| Payload/MyApp.app | Directory | [Bundle directory](/Mobile%20Application/iOS/Overview/app-data-files.md#bundle-container-structure). |
| Payload/MyApp.app/MyApp | File | This file contains the application's executable code |
| Payload/MyApp.app/embedded.mobileprovision | File | This plist file contains the [provisioning profile](/Mobile%20Application/iOS/Overview/deployment.md#provisioning-profiles) for an application. |
| Payload/MyApp.app/Info.plist | File | This file is the manifest of the iOS application. |
| iTunesArtwork | File | PNG image used as the application's icon |
| iTunesMetadata.plist | File | This file contains various bits of information, including the developer's name and ID, the bundle identifier, copyright information, genre, the name of the app, release date, purchase date, etc. |
| WatchKitSupport/WK | Directory | This specific bundle contains the extension delegate and the controllers for managing the interfaces and responding to user interactions on an Apple Watch. |
| META-INF | Directory | This directory contains metadata about what program was used to create the IPA. |

# References

- [Bundle Programming Guide: Bundle Structures](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFBundles/BundleTypes/BundleTypes.html)
- [iOS Testing Guide: Platform Overview](https://mobile-security.gitbook.io/mobile-security-testing-guide/ios-testing-guide/0x06a-platform-overview)
