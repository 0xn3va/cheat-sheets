# Abuse archives

There are weaknesses that exist when a file upload functionality accepts and extracts archives without proper security measures in place.

## Abusing symlinks

`tar` and `zip` allow you to include symlinks in tarballs/archives they generated. If an application does not properly validate the content of the archives, it can lead to arbitrary reading/writing of files.

References:
- [Report: Read files on application server, leads to RCE](https://hackerone.com/reports/178152)

## Abusing tar permissions

If an application uses Unix `tar` command to extract `.tar` files, removes symlinks and accesses subdirectory directly, you can try to bypass the symlink removing process with tar permissions. Unix `tar` command preserves the unix permissions assigned to it while creating the archive. If you create a parent directory which no one have read permissions (set chmod to `300`) while creating the subdirectory with the complete permissions (set the chmod to `700`), you can include symlinks inside the subdirectory that will not be found during the symlink removing process, but will be found when accessing directly since the subdirectory has read permissions.

```bash
$ mkdir parent
$ cd parent
$ tar cf a.tar . --mode=300
$ mkdir sub
$ cd sub
$ ln -s /etc/passwd file.txt
$ cd ..
$ tar -rf a.tar sub
```

{% embed url="https://gitlab.com/gitlab-org/gitlab-foss/-/issues/55501" %}

## Zip Slip

The Zip Slip takes advantage of zips that may contain files with specifically placed payloads set to the names, that once extracted, lead to a path traversal, and can write any file to any directory the webserver has access to. It can affect numerous archive formats, including `tar`, `jar`, `war`, `cpio`, `apk`, `rar` and `7z`. 

{% embed url="https://github.com/ptoomey3/evilarc" %}

{% embed url="https://github.com/snyk/zip-slip-vulnerability" %}

# Abuse filename

## Path as filename

Try to use different kinds of path as a filename:
- Absolute path, for example `filename=/etc/passwd`
- Relative path, for example `filename=../../../../../../etc/passwd`
- UNC path, for example `filename=\\attacker-website.com\file.png`

## Injections via filename

Try to exploit command injection or sqli via filename, for example `a$(whoami)z.png`, ```a`whoami`z.png``` or `a';select+sleep(10);--z.png`

## SSRF via filename

Try to send URL as filename to get blind SSRF, for example `filename=https://172.17.0.1/internal/file`. You can also try to change `type="file"` to `type="url"` within a request.

## DoS via large filename

Try to upload a file with large name, sometimes it leads to DoS.

References:
- [Report: profile-picture name parameter with large value lead to DoS for other users and programs on the platform](https://hackerone.com/reports/764434)

# Bypass restrictions

## Content-Type

Try to change a Content-Type value:
- Allowed MIME-type + disallowed extension
- Disallowed MIME-type + allowed extension
- Remove Content-Type
- Send Content-Type twice within request with allowed and disallowed MIME-types

## Magic bytes

If an application use a file's magic bytes to deduce the Content-type, you can try to bypass security measures by forging the magic bytes of an allowed file. For example, if GIF images are allowed, you can forge a GIF image's magic bytes `GIF89a` to make the server think we are sending it a valid GIF.

References:
- [Full list of known file magic bytes](https://en.wikipedia.org/wiki/List_of_file_signatures)

## Extension

Try to change a file extension:
- Less-common extension, such as `.phtml`
- Double extension, such as `.jpg.svg` or `.svg.jpg`
- Extension with a delimeter, such as `%0a`, `%09`, `%0d`, `%00`, `#`, etc. For example, `file.png%00.svg` or `file.png\x0d\x0a.svg`
- Empty extension, for example `file.`
- Extension with varied capitalization, such as `.sVG`
- Try to cut allowed extension with max filename length.
- Empty filename, for example `.svg`
- Right-to-left override, for example `file.%E2%80%AEphp.jpg`, see [Report: RTL override symbol not stripped from file names](https://hackerone.com/reports/298)
- Send filename twice within request with allowed and disallowed extensions, for example `filename="file.png";filename="file.svg"`

## Invalid regex

- [CVE-2018-14364: How did I find a bug in Gitlab project import and got shell access](https://blog.nyangawa.me/security/CVE-2018-14364-Gitlab-RCE/)

## Windows dots

Within Windows, when a file is created with a trailing full-stop, the file is saved **without** said trailing character, leading to potential blacklist bypasses on Windows file uploads.

For example, if an application is rejecting files that end in `.aspx`, you can upload a file called `shell.aspx.`. Now this filename will bypass the blacklist, as `.aspx != .aspx.`, but upon saving the file to the server, Windows will cut out the trailing `.`, leaving `shell.aspx`.

## Windows ADS

An Alternate Data Stream (ADS) is a little-known feature of the NTFS file system. It has the ability of forking data into an existing file without changing its file size or functionality. In other words, ADS allows you to hide a file inside another one.

The following example hides copy of `calc.exe` inside `file.txt`:

```
C:> echo Somedata > file.txt
C:> type file.txt
Somedata
C:> type c:\windows\system32\calc.exe > file.txt:calc.exe
```

To start the hidden `calc.exe` copy, you can run the following command:

```
C:> start c:\file.txt:calc.exe
```

References:
- [NTFS ALTERNATE DATA STREAMS: THE GOOD AND THE BAD](https://blog.foldersecurityviewer.com/ntfs-alternate-data-streams-the-good-and-the-bad/)

# Third-party vulnerabilities

## Vulnerabilities in image processors

{% embed url="https://github.com/barrracud4/image-upload-exploits" %}

Resources:
- Zeronights 2021: Emil Lerner – HotPics
    - [Video](https://www.youtube.com/watch?v=BQVoHLfAc8A)
    - [Slides](https://zeronights.ru/wp-content/uploads/2021/09/lerner_1.pdf)

## FFmpeg

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/server-side-request-forgery#ffmpeg" %}

## ExifTool

ExifTool versions 7.44 through 12.23 inclusive are vulnerable to a local command execution vulnerability when processing djvu files. If an application is accepting uploaded files, which are passed to ExifTool, it can lead to RCE.

References:
- [Report: RCE when removing metadata with ExifTool](https://hackerone.com/reports/1154542)
- [Writeup: ExifTool CVE-2021-22204 - Arbitrary Code Execution](https://devcraft.io/2021/05/04/exiftool-arbitrary-code-execution-cve-2021-22204.html)

# Configuration files

Some servers/frameworks work with configuration files at runtime to define various settings and restrictions. The most famous examples are the the Apache httpd/Tomcat `.htaccess` and the ASP.NET/IIS `web.config` files. You can check your server/framework and try to upload particular config to bypass some security measures or even execute code.

References:
- [Writeup: Bypass file upload filter with .htaccess](https://thibaud-robin.fr/articles/bypass-filter-upload/)
- [HTSHELLS - Self contained web shells and other attacks via .htaccess files](https://github.com/wireghoul/htshells)
- [Upload a web.config File for Fun & Profit](https://soroush.secproject.com/blog/2014/07/upload-a-web-config-file-for-fun-profit/)

# Potentially dangerous files

## ASP

Try to upload on an IIS server files with the `asp`, `ashx`, `asmx`, `asa`, `aspx`, `cer` or `xamlx` extensions to get RCE.

References:
- [PayloadsAllTheThings: Examples of insecure ASP files](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20Insecure%20Files/Extension%20ASP)

## Adobe ColdFusion

Try to upload ColdFusion files with the `cfm`, `cfml`, `cfc` or `dbm` extensions to get RCE.

### Adobe ColdFusion SSRF

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/server-side-request-forgery#adobe-coldfusion" %}

## JSP

Try to upload JSP files with the `jsp`, `jspx`, `jsw`, `jsv`, or `jspf` extensions to get RCE.

## Perl

Try to upload perl files with the `pl`, `pm`, `cgi`, or `lib` extensions to get RCE.

## SVG

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/svg-abuse" %}

## XML

Try to upload valid XML file with external entities to get XXE.

References:
- [Report: Uploaded XLF files result in External Entity Execution](https://hackerone.com/reports/232614)
- [Report: XXE at ecjobs.starbucks.com.cn/retail/hxpublic_v6/hxdynamicpage6.aspx](https://hackerone.com/reports/500515)
- [Writeup: My first XML External Entity (XXE) attack with .gpx file](https://medium.com/@valeriyshevchenko/my-first-xml-external-entity-xxe-attack-with-gpx-file-5ca78da9ae98)

# Race condition

{% embed url="https://0xn3va.gitbook.io/cheat-sheets/web-application/race-condition" %}

## File upload race condition

If an application uploads a file directly to a target folder before the file passes validation, you can abuse this behavior by using race condition.

Suppose file upload has the following flow:

1. Upload file to a target folder
2. Validate file
3. If the validation fails, remove the file. Otherwise, send the link to a user

You can use race condition to fetch the file between steps 1 and 3 while the validation is in progress.

References:
- [Web Security Academy: File upload vulnerabilities - Exploiting file upload race conditions](https://portswigger.net/web-security/file-upload#exploiting-file-upload-race-conditions)

## URL-based file upload race condition

If an application allows users to upload a file by providing a URL and fetches the file for validation to a user-accessible folder, you can abuse this behavior by using race condition.

Suppose file upload has the following flow:

1. Receive the URL from a user
2. Create a local copy for validation within a user-accessible folder
3. Validate file
4. If the validation fails, reject the URL. Otherwise, send the link to a user

You can use race condition to fetch the file between steps 2 and 4 while the validation is in progress.

References:
- [Web Security Academy: File upload vulnerabilities - Race conditions in URL-based file uploads](https://portswigger.net/web-security/file-upload#race-conditions-in-url-based-file-uploads)

# SSRF via HTTP range requests

If an application download a file from a user-provided link with [HTTP range requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests) you can try to redirect the request one of the chunks to an internal server.

References:
- [Writeup: Vimeo upload function SSRF](https://medium.com/@dPhoeniixx/vimeo-upload-function-ssrf-7466d8630437)

# References

- [Zip Slip Vulnerability](https://snyk.io/research/zip-slip-vulnerability)
- [SecurityTips: File upload bugs](https://github.com/hackerscrolls/SecurityTips/blob/master/MindMaps/File_upload_bugs.png)
- [File Upload Vulnerability Tricks And Checklist](https://www.onsecurity.io/blog/file-upload-checklist/)
- [Slides: File Upload by @0xAwali](https://docs.google.com/presentation/d/1-YwXl9rhzSvvqVvE_bMZo2ab-0O5wRNTnzoihB9x6jI/mobilepresent?slide=id.ga2ef157b83_0_156)