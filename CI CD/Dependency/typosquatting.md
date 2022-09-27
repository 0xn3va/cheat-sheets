# Overview

Typosquatting has long been a known problem for domain names. However, the same approach can be used to get the execution of arbitrary code in build systems or on the machines of company employees that use public registries during dependency installation.

The attack develops along the following way:

- An attacker crafts a package that execute an arbitrary code during installation
- The package is typo versions of popular packages that are prone to be mistyped, for instance, `reqeusts` (instead of `requests`)
- An attacker uploads the package to a public registry, such as pypi, npm, rubygems, and etc.
- If somebody unintentionally installs such a package, an attacker gain an arbitrary code execution.

Additionally, it can work if you bet on mistakes when typing a command. For example, let's look at installing dependencies in `python` with `pip`:

```bash
$ pip install -r requirements.txt
```

However, it is quite easy to miss the `-r` key while typing and the command will already look like this:

```bash
$ pip install requirements.txt
```

In other words, instead of installing dependencies from the `requirements.txt` file, a package with `requirements.txt` name will be installed. So, an attacker only needs to create and upload a malicious package named `requirements.txt`.

Other example is `-` npm package that has over 700.000 downloads, for more details check out "[Empty npm package '-' has over 700,000 downloads — here's why](https://www.bleepingcomputer.com/news/software/empty-npm-package-has-over-700-000-downloads-heres-why/)".

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

# Referneces

- [Writeup: Typosquatting programming language package managers](https://incolumitas.com/2016/06/08/typosquatting-package-managers/)
