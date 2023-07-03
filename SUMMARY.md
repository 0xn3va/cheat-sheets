# Summary

- [Application Security Cheat Sheet](INTRODUCTION.md)

## Android Application

- Overview
    - [Application Data & Files](Mobile%20Application/Android/Overview/app-data-files.md)
    - [Application Package](Mobile%20Application/Android/Overview/app-package.md)
    - [Application Sandbox](Mobile%20Application/Android/Overview/app-sandbox.md)
    - [Application Signing](Mobile%20Application/Android/Overview/app-signing.md)
    - [Deployment](Mobile%20Application/Android/Overview/deployment.md)
    - [Package Manager](Mobile%20Application/Android/Overview/package-manager.md)
- [Intent Vulnerabilities](Mobile%20Application/Android/Intent%20Vulnerabilities/README.md)
    - [Deep Linking Vulnerabilities](Mobile%20Application/Android/Deep%20Linking%20Vulnerabilities/README.md)
- [WebView Vulnerabilities](Mobile%20Application/Android/WebView%20Vulnerabilities/README.md)
    - [WebResourceResponse Vulnerabilities](Mobile%20Application/Android/WebView%20Vulnerabilities/web-resource-response-vulnerabilities.md)
    - [WebSettings Vulnerabilities](Mobile%20Application/Android/WebView%20Vulnerabilities/web-settings.md)

## CI/CD

- Dependency
    - [Dependency Confusion](CI%20CD/Dependency/dependency-confusion.md)
    - [Dependency Hijaking](CI%20CD/Dependency/dependency-hijacking.md)
    - [Typosquatting](CI%20CD/Dependency/typosquatting.md)
- GitHub
    - [GitHub Actions](CI%20CD/Github/actions.md)
    - [Code owners](CI%20CD/Github/codeowners.md)
    - [Dependabot](CI%20CD/Github/dependabot.md)
    - [Redirect](CI%20CD/Github/redirect.md)
    - [Releases](CI%20CD/Github/releases.md)

## Cloud

- AWS
    - [Amazon API Gateway](Cloud/AWS/api-gateway.md)
    - [Amazon Cognito](Cloud/AWS/amazon-cognito.md)
    - [Amazon S3](Cloud/AWS/s3.md)

## Container

- Overview
    - [Container Basics](Container/Overview/basics.md)
    - [Docker Engine](Container/Overview/docker-engine.md)
- Escaping
    - [CVE List](Container/Escaping/cve-list.md)
    - [Exposed Docker Socket](Container/Escaping/exposed-docker-socket.md)
    - [Excessive Capabilities](Container/Escaping/excessive-capabilities.md)
    - [Host Networking Driver](Container/Escaping/host-networking-driver.md)
    - [PID Namespace Sharing](Container/Escaping/pid-namespace-sharing.md)
    - [Sensitive Mounts](Container/Escaping/sensitive-mounts.md)
- [Container Analysis Tools](Container/container-analysis-tools.md)

## Framework

- Spring
    - [Overview](Framework/Spring/overview.md)
    - [Mass Assignment](Framework/Spring/mass-assignment.md)
    - [Routing Abuse](Framework/Spring/routing-abuse.md)
    - [SpEL Injection](Framework/Spring/spel-injection.md)
    - [Spring Boot Actuators](Framework/Spring/spring-boot-actuators.md)
    - [Spring Data Redis Insecure Deserialization](Framework/Spring/spring-data-redis-insecure-deserialization.md)
    - [Spring View Manipulation](Framework/Spring/view-manipulation.md)
- React
    - [Overview](Framework/React/overview.md)
    - [Security Issues](Framework/React/security-issues.md)

## Linux

- [Overview](Linux/Overview/README.md)
    - [Philosophy](Linux/Overview/philosophy.md)
    - [File](Linux/Overview/file.md)
    - [File Descriptor](Linux/Overview/file-descriptor.md)
    - [I/O Redirection](Linux/Overview/io-redirection.md)
    - [Process](Linux/Overview/process.md)
    - [Inter Process Communication](Linux/Overview/inter-process-communication.md)
    - [Shell](Linux/Overview/shell.md)
    - [Signals](Linux/Overview/signals.md)
    - [Socket](Linux/Overview/socket.md)
    - [User Space vs Kernel Space](Linux/Overview/user-kernel-space.md)
- [Bash Tips](Linux/bash-tips.md)

## iOS Application

- Overview
    - [Application Data & Files](Mobile%20Application/iOS/Overview/app-data-files.md)
    - [Application Package](Mobile%20Application/iOS/Overview/app-package.md)
    - [Application Sandbox](Mobile%20Application/iOS/Overview/app-sandbox.md)
    - [Application Signing](Mobile%20Application/iOS/Overview/app-signing.md)
    - [Deployment](Mobile%20Application/iOS/Overview/deployment.md)
- Getting Started
    - [IPA Patching](Mobile%20Application/iOS/Getting%20Started/ipa-patching.md)
    - [Source Code Patching](Mobile%20Application/iOS/Getting%20Started/source-patching.md)
    - [Testing with Objection](Mobile%20Application/iOS/Getting%20Started/objection.md)

## Resources

- Lists
    - [Payloads](Resources/Lists/payloads.md)
    - [Wordlists](Resources/Lists/wordlists.md)
- Researching
    - [Web Application](Resources/Researching/web-application.md)
    - [Write-ups](Resources/Researching/write-ups.md)
- Software
    - [AWS Tools](Resources/Software/aws-tools.md)
    - [Azure Tools](Resources/Software/azure-tools.md)
    - [Component Analysis](Resources/Software/component-analysis.md)
    - [Docker Analysis](Resources/Software/docker-analysis.md)
    - [Dynamic Analysis](Resources/Software/dynamic-analysis.md)
    - [Fuzzing](Resources/Software/fuzzing.md)
    - [GCP Tools](Resources/Software/gcp-tools.md)
    - [Reverse Engineering](Resources/Software/reverse-engineering.md)
    - [Static Analysis](Resources/Software/static-analysis.md)
    - [Vulnerability Scanning](Resources/Software/vulnerability-scanning.md)
- Training
    - [Secure Development](Resources/Training/secure-development.md)

## Web Application

- [Abusing HTTP hop-by-hop Request Headers](Web%20Application/Abusing%20HTTP%20hop-by-hop%20Request%20Headers/README.md)
- [Broken Authentication](Web%20Application/Broken%20Authentication/README.md)
    - [Two-Factor Authentication Vulnerabilities](Web%20Application/Broken%20Authentication/two-factor-authentication-vulnerabilities.md)
- [Command Injection](Web%20Application/Command%20Injection/README.md)
    - [Argument Injection](Web%20Application/Command%20Injection/argument-injection.md)
- [Content Security Policy](Web%20Application/Content%20Security%20Policy/README.md)
- [Cookie Security](Web%20Application/Cookie%20Security/README.md)
    - [Cookie Bomb](Web%20Application/Cookie%20Security/cookie-bomb.md)
    - [Cookie Jar Overflow](Web%20Application/Cookie%20Security/cookie-jar-overflow.md)
    - [Cookie Tossing](Web%20Application/Cookie%20Security/cookie-tossing.md)
- [CORS Misconfiguration](Web%20Application/CORS%20Misconfiguration/README.md)
- [File Upload Vulnerabilities](Web%20Application/File%20Upload%20Vulnerabilities/README.md)
- [GraphQL Vulnerabilities](Web%20Application/GraphQL%20Vulnerabilities/README.md)
- HTML Injection
    - [base](Web%20Application/HTML%20Injection/base.md)
    - [iframe](Web%20Application/HTML%20Injection/iframe.md)
    - [link](Web%20Application/HTML%20Injection/link.md)
    - [meta](Web%20Application/HTML%20Injection/meta.md)
    - [target attribute](Web%20Application/HTML%20Injection/target.md)
- [HTTP Header Security](Web%20Application/HTTP%20Headers%20Security/README.md)
- [HTTP Request Smuggling](Web%20Application/HTTP%20Request%20Smuggling/README.md)
- [Improper Rate Limits](Web%20Application/Improper%20Rate%20Limits/README.md)
- [JavaScript Prototype Pollution](Web%20Application/JavaScript%20Prototype%20Pollution/README.md)
- [JSON Web Token Vulnerabilities](Web%20Application/JSON%20Web%20Token%20Vulnerabilities/README.md)
- [OAuth 2.0 Vulnerabilities](Web%20Application/OAuth%202.0%20Vulnerabilities/README.md)
    - [OpenID Connect Vulnerabilities](Web%20Application/OAuth%202.0%20Vulnerabilities/openid-connect.md)
- [Race Condition](Web%20Application/Race%20Condition/README.md)
- [Server Side Request Forgery](Web%20Application/Server%20Side%20Request%20Forgery/README.md)
    - [Post Exploitation](Web%20Application/Server%20Side%20Request%20Forgery/post-exploitation.md)
- [SVG Abuse](Web%20Application/SVG%20Abuse/README.md)
- [Weak Random Generation](Web%20Application/Weak%20Random%20Generation/README.md)
- [Web Cache Poisoning](Web%20Application/Web%20Cache%20Poisoning/README.md)
