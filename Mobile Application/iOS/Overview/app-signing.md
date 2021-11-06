Application signing allows iOS to identify who signed app and to verify that app has not been modified since developer signed it. The Signing Identity consists of a public-private key pair that Apple creates for developers.

# Certificate Signing Request

A Certificate Signing Request or CSR is a block of encoded text that is given to a Certificate Authority when applying for a certificate. It is usually generated on the server where the certificate will be installed and contains information that will be included in the certificate such as the organization name, common name (domain name), locality, and country. It also contains the public key that will be included in the certificate. A private key is usually created at the same time the CSR is created, making a key pair.

Apple (as a certificate authority) will use the CSR to create an SSL certificate for the developer, but it does not need a private key to create one. A certificate created with a particular CSR will only work with the private key that was generated with it. So if the private key is lost, the certificate will no longer work.

## What is contained in a CSR?

| Name | Explanation | Examples |
| --- | --- | --- |
| Common Name | The server fully qualified domain name (FQDN). This must match exactly what you type in your web browser or you will receive a name mismatch error. | <p>*.google.com</p><p>mail.google.com</p> |
| Organization | The legal name of the organization. This should not be abbreviated and should include suffixes such as Inc, Corp, or LLC. | Google Inc. |
| Organizational Unit | The division of the organization handling the certificate. | <p>Information Technology</p><p>IT Department</p> |
| City/Locality | The city where the organization is located. | Mountain View |
| State/County/Region | The state/region where the organization is located. This should not be abbreviated. | California |
| Country | The two-letter ISO code for the country where the organization is location. | <p>US</p><p>GB</p> |
| Email address | An email address used to contact the organization. | webmaster@google.com |
| Public Key | The public key that will go into the certificate. | |

## What does a CSR look like?

Most CSRs are created in the Base-64 encoded PEM format.

```
-----BEGIN CERTIFICATE REQUEST-----
MIIByjCCATMCAQAwgYkxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlh
MRYwFAYDVQQHEw1Nb3VudGFpbiBWaWV3MRMwEQYDVQQKEwpHb29nbGUgSW5jMR8w
HQYDVQQLExZJbmZvcm1hdGlvbiBUZWNobm9sb2d5MRcwFQYDVQQDEw53d3cuZ29v
Z2xlLmNvbTCBnzANBgkqhkiG9w0BAQEFAAOBjQAwgYkCgYEApZtYJCHJ4VpVXHfV
IlstQTlO4qC03hjX+ZkPyvdYd1Q4+qbAeTwXmCUKYHThVRd5aXSqlPzyIBwieMZr
WFlRQddZ1IzXAlVRDWwAo60KecqeAXnnUK+5fXoTI/UgWshre8tJ+x/TMHaQKR/J
cIWPhqaQhsJuzZbvAdGA80BLxdMCAwEAAaAAMA0GCSqGSIb3DQEBBQUAA4GBAIhl
4PvFq+e7ipARgI5ZM+GZx6mpCz44DTo0JkwfRDf+BtrsaC0q68eTf2XhYOsq4fkH
Q0uA0aVog3f5iJxCa3Hp5gxbJQ6zV6kJ0TEsuaaOhEko9sdpCoPOnRBm2i/XRD2D
6iNh8f8z0ShGsFqjDgFHyF3o+lUyj+UC6H1QW7bn
-----END CERTIFICATE REQUEST-----
```

# Signing process

- Create a Certificate Signing Request through the Keychain Access Application.
- Keychain Application will create a private key (stored in the keychain) and a certSigningRequest file which developer will then upload to Apple.
- Apple will proof the request and issue a certificate. The Certificate will contain the public key that can be downloaded. After downloaded, the developer need to put it into Keychain Access Application. The Certificate will be pushed into the Keychain and paired with the private key to form the **Code Signing Identity**.
- During app installation, iOS verifies that the private key that was used to sign the app matches the public key in the certificate. If this fails, the app is not installed.

# The digital signature

Signed code contains several different digital signatures:
- If the code is universal, the object code for each slice (architecture) is signed separately. This signature is stored within the binary file itself.
- Various data components of the application bundle (such as the Info.plist file, if there is one) are also signed. These signatures are stored in a file called `_CodeSignature/CodeResources` within the bundle.
- Nested code, such as libraries, helper tools, and other bits of code that are embedded in the app are themselves signed, and their signatures are also stored in `_CodeSignature/CodeResources` within the bundle.

# References

- [What is a provisioning profile & code signing in iOS?](https://medium.com/@abhimuralidharan/what-is-a-provisioning-profile-in-ios-77987a7c54c2)
- [What is a CSR (Certificate Signing Request)?](https://www.sslshopper.com/what-is-a-csr-certificate-signing-request.html)
- [Code Signing Guide: Understanding the Code Signature](https://developer.apple.com/library/archive/documentation/Security/Conceptual/CodeSigningGuide/AboutCS/AboutCS.html)
