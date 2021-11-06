Unlike Android, you cannot install any application on an iOS device, as only Apple allows the application to run. When developers submit an application to the App Store, Apple manually installs the application on test devices and verifies that the application works as described, does not crash during normal use, and complies with their [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/). Once the application passes the review process, Apple re-signs it with its security certificate. And only after that, the application is ready for publication in the App Store.

Apple allows developers to run their applications on an iOS device on behalf of Apple using a provisioning profile. However, this is not the same "sideloading" that can be done on Android, and provisioning profiles impose restrictions on an application.

# Provisioning profiles

The provisioning profile is a collection of digital entities that uniquely ties developers and devices to an authorized development team and enables a device to be used for testing. In other words, the provisioning profile acts as a link between the device and the developer account. During development, the developer chooses on which devices the application can run and which application services it can access. This allows for sufficient flexibility for debugging and testing on the device, including distributing the application to beta testers.

The provisioning profile is downloaded from a developer account as a .mobileprovision file and embedded in the application bundle. In turn, the entire bundle is signed.

## .mobileprovision file

The .mobileprovision file has the following structure:
- **File header** - is a binary data (looks like it does not contain any useful data).
- **plist** - is a plist file with all project-related information.
- **File footer** - is a binary data (looks like that is the file signature).

Example of ad hoc profile plist file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>AppIDName</key>
  <string>iOS My App</string>
  <key>ApplicationIdentifierPrefix</key>
  <array>
      <string>29QTLV4HFP</string>
  </array>
  <key>CreationDate</key>
  <date>2019-09-27T22:32:55Z</date>
  <key>Platform</key>
  <array>
    <string>iOS</string>
  </array>
  <key>IsXcodeManaged</key>
  <false/>
  <key>DeveloperCertificates</key>
  <array>
      <data>
      MIIFYTCCBEmgAwIBAgIIYToPcWigCVswDQYJKoZIhvcNAQ
      ...
      uFL3pxVdY562UQ58glSEyw4OlkMFLD5IxGZovLuaULzUtq
      FBO/19kvi29WEdJ0kvwhHa+Ba4WbjdCC3aiusr+qHQ==
      </data>
  </array>
  <key>Entitlements</key>
  <dict>
      <key>application-identifier</key>
      <string>29QTLV4HFP.com.mycompany.myapp</string>
      <key>get-task-allow</key>
      <false/>
      <key>keychain-access-groups</key>
      <array>
          <string>29QTLV4HFP.*</string>
      </array>
  </dict>
  <key>ExpirationDate</key>
  <date>2020-02-10T22:32:55Z</date>
  <key>Name</key>
  <string>MyApp Ad Hoc</string>
  <key>ProvisionedDevices</key>
  <array>
      <string>6d08...2ef8</string>
      <string>effd...0381</string>
      <string>a8b6...c437</string>
  </array>
  <key>TimeToLive</key>
  <integer>136</integer>
  <key>UUID</key>
  <string>C248F364-66BD-45E2-BC7C-D736B9FBC809</string>
  <key>Version</key>
  <integer>1</integer>
</dict>
</plist>
```

The provisioning profile consists of several items, the most important of which are described below.

| Item | Description |
| --- | --- |
| App ID | An App ID is a two-part string (`29QTLV4HFP.com.mycompany.myapp`: alpha-numeric characters and App Bundle ID) used to identify one or more apps from a single development team (this can include a `*` wild card to be used for many applications with similar bundle identifiers). |
| <p>Development Certificates | Development Certificate is a unique security certificate issued by Apple that uniquely identifies you as the developer or publisher of the application. The private key of the distribution certificate is used to sign the application. There are two types of signing certificates:</p><p>**Development certificate** is used for individual developers who are actively debugging and developing an application.</p><p>**Production certificate** is used either a production setting (final build for the App Store) or a QA build that is distributed through Apple's TestFlight Beta or another app test distribution system. These certificates identify you as an App Store Publisher to Apple or as a Test Distributor and can only be used with production provisioning profiles.</p> |
| Unique Device Identifiers | List of device IDs on which the application can run on. |
| Entitlements | <p>Entitlements are special permissions or capabilities required by an application to interact with other Apple services external to the application. Because iOS apps are isolated from each other, they cannot directly interact with other operating system services without entitlements. The types of entitlements include the following:</p><p>> Push Notifications</p><p>> Cloud Kit</p><p>> Keychain Sharing</p><p>> Keychain Sharing</p><p>> etc.</p> |

The development and app store profiles contain almost the same data with small differences:
- The development profile is almost identical to the ad hoc profile except of `Entitlements.get-task-allow` set to `true`.
- The app store profile does not contain the `ProvisionedDevices` key.

## Types of provisioning profiles

There are 4 different types of provisioning profiles:
- Development
- Ad Hoc
- App Store
- Enterprise

### Development provisioning profile

The development provisioning profiles are used only when developing an application. The development provisioning profile must be installed on each device on which developers want to run the application. If the information in the provisioning profile does not match certain criteria, the application will not start.

Devices specified within the development provisioning profile can be used for testing only by those individuals whose development certificates are included in the profile. A single device can contain multiple provisioning profiles.

### Ad Hoc provisioning profile

The ad hoc provisioning profile is similar to the development provisioning profile, but it allows development teams to distribute applications to a specific target set of iOS devices outside of the App Store. Ad hoc provisioning profiles are used to distribute applications to testers who are not included in the iOS Developer Program for an organization.

An app deployed with an ad hoc provisioning profile will be almost identical to the production version is submitting to the App Store (ie. it will need a distribution certificate for push notifications to work and etc.)

### App Store provisioning profile

The app store provisioning profile is used to submit app to the App Store for distribution. The difference between development and app store profiles is that app store profiles do not specify any device IDs.

### Enterprise provisioning profile

The [Apple Developer Enterprise Program](https://developer.apple.com/programs/enterprise/) allows companies to create and distribute in-house applications to their employee's iOS devices through the use of the enterprise provisioning profile.

Enterprise provisioning profiles do not need a list of device IDs giving companies the flexibility to install their in-house proprietary apps to an unlimited number of devices that are registered with their enterprise.

# Deployment process

![](img/app-to-profile-mapping.png)

When you deploy the application on a device the following things happens:
- The provisioning profile in the Mac goes to the developer certificate in your keychain.
- Xcode uses the certificate to sign the code.
- Device's UUID is matched with the IDs in the provisioning profile.
- App ID in the provisioning profile is matched with the bundle identifier in the application.
- The entitlements required are associated with the App ID.
- The private key used to sign the application matches the public key in the certificate.

If all the above steps are successful the signed binary is sent to the device and is validated against the same provisioning profile in the application and finally launched. If anyone of these conditions fail, then the application will not install — and you will see a greyed-out app icon.

# References

- [What is a provisioning profile & code signing in iOS?](https://medium.com/@abhimuralidharan/what-is-a-provisioning-profile-in-ios-77987a7c54c2)
- [.mobileprovision Files Structure and Reading](https://web.archive.org/web/20130502092617/http://idevblog.info/mobileprovision-files-structure-and-reading)
- [Making Sense Of iOS Provisioning](https://www.sharpmobilecode.com/making-sense-of-ios-provisioning/)
