# Overview

Github allows you to define for individuals or teams that are responsible for code in a repository or [code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners). To do so, you should add the `CODEOWNERS` file to one of the following locations:

- `.github/`
- `/`
- `docs/`

After which you can set up rules for protected branches and require mandatory approval from code owners.

References:
- [Github Docs: About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)

# Code ownership takeover

The documentation for code owners [said](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) that the `CODEOWNERS` file can be defined in `/`, `docs/`, or `.github/` directory:

```
To use a CODEOWNERS file, create a new file called CODEOWNERS in the root, docs/, or .github/ directory of the repository, in the branch where you'd like to add the code owners.
```

However, what happens if a repository contains multiple CODEOWNERS files? Actually, among the allowed paths there is the following priority:

- `.github/`
- `/`
- `docs/`

So, if Github finds `CODEOWNERS` file in `.github/`, it will ignore `CODEOWNERS` files in `/` and `docs/`. In other words, if `CODEOWNERS` file has been created in `/` or `docs/`, an attacker with write permissions is able to add `CODEOWNERS` file to `.github/`, takeover code ownership, and bypass branch protection rules.Now the attacker is the owner of the code of the entire repository and can approve any changes.

Suppose there is a repository where `.github/` has separate owners who are responsible for changes to that directory and `CODEOWNERS` file is stored in `/`. In such case, the `CODEOWNERS` file may look like this:

```
* @owner-team
.github/ @dev-team
```

A member of the `@dev-team` team, or an attacker who gains access to the account of this member, can elevate their privileges in this repository using the next steps:

1. Using a personal Github account or other compromised account fork the repository.
1. Add `.github/CODEOWNERS` file with the following content:

    ```
    * @dev-team
    ```

1. Create a PR to the target repo.
1. Approve the PR (since an attacker has access to the account that is an code owner of the `.github/`, they can approve any changes within `.github/`).
1. Merge changes.
1. Now an attacker is a code owner for the whole repository and they are able to approve any changes, including those outside `.github/`.
