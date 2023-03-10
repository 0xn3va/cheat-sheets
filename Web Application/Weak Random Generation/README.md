# Overview

Applications use random values in many flows for security purposes, such as password recovery or session generation. However, not every value that seems random actually is. As a result, if an application relies on generators to produce values that can be predicted, then that application is vulnerable.

# When random values can be predicted?

There are several conditions that may allow you to predict generated random values:

- Insufficient length of generated values (this usually means the lenght < 16 bytes).
- Short alphabet that is used for generation.
- Using static values for generation.
- Using values that can be easily guessed (for example, timestamp).
- Using statistical random number generators whose output can be reproduced.

Often, the fulfillment of one or more of the conditions above will result in the ability to predict generated values.

# Random generation

## Go

Weak generation:

- [math/rand](https://pkg.go.dev/math/rand)

Crypto-strong generation:

- [crypto/rand](https://pkg.go.dev/crypto/rand)

## Java

Weak generation:

- [java.util.Random](https://docs.oracle.com/javase/8/docs/api/java/util/Random.html)
- [org.apache.commons.lang3.RandomStringUtils](https://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/RandomStringUtils.html)

Crypto-strong generation:

- [java.security.SecureRandom](https://docs.oracle.com/javase/8/docs/api/java/security/SecureRandom.html) 

    For a UNIX-like OS, the default strong generation algorithm is `NativePRNGBlocking`, which is based on `/dev/random`. As a result, `SecureRandom.getInstanceStrong()` will return a `SecureRandom` implementation that can block the current thread when the `generateSeed` or `nextBytes` methods is called.

References:
- [Everything about Java's SecureRandom](https://metebalci.com/blog/everything-about-javas-securerandom/)
- [CRACKING THE ODD CASE OF RANDOMNESS IN JAVA](https://www.elttam.com/blog/cracking-randomness-in-java/)
- [elttam/rsu-cracker - RandomStringUtils/nextInt Cracker](https://github.com/elttam/rsu-cracker)

## Node.js

Weak generation:

- [Math.random()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random)

Crypto-strong generation:

- [node:crypto](https://nodejs.org/api/crypto.html)

## Python

Weak generation:

- [random](https://docs.python.org/3/library/random.html)

Crypto-strong generation:

- [secrets](https://docs.python.org/3/library/secrets.html)

## Ruby

Weak generation:

- [Random](https://ruby-doc.org/core-2.4.0/Random.html)
- [Kernel.rand](https://ruby-doc.org/core-2.4.0/Kernel.html#method-i-rand)

Crypto-strong generation:

- [SecureRandom](https://ruby-doc.org/stdlib-2.5.1/libdoc/securerandom/rdoc/SecureRandom.html)

# UUID/GUID

Universally unique identifier (UUID) or globally unique identifier (GUID) is a 128-bit label used for information in computer systems. UUID/GUID has the following format:

```
123e4567-e89b-12d3-a456-426614174000
```

There are five versions of UUID/GUID versions defined in the [RFC4122](https://datatracker.ietf.org/doc/html/rfc4122#section-4.2.2):

- **Version 0** Only seen in the nil UUID/GUID `00000000-0000-0000-0000-000000000000`.
- **Version 1** The UUID/GUID is generated in a predictable manner based on:
    - The current time.
    - A randomly generated "clock sequence" which remains constant between UUIDs/GUIDs during the uptime of the generating system.
    - A "node ID", which is generated based on the system's MAC address if it is available.
- **Version 3** The UUID/GUID is generated using an MD5 hash of a provided name and namespace.
- **Version 4** The UUID/GUID is randomly generated.
- **Version 5** The UUID/GUID is generated using a SHA1 hash of a provided name and namespace.

You can find the UUID/GUID version number directly after the second hyphen. For example, the UUID/GUID shown above is a version 4.

```
bcd510ca-3357-48d7-8e3f-1206b9c09632
              ^
```

As you can see there is only version 4 which uses a random number generator to generate the values. Therefore, other versions can be potentially predicted. In the references you can find a link to a tool that allows generating UUID/GUID of version 1 based on a creation time and a UUID/GUID sample.

References:
- [Intruder - Cyber Security Research: In GUID We Trust](https://www.intruder.io/research/in-guid-we-trust)
- [intruder-io/guidtool: A tool to inspect and attack version 1 GUIDs](https://github.com/intruder-io/guidtool)
