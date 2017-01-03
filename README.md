# iOS Dropbear SSH

This is a modified version of Matt Johnston's [dropbear](https://matt.ucc.asn.au/dropbear/dropbear.html) ssh daemon to be used on iOS in combination with exploits such as Ian Beer's [mach_portal](https://bugs.chromium.org/p/project-zero/issues/detail?id=965#c2).

## Installation
The following description assumes that you followed the instructions to build and run mach_portal.
  * Download and unpack [jtool](http://newosxbook.com/tools/jtool.tar)
  * Run ```xcodebuild -showsdks``` to determine iOS SDK Version
  * Adjust JTOOL and SDK variables within build.sh (and ARCH if needed)
  * Run build.sh 

# Usage
As mach_portal spawns a listening shell on TCP port 4141, the easiest way to launch dropbear is to package it with mach_portal and launch if via netcat.
Assuming you have installed the iosbinpack, you need the following steps:
  * Copy dropbear, dropbearkey, dbclient, and dropbearconvert to usr/local/bin within iosbinpack
  * Generate authorized_keys file and store it under etc/dropbear within iosbinpack
  * Recompile mach_portal

Use the -S option of dropbear to pass a chroot-like system environment, i.e. the iosbinpack directory, which is used as a root directory for /bin/sh on all logins.

The following is an example run:
```
❯ nc 10.0.20.44 4141
id
uid=0(root) gid=0(wheel) groups=0(wheel),1(daemon),2(kmem),3(sys),4(tty),5(operator),8(procview),9(procmod),20(staff),29(certusers),80(admin)
# let's start dropbear ssh
dropbear -S /var/containers/Bundle/Application/53860657-F635-4693-90F3-61A6FA550168/mach_portal.app/iosbinpack64/ -E -m -F
[233] Jan 03 15:49:20 Not backgrounding
[235] Jan 03 15:49:25 Child connection from 10.0.20.10:52228
[235] Jan 03 15:49:27 Pubkey auth succeeded for 'root' with key sha1!! 0e:4d:7c:54:8b:f7:b6:ce:5a:19:c6:29:53:9a:39:57:cb:a1:d5:ca from 10.0.20.10:52228
[235] Jan 03 15:49:27 User root executing login shell
```

Please note that you have to adjust the -S option here to match the iosbinpack within the bundle path of your mach_portal app.

## FAQ
### Why are you releasing this?
With the release of mach_portal, Ian Beer and Google Project Zero have released a great opportunity for security researchers to conduct iOS research. At the same time, the TCP-bindshell is not very convenient when it comes to exploring the system using multiple sessions, copying files to and from the device etc. While iosbinpack includes a dropbear version that is compiled for iOS, it lacks modifications needed in order to make it work with jailbreaks such as mach_portal. Since only the iosbinpack binaries are distributed, changes can only be made on the binary version, which is not convenient either.

Especially if you are new in iOS research, exploring the system from a shell is a great way to get to know the system. At the same time we hope that this can serve as a simple example on how to build native code for iOS devices.

### What are the changes?
In order to run, dropbear expects host keys to be present under a fix directory structure, which doesn't exist. Instead, we generate host keys under our chroot-like system environment (etc/dropbear). It also uses pwnam entries when determining the login shell, which on an iOS device would be the non-existent /bin/sh file. Based on the -S option, we instead look for bin/sh within that directory structure. Lastly, dropbear can authenticate using passwords and public keys. 

Password authentication is sub-optimal as we either would need to allow arbitrary passwords or fall back to the infamous and insecure alpine system password. Public key authentication can be used, but requires key material to be deployed in user home directories. Instead, we simply load an authorized_keys file from within etc/dropbear in our system environment (-S) for all logins.

### Why this and not yalu or ..?
We don't aim at providing similar functionality as contained in jailbreaks such as yalu or others that may already package Cydia and ship an ssh server. Instead, we focus on packaging an ssh server only that is useful for exploring the system, while keeping system modifications minimal. There are multiple reasons for this, but first and foremost our intention is to retain the original file system and not cause system changes by using additional vulnerabilities in order to achieve persistence.

### Can I used password logins?
In case you insist on enabling password-based logins, remove the BYPASS_PASSWD ifdef in svr-authpasswd.c.

### Isn't leaving the original environment a potential security problem?
Not scrubbing the environment can be a security issue. Similarly, not caring about authorized_keys file permissions can be. Since we assume that this is used in a research environment, we decided to leave the original environment for convenience. As mach_portal for example adjusts PATH to include iosbinpack binaries, we don't have to setup these paths again from within dropbear and drop the user into a decent shell environment.
