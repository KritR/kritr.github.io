---
title: Developing and Contributing a MacPort
date: 2019-06-30 01:50 UTC
thumbnail_image: developing-and-contributing-a-macport/header-design.png
cover_image: 2019-06-30-developing-and-contributing-a-macport/header-design.png
tags:
  - mac
  - contributing
  - open source
  - development
  - pull request
  - pr
  - github
  - go
  - package manager
  - osx
---

I was looking for the DigitalOcean command line tool to install on my Mac.
Unfortunately, they didn't already have a MacPorts
compatible package I could install. In the end, I created my own Port
--what MacPorts calls a software package--
and got to learn the ins and outs of how this package manager actually works. If
you are on a Mac and aren't already using a package manager, I highly recommend
checking out either [MacPorts](https://macports.org) or [Brew](https://brew.sh).

While the documentation for MacPorts is fairly maintained and readable, I
struggled in finding a starting point to set up my port development
environment.

This article aims to outline the procedures I found in developing my port and
hopefully gives some insight into the process.

## Setting up a Port Development Environment

Initial setup for development is fairly straightforward.

First create a directory for the custom Portfiles. For permission reasons, it's
easiest to create the custom port in a root level directory owned by root.
I chose to create mine in `/opt` just because that's MacPort's primary
directory.

```
> mkdir /opt/custom/
```

Then create a subdirectory for the specific package. This will also
involve deciding on a category for the package. A list of all available
categories is on the [available ports page](https://www.macports.org/ports.php).

The directory created will have the path `/opt/custom/category_name/port_name`.

```
> mkdir -p /opt/custom/www/doctl/
```

The directory structure will look like the following when finished.

```
# /opt
# └── custom
#     └── www
#         └── doctl
```

Now create a placeholder Portfile for the port.

```
> touch /opt/custom/www/doctl/Portfile
```

Running the `portindex` command in the `/opt/custom` directory
creates a PortIndex file so MacPorts know which ports are in the directory.

```
> cd /opt/custom
> portindex
```

It successfully finds the port and obviously fails because the Portfile is empty.

```
# Creating port index in /opt/custom
# Failed to parse file www/doctl/Portfile: invalid command name "universal_setup"

# Total number of ports parsed:   1
# Ports successfully parsed:      0
# Ports failed:                   1
# Up-to-date ports skipped:       0
```

MacPorts also needs to recognize this directory as a
[local Portfile repository](https://guide.macports.org/chunked/development.local-repositories.html). Adding it to the sources file located `/opt/local/etc/macports/sources.conf` should fix that.
This path may vary if you've changed your MacPorts install prefix.

```
> $EDITOR /opt/local/etc/macports/sources.conf
```

By adding the directory `file:///opt/custom/` before the main
repository for ports, MacPorts will recognize the custom ports
before the public ones.

`sources.conf` should end up looking like this.

```
file:///opt/custom/
rsync://rsync.macports.org/macports/release/tarballs/ports.tar [default]
```

And we're now set to fill the contents of the actual Portfile.

## Creating a Portfile

The corresponding documentation for this section of Portfile creation is can be
found at <https://guide.macports.org/#development.introduction>

Let's begin editing the Portfile.

```
> $EDITOR /opt/custom/www/doctl/Portfile
```

First step is to add the editor configuration as a comment at the top, which will
automatically tell Vim/Emacs how to format the Portfile.

```
# -*- coding: utf-8; mode: tcl; tab-width: 4; indent-tabs-mode: nil; c-basic-offset: 4 -*- vim:fenc=utf-8:ft=tcl:et:sw=4:ts=4:sts=4
```

Next specify the `PortSystem`, a required field for every Portfile that
should be set to `1.0`

```
PortSystem          1.0
```

At this point it's necessary to figure out the type of port being built.

By default MacPorts is set for building programs with a
Makefile, and runs `./configure ; make ; make install`.
In addition, each Port should have to specify its name, version, and all
other build details.
However, MacPorts also comes with a set of base definitions for Portfiles,
and using one of these definitions can be done by assigning the new port to a
Portgroup.
The full list of PortGroups is
[here](https://guide.macports.org/chunked/reference.portgroup.html).

In my case, the command line application I'm packaging is built in Golang, so I'm
going to set the Portgroup to Golang and and define the appropriate variables.

```
PortGroup           golang 1.0
go.setup            github.com/digitalocean/ doctl 1.20.1 v
```

Defining go.setup tells the portfile to fetch its dependencies from the digital
ocean github repo, on the `doctl` repository, and to use the release `1.20.1`
, which is prefixed by the character `v`.
The actual release link is <http://github.com/digitalocean/doctl/releases/v1.20.1>.

Now the Portfile automatically defines the program `version` and its
`master_sites` for us.
Each Portgroup will define a different set of variables for you, so check
and make sure to include the required fields.

Now to define the rest of the fields:

```
revision            0
categories          www devel
platforms           darwin
license             Apache-2
maintainers         {@kritr gmail.com:krdevmail} openmaintainer
description         A command line interface for the DigitalOcean API
long_description    ${description}
```

Revision 0 means this is the 0th revision of this specific version of the package.

Categories should be listed in the order of priority, starting with the one
defined as the parent directory for this port.

By default the platform is set to `darwin` for OSX, other common ones include
`linux` and `freebsd`.

The license should be set to the license of the original package.

Maintainers should be set to you. The format for this should be

```
{username domain:email}
```

It's setup this way to prevent unwanted spam for bots searching the internet for
email looking strings.

I've also added the `openmaintainer` tag, which specifies that this port can
be updated by other people for minor changes such as version bumps. I'll still
be tagged on the pull request, but I won't be needed to approve the change.

The description is simply the description for the package.
Since the description is enough and the Portfile is Tcl, it's easy to assign
the long_description variable to description.

The next step is to define the package checksums. Because the Port is fetching
files from the internet, it's important these files stay the same and aren't
hijacked. Checking the checksum after a download prevents corrupt or malicious
files from being installed.

By reindexing the custom ports directory, MacPorts can help in acquiring the checksum.

```
cd /opt/custom
portindex
```

Now that MacPorts knows that the port exists,
Running the following command will acquire the suggested checksums.

```
port -d checksum doctl
```

the output will contain a set of lines akin to the following:

```
The correct checksum line may be:
checksums           rmd160  689b390b734667042d2b1af2bc1ad875d147eafc \
                    sha256  654bc9b7194fc9c12271e33cdcf4520c97ddb31091b16a5a9fbaadb83907e036 \
                    size    15653842
```

Simply copy those last 3 lines back into the Portfile. Saving the Portfile
and rerunning `port -d checksum doctl` should return with a series of successes.

The last couple steps are very port specific. In my case, this port depends on
github for building the package.
So I'll add the following.

```
depends_build-append port:git
```

In addition I need to set some Go environment variables for during the build and to
shift the build target directory.

```
build.env-append    GO111MODULE=on
build.target        ${worksrcpath}/cmd/${name}
```

In those lines you'll notice `${worksrcpath}` being used. This is the directory
MacPorts downloads the build files to. It varies as MacPorts creates its own
directories to build ports in, which is why it's important to use the globally
configured variables to reference paths.
A full list of globals can be found [here](https://guide.macports.org/chunked/reference.variables.html).

Last but not at least is the install command.

```
destroot {
    xinstall -m 0755 ${worksrcpath}/${name} ${destroot}${prefix}/bin
}
```

Destroot is one of the commands that will run after the build has completed.
It's one of the many [phases in the
build](https://guide.macports.org/chunked/reference.phases.html). MacPorts does
an intermediate staging of the build during this phase of the installation,
which is why `xinstall` command is run, to place the final binary file
in the \$destroot.

I specified permissions `0755` on the file, and place the binary file sourced from
`${worksrcpath}/${name}` into `${destroot}${prefix}/bin` which is the
standard binary install directory for MacPorts.

I should note `xinstall` is a handly extension to copy files and create
directories for use. For a full list of useful tcl commands in MacPorts, check
out <https://guide.macports.org/chunked/reference.tcl-extensions.html>.

And that's all the work that's needed to define a MacPorts file!

But... if you're feeling frisky. There's also the concept of variants. These can
be used to define variations of your default build.
In my case I define a variant for zsh_completion.

```
variant zsh_completion description {Install zsh completion} {
    depends_run-append path:${prefix}/bin/zsh:zsh
    post-destroot {
        set site-functions ${destroot}${prefix}/share/zsh/site-functions
        xinstall -d ${site-functions}
        set completion-file [open ${site-functions}/_${name} "w"]
        puts ${completion-file} [exec ${destroot}${prefix}/bin/${name} completion zsh]
        close ${completion-file}
    }
}
```

While not going into as much detail, this creates a variant called
zsh_completion with the description "Install zsh completion".
Then it sets zsh as a dependencies, and tells MacPorts to run a series of
functions that create the directory for installing the zsh completion
files, run a command to return the zsh completion file, and write it out to an
actual file within the install directory.

The end result for this port looked like this:

```
# -*- coding: utf-8; mode: tcl; tab-width: 4; indent-tabs-mode: nil; c-basic-offset: 4 -*- vim:fenc=utf-8:ft=tcl:et:sw=4:ts=4:sts=4

PortSystem          1.0
PortGroup           golang 1.0

go.setup            github.com/digitalocean/doctl 1.20.1 v
revision            0

checksums           rmd160  689b390b734667042d2b1af2bc1ad875d147eafc \
                    sha256  654bc9b7194fc9c12271e33cdcf4520c97ddb31091b16a5a9fbaadb83907e036 \
                    size    15653842

categories          www devel
platforms           darwin
license             Apache-2
maintainers         {@kritr gmail.com:krdevmail} openmaintainer
description         A command line interface for the DigitalOcean API
long_description    ${description}

depends_build-append port:git

build.env-append    GO111MODULE=on
build.target        ${worksrcpath}/cmd/${name}

destroot {
    xinstall -m 0755 ${worksrcpath}/${name} ${destroot}${prefix}/bin
}

variant bash_completion {
    depends_run-append path:etc/bash_completion:bash-completion
    post-destroot {
        set completions_path ${destroot}${prefix}/share/bash-completion/completions
        xinstall -d ${completions_path}
        set completion-file [open ${completions_path}/${name} "w"]
        puts ${completion-file} [exec ${destroot}${prefix}/bin/${name} completion bash]
        close ${completion-file}
    }
}

variant zsh_completion description {Install zsh completion} {
    depends_run-append path:${prefix}/bin/zsh:zsh
    post-destroot {
        set site-functions ${destroot}${prefix}/share/zsh/site-functions
        xinstall -d ${site-functions}
        set completion-file [open ${site-functions}/_${name} "w"]
        puts ${completion-file} [exec ${destroot}${prefix}/bin/${name} completion zsh]
        close ${completion-file}
    }
}

```

Running port install should now result in our package being correctly installed.

```
> port install doctl
```

## Publishing the Port

Finally it's publishing time.
Detailed instruction for this can be found
<https://guide.macports.org/chunked/project.contributing.html>.

I'll simply do my best to give a rundown of the process.

1. Make sure the port doesn't already exist
2. Run port lint on the port

```
> port lint --nitpick doctl
```

3. Fix any changes that need to be made.
4. Fork the [MacPorts Repository on
   Github](https://github.com/macports/macports-ports)
5. Add the port's folder to the existing category directory in the fork
6. Make one detailed commit of your change (or simply rebase and squash all your
   commits with a simple title `doctl: new port`)
7. Make a Pull Request on Github and follow instructions appropriately
8. Wait to hear back from a maintainer
9. Fix any small changes they request
10. Celebrate your accepted pull request !

![Pull Request Accepted](2019-06-30-developing-and-contributing-a-macport/accepted.png)

## Upgrading a Port

While I could cover this as well, the folks at MacPorts have solid documentation
in this area already, that I shall link <https://trac.macports.org/wiki/howto/Upgrade>.

## Conclusion

And so the doctl port is now available on MacPorts.

I myself needed doctl for my own uses but contributing a Portfile
or any other package manager submission is a good way
to get involved in the open source community and helps you understand the
intracacies of building different projects. It's also really convenient for the
hundreds of other developers that will probably end up using your work at some point
in time.

In time we may get down to fewer package managers with more unified tooling, but
for now, we'll keep chipping away at the endless supply of new software to
package.
