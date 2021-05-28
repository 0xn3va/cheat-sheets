![](https://drawings.jvns.ca/drawings/bashtips.png)

# Bash multiprocessing

{% embed url="https://fr1nge.xyz/posts/supercharge-your-bash-scripts-with-multiprocessing/" %}

# Useful commands

## aria2c

[aria2](https://github.com/aria2/aria2) is a lightweight multi-protocol & multi-source, cross platform download utility operated in command-line.

```bash
# Multi-thread downloading
$ aria2c -x5 <URL>

# Restarting aria2c continues unfinished download
$ aria2c -x5 <URL>
^C
$ aria2c -x5 <URL> # downloading continues

# Download torrent files (just pass the torrent file to the input)
$ aria2c <name>.torrent
```

## chmod

```bash
# Enable all permissions for the owner, write permissions to the group and execute permissions to others
# or rwx-w---x
$ chmod 721 <name>

### Verbose form
# Enable user for rwx
$ chmod u+rwx <name>

# Enable group for w
$ chmod g+w <name>

# Enable others for x
$ chmod o+x <name>

# Enable everyone for x
$ chmod a+x <file>

### Remove permissions (use '-' instead of '+')
# Disable group and others for x
$ chmod og-x <name>

### setuid (4), setgid (2), sticky (1) bits
# Set setgid bit
# or rwxr-sr-x
$ chmod 2755 <name>

## Verbose form
# Set setuid bit 
$ chmod u+s <name>

# Set setgid bit
$ chmod g+s <name>

# Set sticky bit
$ chmod o+t <name>
```

## find

```bash
# Case sensitive search
$ find / -name '*some*'

# Case insensitive search
$ find / -iname '*some*'

# ls style output formatting
$ find / -iname '*some*' -ls

# Delete found files (danger: there is no confirmation)
$ find dir_to_delete/ -delete

# Executing a script with search results
# Format: 
#   -exec <command> {} \;
#   <commad> - script/command to execute
#   {} - place for the found file
#   \; - end of <command>
$ find / -iname '*some*' -exec ./script.sh {} \;
```

## grep

```bash
# Search string without using regex
# fgrep is an alias for grep -F
$ fgrep

# Using perl-compatible regex (or "real" regex)
$ grep -P

# Searching inside gz archives
$ zgrep

# Highlight found words in search results
$ grep --color=force

# Invert the sense of matching
$ grep -v

# grep by file contents
$ grep -rnw '/path/to/somewhere/' -e 'pattern'
```

## ipython

[ipython](https://github.com/ipython/ipython) is a handy command shell for python, which supports:
- Tab completion
- Help by `.method?` + enter
- Embedding in a script for debugging using the interactive shell:
    ```python
    from IPython import embed; embed()
    ```

## ncdu

[ncdu](https://dev.yorhel.nl/ncdu) is a disk usage analyzer with an ncurses interface.

```bash
$ ncdu /
```

Useful hotkeys:
- Navigation by arrows
- `s` - sort by size
- `C` - sort by quantity
- `c` - show quantity
- `d` - delete

## pv

[pv](https://linux.die.net/man/1/pv) - monitor the progress of data through a pipe.

```bash
$ ./app1 | pv | ./app2
$ cat /dev/urandom | pv | xxd > /dev/null

# cat-like behavior with a progress bar
$ pv file | ./app
$ pv some_file.txt | bzip2 > /dev/null

# tar compression progress bar
$ tar cz /folder | pv > folder.tar.gz

# Network data transfer progress bar
$ pv folder.tar.gz | nc -nlvp 1337

# Monitor other process
$ pv -d PID

# Limit speed
$ pv file.txt -L 2 # 2 bytes per second
$ pv file.txt -L -l 2 # 2 lines per second
```

## tar

[tar](https://linux.die.net/man/1/tar) saves many files together into a single tape or disk archive, and can restore individual files from the archive.

Useful operation mode:
- `-c` - create a new archive
- `-f` - use archive file or device ARCHIVE
- `-j, --bzip2` - compress/decompress the archive through bzip2
- `-z, --gzip` - compress/decompress the archive through gzip
- `-t, --list` - list the contents of an archive
- `-x` - extract files from an archive
- `-C, --directory=DIR` - change output directory to DIR
- `-v` verbosely list files processed

```bash
# Create tar archive
$ tar cf archive.tar foo bar

# Create bzip2 archive
$ tar cf archive.tar.bz2 foo bar

# Create gzip archive
$ tar cf archive.tar.gz foo bar

# Extract archive
$ tar xf archive.tar.gz
```

## youtube-dl

[youtube-dl](https://github.com/ytdl-org/youtube-dl) is a command-line program to download videos from youtube and other video sites.

```bash
$ youtube-dl https://youtu.be/somth12asd34
```