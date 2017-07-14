# docker centeos-repo mirror

This docker image can be startet as a container
and it can serve as a replacment for the standard
http://mirrorlist.centos.org

When there is a mirror-query then it will answer
with an URL pointing back to the docker container.

    ORIG: http://mirrorlist.centos.org/?release=7&repo=os&arch=x86_64
    HERE: http://172.22.0.2/?release=7&repo=os&arch=x86_64
    Answered Mirrorlist:
       http://172.22.0.2/7/os/x86_64

In that way it can serve as an endpoint for another
container requiring yum packages from the central
operation system install repositories.

## building the image

There are two ways (1) do sync to local disk and then
build an image as a copy or (2) do start a container
and use wget-mirror to build the copy.

     # Variant 1
     make sync
     make build
     make check

     # Variant 2
     make 7.3
     make check


# Tipps and Tricks

Probably you will not be able to build the images.

That's because docker has a so called "baseimage"
that is the source of all "image" snapshots. No
image can be bigger than the baseimage. And the
baseimage has a default size of 10GB.

Edit /etc/sysconfig/docker and increase that like

    DOCKER_OPTS="--storage-opt dm.basesize=30G"

Theoretically one can just do "service docker restart"
but in reality some versions needed the old baseimage
to be removed from disk so that it is allocated again.
