On UNIX the streams for getting input and writing output are predefined. There are always three default files open:
- `stdin` is the input data source (the keyboard)
- `stdout` is the output data source (the screen)
- `stderr` is the standard error output source (error messages output to the screen)

These, and any other open files, can be redirected. Redirection simply means capturing output from a file, command, program, script, or even code block within a script and sending it as input to another file, command, program, or script.

```bash
## command_output >
# Redirect stdout to a file
# Creates the file if not present, otherwise overwrites it.
$ ls -la > ls_result.txt

## : > filename
# The '>' truncates file 'filename; to zero length
# If file not present, creates zero-length file (same effect as 'touch')
# The ':' serves as a dummy placeholder, producing no output
$ : > empty_file.txt

## command_output >>
# Redirect stdout to a file
# Creates the file if not present, otherwise appends to it
$ ls -la >> ls_result.txt
```

Each open file gets assigned a [file descriptor](/Linux/Overview/file-descriptor.md). The file descriptors for `stdin`, `stdout`, and `stderr` are `0`, `1`, and `2`, respectively. For opening additional files, there remain descriptors `3` to `9`.

```bash
## m>n
# 'm' is a file descriptor, which defaults to 1, if not explicitly set
# 'n' is a filename
# File descriptor 'm' is redirect to file 'n'
$ ls -la > ls_result.txt
$ ls -la 1>ls_result.txt
$ ./script.sh 2>error.log

## m>&n
# 'm' is a file descriptor, which defaults to 1, if not set
# 'n' is another file descriptor
$ ./script.sh 2>&1 > log.txt

## &>filename
# Redirect both stdout and stderr to file 'filename'
$ ./script.sh &>filename

## < filename
# Accept input from a file
$ grep "*some*" <filename
$ grep "*some*" 0<filename
```

You can ensure none of your redirects clobber an existing file by setting the `noclobber` option in the shell:

```bash
$ set -o noclobber
$ sort file.txt > file.txt
-bash: file.txt: cannot overwrite existing file
```

# References

- [How Unix Works: Become a Better Software Engineer](https://neilkakkar.com/unix.html)
- [Advanced Bash-Scripting Guide: I/O Redirection](https://tldp.org/LDP/abs/html/io-redirection.html)