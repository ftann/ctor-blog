---
title: "Gradle remote build"
keywords: ["gradle", "build"]
tags: ["build", "gradle", "java", "ssh"]
date: 2021-02-22T00:02:53+01:00
---

In my job I have a macbook as work horse. It's an astonishing machine but certainly not the fastest
around to compile software. I work from home where my main rig is idling and the macbook is nearly
melting. __No bueno!__

I could also try to switch to my main machine for developing, but I like to keep personal stuff
separate from work.

The solution is to let the main rig do the heavy lifting while the macbook does what it's doing
best, looking pretty.

# A [Mirakle][0]

Mirakle saves our precious macbooks from heat death, decreases build times drastically and increases
battery life. It's a gradle plugin that leverages ssh and rsync to run the build on a remote
machine.

# ssh key

A key is necessary to connect to the remote machine without password prompt. To create one head to
your terminal:

```shell
ssh-keygen -t rsa -b 4096 -C "build" -f build
```

From the newly generated key pair we need the public key from `build.pub` (just referenced as
`build-pub-key`) on the remote machine. The private key `build` is needed on the macbook.

# Setup remote machine

On the remote machine we use docker to keep the work stuff contained. An easy to
use [docker container image][1] is provided by the linuxserver.io project. We also take the easy
route here and directly supply the public key to the container.

Write the following content to a new file `Dockerfile` in a new directory (e.g. `build-server`).

```dockerfile
FROM ghcr.io/linuxserver/openssh-server

ENV USER_NAME=build \
    PUBLIC_KEY="ssh-rsa <build-pub-key> build"
    
RUN mkdir /config/.gradle && \
  echo "org.gradle.parallel=true" >> /config/.gradle/gradle.properties && \
  echo "org.gradle.caching=true" >> /config/.gradle/gradle.properties && \
  chmod 755 -R /config/.gradle
RUN apk add --no-cache git openjdk11 rsync
```

To build and then run the container you can use your terminal and change the directory to the
`build-server`.

```shell
docker build -t build-server .
docker run -p 2222:2222 --name build-server build-server:latest
```

Now we have a remote build server running!

## Installed packages

My company uses the nebula gradle plugin to determine the version number of our projects and
therefore `git` is required. `openjdk11` and `rsync` are required by Mirakle.

## Gradle properties

Gradle doesn't run task parallel by default (version 6.8.2). It's enabled by setting
`org.gradle.parallel` to true. The build cache is enabled that further decreases the build times.

# Setup local machine

Our beloved macbook comes with all the necessary tools already installed. Only the ssh connection
must be set up. To do so make sure that at least the [created private key](#ssh-key) is available
as `~/.ssh/build`.

Add an entry for the connection in `~/.ssh/config`:

```shell
Host build
  User build
  HostName <the-build-server>
  Port 2222
  IdentityFile ~/.ssh/build
  PreferredAuthentications publickey
  ControlMaster auto
  ControlPath /tmp/%r@%h:%p
  ControlPersist 1h
```

Have a look at the wiki entry of Mirakle to [local machine setup][3].

# Setup Mirakle

To enable Mirakle create a file in `~/.gradle/init.d/mirakle_init.gradle` with the following
content:

```groovy
initscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath "com.instamotor:mirakle:1.4.1"
    }
}

if (startParameter.taskNames.contains("mirakle")) {

    apply plugin: Mirakle

    rootProject {
        mirakle {
            host "build" // The configured ssh connection.
            excludeCommon = [".gradle", ".idea"] // ".git" is excluded by default.
            downloadInParallel true
        }
    }

}
```

The default is to run builds on the local machine. To send a build to the build server add `mirakle`
to the gradle tasks.

```shell
./gradlew build mirakle
```

## Before first usage

The configured ssh server probably isn't listed in the `known_hosts` file and this will fail the
build. To solve this, add the build server to the known hosts by connecting once:

```shell
ssh build
```

## To nebula users

Make sure that the `.git` folder isn't excluded from being copied to the remote machine. Nebula
requires the repository information to be present.

# The count

A quick comparison of a clean build with my macbook with a [i7-7820HQ][4], my main rig with an
[i5-9600K][5] and the company build server with a [E5-2690 v3][6].

Machine | t
:------ | ---:
MacBook | 7m 41s
Workstation | 4m 4s
Server | 2m 41s

# Conclusion

Those numbers matter! Faster build times mean less downtime for developers. Calculate for yourself
how much time and money you could save.

# So long

Do you like the guide and want to give feedback or found a mistake? Then send me a mail
to `f4ntasyland /at/ protonmail /dot/ com`

You can always buy me a beer.
`（ ^_^）o自自o（^_^ ）`

_[xmr][6]:
473WTZ1gWFdjdEyioCQfbGQKRurZQoPDJLhuFJyubCrk4TRogKCtRum63bFMx2dP2y4AN1vf2fN6La7V7eB2cZ4vNJgMAcG_

[0]: https://github.com/Adambl4/mirakle

[1]: https://github.com/linuxserver/docker-openssh-server

[2]: https://docs.gradle.org/current/userguide/build_environment.html

[3]: https://github.com/Adambl4/mirakle/blob/development/docs/SETUP_LOCAL.md

[4]: https://ark.intel.com/content/www/us/en/ark/products/97496/intel-core-i7-7820hq-processor-8m-cache-up-to-3-90-ghz.html

[5]: https://ark.intel.com/content/www/us/en/ark/products/134896/intel-core-i5-9600k-processor-9m-cache-up-to-4-60-ghz.html

[6]: https://ark.intel.com/content/www/us/en/ark/products/81713/intel-xeon-processor-e5-2690-v3-30m-cache-2-60-ghz.html

[7]: https://www.getmonero.org/
