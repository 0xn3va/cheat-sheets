# Overview

When a user change their username or an organization name, Github creates a redirect route that allows the repositories to be accessible from their old URLs. After changing a username or an organization name old ones become available to claim. This means an attacker can claim the abandoned username or organization name and break the redirection. Therefore, if someone uses the old URL, they will be dealing with attacker's repository.

The same applies to repositories that have been transferred.

# References

- [New Type of Supply Chain Attack Could Put Popular Admin Tools at Risk](https://www.intezer.com/blog/malware-analysis/chainjacking-supply-chain-attack-puts-popular-admin-tools-at-risk/)
- [Report: Dependency repository hijacking aka Repo Jacking from GitHub repo rubygems/bundler-site & rubygems/bundler.github.io + bundler.io docs](https://hackerone.com/reports/1430405)
