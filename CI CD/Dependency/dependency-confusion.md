# Overview

Companies use internal registers for placing both internal and public dependencies. In other words, they try to make a single registries with all dependencies, where dependencies are added by company employees. The goal of this approach is to minimize dependence on public registries and, accordingly, reduce the likelihood that a compromised dependency will be used internally.

However, it is not always possible to prohibit absolutely all traffic to public registries. Therefore, if a package manager tries to find an internal dependency in a public registry when resolving it, this can lead to dependency confusion.

The attack develops along the following way:

- Company has an internal package named `internal-lib`.
- `internal-lib` is not placed within a public registry, such as pypi, npm, rubygems, and etc.
- An attacker crafts a package that executes arbitrary code during installation and uploads it to a public registry as `internal-lib`.
- A package manager uses `internal-lib` from a public registry during installation that leads to code execution.

{% embed url="https://0xsapra.github.io/website//Exploiting-Dependency-Confusion" %}

# How to execute code during installation?

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/command-injection/parameters-injection" %}

# Tips

## Data collection

To strike a balance between the ability to identify an organization based on the data, and the need to avoid collecting too much sensitive information, you can collect the following data:

- username
- hostname
- current path

Along with the external IPs, there should be enough data to help security teams identify potentially vulnerable systems and avoid mistaking testing as a real attack.

## Data retrieval

Since most of the possible targets lie deep within well-protected corporate networks, DNS exfiltration is the best way to retrieve collected data. Send hex-encoded data as a part of a DNS query to a custom authoritative name server. You can use the following resources to implement DNS exfiltration:

- [1u.ms](https://github.com/neex/1u.ms) is a small set of zero-configuration DNS utilities that provide easy to use DNS rebinding utility, as well as a way to get resolvable resource records with any given contents
- [Interactsh](https://github.com/projectdiscovery/interactsh) is an OOB interaction gathering server and client library

# References

- [Writeup: Dependency Confusion: How I Hacked Into Apple, Microsoft and Dozens of Other Companies](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610)
- [Report: Insecure Bundler configuration fetching internal Gems (okra) from Rubygems.org](https://hackerone.com/reports/1104874)
- [Geek Freak: Dependency Confusion](https://dhiyaneshgeek.github.io/web/security/2021/09/04/dependency-confusion/)
- [Writeup: RyotaK's Blog - Remote code execution in cdnjs of Cloudflare](https://blog.ryotak.me/post/cdnjs-remote-code-execution-en/)
- [Black Hat EU 2021 - Picking Lockfiles: Attacking and Defending Your Supply Chain](https://gitlab.com/gitlab-com/gl-security/threatmanagement/redteam/redteam-public/red-team-tech-notes/-/tree/master/blackhat-eu-2021-picking-lockfiles)
- [DustiLock](https://github.com/Checkmarx/dustilock) is a tool to find which of your dependencies is susceptible to a Dependency Confusion attack.
- [Confuser](https://github.com/doyensec/confuser) is a tool to detect Dependency Confusion vulnerabilities that allows scanning `packages.json` files, generating and publishing payloads to the NPM repository, and finally aggregating the callbacks from vulnerable targets.
