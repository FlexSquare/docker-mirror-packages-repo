# docker mirror packages repo

These docker images can be started as a container
and it can serve as a replacement for the standard
packages repo of the operating system.

Tested repos are

 * localhost:5000/mirror-packages/**centos-repo:7.3.1611** _(17.3GB)_
 * localhost:5000/mirror-packages/**centos-repo:7.4.1708** _(13.2GB)_
 * localhost:5000/mirror-packages/**centos-repo:7.5.1804** _(14.7GB)_
 * localhost:5000/mirror-packages/**centos-repo:7.7.1908** _(14.6GB)_
 * localhost:5000/mirror-packages/**opensuse-repo:42.3**   _(38.6GB)_
 * localhost:5000/mirror-packages/**opensuse-repo:15.0**   _(48.9GB)_
 * localhost:5000/mirror-packages/**opensuse-repo:15.1**   _(54.6GB)_
 * localhost:5000/mirror-packages/**ubuntu-repo:16.04**    _(74.7GB)_
 * localhost:5000/mirror-packages/**ubuntu-repo:18.04**    _(19.5GB)_
 * localhost:5000/mirror-packages/**ubuntu-repo/universe:16.04** _(186GB)_
 * localhost:5000/mirror-packages/**ubuntu-repo/updates:16.04** _(74.7GB)_
 * localhost:5000/mirror-packages/**ubuntu-repo/updates:18.04** _(19.5GB)_

Remember that the internet package repos contain updates for the last
distribution release, so they do grow in size over time. As such the
image sizes (as shown above) may differ after each sync.

## BACKGROUND

Consider that you have a Dockerfile with

    FROM centos:7.5.1804
    RUN yum install -y httpd httpd-tools

This will reach out to the internet downloading 20+ rpm packages.

It would be better to have a local mirror of the default yum 
repositories for centos. When using a docker environment you 
can reassign the systems' package repo URL to a local container 
that can answer the yum download requests.

That mode is not only faster but you can also commit the yum repo
mirror in your binary repository. New updates on the external 
internet yum repository will not suddenly change rebuild results. 
This has been extensively used in testing docker-related tools 
where the result image was dropped right away after testing it.

## CENTOS

Mimics the default URL of http://mirrorlist.centos.org

The default package repositories in CentOS look like this:

    # /etc/yum.repos.d/CentOS-Base.repo
    [base]
    name=CentOS-$releasever - Base
    mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os
    #baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/

So instead of pointing directly to the storage host "mirror.centos.org" 
there is a reference to a web service on "mirrorlist.centos.org" that 
will return a list of very different storage host options.

The provided "centos-repo" image is able to answer mirrorlist requests
on its port 80/http. It will simply return a single URL pointing back
to itself. A storage path is attached to serve as the download baseurl.

Example: with --add-host "mirrorlist.centos.org:172.22.0.2"

    REQUEST: http://mirrorlist.centos.org/?release=7&repo=os&arch=x86_64
    REALLLY: http://172.22.0.2/?release=7&repo=os&arch=x86_64
    Answered Mirrorlist:
       http://172.22.0.2/7/os/x86_64

The provided "centos-repo" image will furthermore answer download requests
for yum packages on 80/http - simply the url path is mapped to a local path
in the container and a sendfile will return the content. These are mostly 
`*.rpm` packages as well as some package index files.

For more information check the [centos-repo mirror info](./centos-repo-mirror.info.md)

Currently tested are

     make centos-7.5   # really centos-7.5.1804
     make centos-7.4   # really centos-7.4.1708
     make centos-7.3   # really centos-7.3.1611

## OPENSUSE

Mimics the default URL of http://download.opensuse.org

Unlike for centos there is no mirrorlist involved here. All the default 
zypper targets use that single URL. The package indexes are served at 
the same path locations.

The provided "opensuse-repo" image will answer download requests 
for zypper packages on on 80/http - simply the url path is 
mapped to a local path in the container and a sendfile will 
return the content. These are mostly `*.rpm` packages as well
as some package index files.

For mor information check the [opensuse-repo mirror info](./opensuse-repo-mirror.info.md)

Currently tested are

     make opensuse-15.0
     make opensuse-42.3
     make opensuse-42.2

## UBUNTU

Mimics the default URL of http://archive.ubuntu.com

Most of the default apt-get targets use that URL. The
only exception is http://security.ubuntu.com for getting
update packages.

The provided "ubuntu-repo" image will answer download requests 
for ubuntu packages on on 80/http - simply the url path 
is mapped to a local path in the container and a sendfile 
will return the content. These are mostly `*.deb` packages 
as well as some package index files.

For more information check the [ubuntu-repo mirror info](./ubuntu-repo-mirror.info.md)

Currently tested are

     make ubuntu-18.10
     make ubuntu-16.04

## IMPLEMENTATION

Every package repo has a different way to store data in its
file tree. In general it is not a good idea to just rsync
all of it. Sure you can but...

...for example Ubuntu uses one host for all distro versions.
All `*.deb` packages for all versions are stored in a single
`$host:/pool/` subdirectory. The deb packages for a single
version are listed in a `$host:/dists/$version/Packages.gz` 
file. The distro version is not given by number (16.04) but 
by code name (xenial).

The makefiles in this project know about that - so the rsync
will first download the `Packages.gz` files, in order to
unpack/filter them into a list of `/pool/x/y/*.deb` paths
in a temporary file. Then `rsync --files-from=pool-files.tmp` 
will do the rest.

When the rsync is complete, only a little web service script
is added on top of building the image. So when the image is
started as a container, it does accept requests on 80/http
serving the rsync'ed files back.

For tips and tricks please see [PROBLEMS FAQ](./PROBLEMS.md).
