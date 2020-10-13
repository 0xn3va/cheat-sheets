You can use the following toolkit to interact with the patched app:

{% embed url="https://github.com/sensepost/objection" %}

# Running

After the patched app has been launched, you need to find out its PID:

```bash
$ frida-ps -Ua
```

```
PID  Name                     Identifier
----  -----------------------  ---------------------------------
1234  MyApp                    com.mycompany.myapp
  23  Calendar                 com.apple.mobilecal
  16  Camera                   com.apple.camera
...
```

Use the following command to launch the application:

```bash
$ objection -g 1234 explore
```

# Commands

A quick guide to the most used objection's commands.

## Environment

```bash
$ env
```

## Run OS command

```bash
$ !cat Info.plist 
```

## Download file

```bash
$ file download Info.plist
```

## Import Frida scripts

In order to import and **run** the Frida script use the following command:

```bash
$ import "/tmp/frida-script.js"
```
