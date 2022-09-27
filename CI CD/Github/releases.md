# Overview

Github provides the release functionality as a way to publish iterations of packaged software as releases. Releases include a compresed snapshot of the source code as `.zip` and `.tar.gz` files. Additionally, Github allows you to add extra files that you can attach when creating or editing a release. Moreover, these extra files can be modified after a release has been created.

To reproduce the issue follow the next steps:

1. Create a public repository on Github
2. Create a release and add `test.sh` file
3. Invite an `attacker` user as a collaborator
4. `attacker` can edit the release and modify `test.sh` file
5. There is no indication the release was modified in Github UI

In other words, an attacker can compromise the account of any project collaborator and modify releases without the knowledge of project owners. This is possible for the following reasons:

1. Release assets can be modified after initial publication (excluding source code snapshots)
2. Any project collaborators can modify releases. There are no permissions to allow an owner to prevent a release from being changed
3. UI does not notify or indicate that a release has been modified ([the releases API](https://docs.github.com/en/rest/reference/repos#releases) exposes additional information about release assets)
4. The `verified` flag is displayed if a git commit has been verified (this only applies to the source code snapshot, not extra files)

# References

- [Writeup: Supply Chain Attacks via GitHub.com Releases](https://wwws.nightwatchcybersecurity.com/2021/04/25/supply-chain-attacks-via-github-com-releases/)
