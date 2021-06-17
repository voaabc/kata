---
layout: post
title: What is a Container Image?
categories: Docker
tags: Docker
---

* content
{:toc}




### Container Image

A container image is a tar file containing tar files. Each of the tar file is a layer. Once all tar files have been extract into the same location then you have the container's filesystem.
> 容器映像是包含tar文件的tar文件。每个tar文件都是一个层。一旦所有tar文件都被提取到同一个位置，那么就有了容器的文件系统。

This can be explored via Docker. Pull the layers onto your local system.
> 这可以通过Docker进行探索。将图层拉到本地系统上。

`docker pull redis:3.2.11-alpine`

Export the image into the raw tar format.
> 将映像导出为原始tar格式。

`docker save redis:3.2.11-alpine > redis.tar`

Extract to the disk
> 提取到磁盘

`tar -xvf redis.tar`

All of the layer tar files are now viewable.
> 所有的层tar文件现在都是可见的。

`ls`

The image also includes metadata about the image, such as version information and tag names.
> 镜像还包括有关镜像的元数据，如版本信息和标记名称。

`cat repositories`

`cat manifest.json`

Extracting a layer will show you which files that layer provides.
> 提取一个层将显示该层提供哪些文件。

`tar -xvf da2a73e79c2ccb87834d7ce3e43d274a750177fe6527ea3f8492d08d3bb0123c/layer.tar`


### Creating Empty Image
As an image is just a tar file, an empty image can be created using the command below.
> 由于映像只是一个tar文件，所以可以使用下面的命令创建一个空映像。

`tar cv --files-from /dev/null | docker import - empty`

By importing the tar, the additional metadata will be created.
> 通过导入tar，将创建额外的元数据。

`docker images`

However, as the container doesn't contain anything, it can't start a process.
> 但是，由于容器不包含任何内容，因此无法启动进程。


### Creating Image without Dockerfile
The previous idea of importing a Tar file can be extended to create an entire image from scratch.
> 可以扩展前面导入Tar文件的思想，从头创建整个映像。

To start, we'll use BusyBox as the base. This will provide us with the foundation linux commands. This is defined as the rootfs. The rootfs is...
> 首先，我们将使用BusyBox作为基础。这将为我们提供基础linux命令。这被定义为rootfs。rootfs是...

Docker provide a script to download the BusyBox rootfs at
> Docker提供了一个脚本来下载BusyBox rootfs

https://github.com/moby/moby/blob/a575b0b1384b2ba89b79cbd7e770fbeb616758b3/contrib/mkimage/busybox-static

`curl -LO https://raw.githubusercontent.com/moby/moby/a575b0b1384b2ba89b79cbd7e770fbeb616758b3/contrib/mkimage/busybox-static && chmod +x busybox-static`

`./busybox-static busybox`

Running the script will download the rootfs and main binaries.
> 运行该脚本将下载rootfs和主二进制文件。

`ls -lha busybox`

The default Busybox rootfs doesn't include any version information so let's created a file.
> 默认BusyBox Rootfs不包含任何版本信息，因此让我们创建一个文件。

`echo KatacodaPrivateBuild > busybox/release`

As before, the directory can be converted into a tar and automatically imported into Docker as an image.
> 与前面一样，该目录可以转换为tar，并自动作为镜像导入到Docker中。

`tar -C busybox -c . | docker import - busybox`

This can now be launched as a container.
> 它现在可以作为一个容器启动。

`docker run busybox cat /release`


```sh
$ docker pull redis:3.2.11-alpine
3.2.11-alpine: Pulling from library/redis
ff3a5c916c92: Pull complete 
aae70a2e6027: Pull complete 
87c655da471c: Pull complete 
bc3141806bdc: Pull complete 
53616fb426d9: Pull complete 
9791c5883c6a: Pull complete 
Digest: sha256:ebf1948b84dcaaa0f8a2849cce6f2548edb8862e2829e3e7d9e4cd5a324fb3b7
Status: Downloaded newer image for redis:3.2.11-alpine
$ docker save redis:3.2.11-alpine > redis.tar
$ tar -xvf redis.tar
46a2fed8167f5d523f9a9c07f17a7cd151412fed437272b517ee4e46587e5557/
46a2fed8167f5d523f9a9c07f17a7cd151412fed437272b517ee4e46587e5557/VERSION
46a2fed8167f5d523f9a9c07f17a7cd151412fed437272b517ee4e46587e5557/json
46a2fed8167f5d523f9a9c07f17a7cd151412fed437272b517ee4e46587e5557/layer.tar
498654318d0999ce36c7b90901ed8bd8cb63d86837cb101ea1ec9bb092f44e59/
498654318d0999ce36c7b90901ed8bd8cb63d86837cb101ea1ec9bb092f44e59/VERSION
498654318d0999ce36c7b90901ed8bd8cb63d86837cb101ea1ec9bb092f44e59/json
498654318d0999ce36c7b90901ed8bd8cb63d86837cb101ea1ec9bb092f44e59/layer.tar
ad01e7adb4e23f63a0a1a1d258c165d852768fb2e4cc2d9d5e71698e9672093c/
ad01e7adb4e23f63a0a1a1d258c165d852768fb2e4cc2d9d5e71698e9672093c/VERSION
ad01e7adb4e23f63a0a1a1d258c165d852768fb2e4cc2d9d5e71698e9672093c/json
ad01e7adb4e23f63a0a1a1d258c165d852768fb2e4cc2d9d5e71698e9672093c/layer.tar
ca0b6709748d024a67c502558ea88dc8a1f8a858d380f5ddafa1504126a3b018.json
da2a73e79c2ccb87834d7ce3e43d274a750177fe6527ea3f8492d08d3bb0123c/
da2a73e79c2ccb87834d7ce3e43d274a750177fe6527ea3f8492d08d3bb0123c/VERSION
da2a73e79c2ccb87834d7ce3e43d274a750177fe6527ea3f8492d08d3bb0123c/json
da2a73e79c2ccb87834d7ce3e43d274a750177fe6527ea3f8492d08d3bb0123c/layer.tar
db1a23fc1daa8135a1c6c695f7b416a0ac0eb1d8ca873928385a3edaba6ac9a3/
db1a23fc1daa8135a1c6c695f7b416a0ac0eb1d8ca873928385a3edaba6ac9a3/VERSION
db1a23fc1daa8135a1c6c695f7b416a0ac0eb1d8ca873928385a3edaba6ac9a3/json
db1a23fc1daa8135a1c6c695f7b416a0ac0eb1d8ca873928385a3edaba6ac9a3/layer.tar
f07352aa34c241692cae1ce60ade187857d0bffa3a31390867038d46b1e7739c/
f07352aa34c241692cae1ce60ade187857d0bffa3a31390867038d46b1e7739c/VERSION
f07352aa34c241692cae1ce60ade187857d0bffa3a31390867038d46b1e7739c/json
f07352aa34c241692cae1ce60ade187857d0bffa3a31390867038d46b1e7739c/layer.tar
manifest.json
repositories
$ ls
46a2fed8167f5d523f9a9c07f17a7cd151412fed437272b517ee4e46587e5557
498654318d0999ce36c7b90901ed8bd8cb63d86837cb101ea1ec9bb092f44e59
ad01e7adb4e23f63a0a1a1d258c165d852768fb2e4cc2d9d5e71698e9672093c
ca0b6709748d024a67c502558ea88dc8a1f8a858d380f5ddafa1504126a3b018.json
da2a73e79c2ccb87834d7ce3e43d274a750177fe6527ea3f8492d08d3bb0123c
db1a23fc1daa8135a1c6c695f7b416a0ac0eb1d8ca873928385a3edaba6ac9a3
f07352aa34c241692cae1ce60ade187857d0bffa3a31390867038d46b1e7739c
manifest.json
redis.tar
repositories
$ cat repositories
{"redis":{"3.2.11-alpine":"46a2fed8167f5d523f9a9c07f17a7cd151412fed437272b517ee4e46587e5557"}}
$ cat manifest.json
[{"Config":"ca0b6709748d024a67c502558ea88dc8a1f8a858d380f5ddafa1504126a3b018.json","RepoTags":["redis:3.2.11-alpine"],"Layers":["498654318d0999ce36c7b90901ed8bd8cb63d86837cb101ea1ec9bb092f44e59/layer.tar","ad01e7adb4e23f63a0a1a1d258c165d852768fb2e4cc2d9d5e71698e9672093c/layer.tar","da2a73e79c2ccb87834d7ce3e43d274a750177fe6527ea3f8492d08d3bb0123c/layer.tar","db1a23fc1daa8135a1c6c695f7b416a0ac0eb1d8ca873928385a3edaba6ac9a3/layer.tar","f07352aa34c241692cae1ce60ade187857d0bffa3a31390867038d46b1e7739c/layer.tar","46a2fed8167f5d523f9a9c07f17a7cd151412fed437272b517ee4e46587e5557/layer.tar"]}]
$ tar -xvf da2a73e79c2ccb87834d7ce3e43d274a750177fe6527ea3f8492d08d3bb0123c/layer.tar
etc/
etc/apk/
etc/apk/world
lib/
lib/apk/
lib/apk/db/
lib/apk/db/installed
lib/apk/db/lock
lib/apk/db/scripts.tar
lib/apk/db/triggers
sbin/
sbin/su-exec
var/
var/cache/
var/cache/misc/
$ tar cv --files-from /dev/null | docker import - empty
sha256:0181d2bfe2870b9910a9fdea6a651a4a93799e99ec649ad36bc9cde141b25d04
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
empty               latest              0181d2bfe287        18 seconds ago      0B
redis               3.2.11-alpine       ca0b6709748d        3 years ago         20.7MB
$ curl -LO https://raw.githubusercontent.com/moby/moby/a575b0b1384b2ba89b79cbd7e770fbeb616758b3/contrib/mkimage/busybox-static && chmod +x busybox-static
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   782  100   782    0     0   2384      0 --:--:-- --:--:-- --:--:--  2391
$ ./busybox-static busybox
$ ls -lha busybox
total 20K
drwxr-xr-x  5 root root 4.0K Jun 17 02:25 .
drwx------ 15 root root 4.0K Jun 17 02:25 ..
drwxr-xr-x  2 root root 4.0K Jun 17 02:25 bin
drwxr-xr-x  2 root root 4.0K Jun 17 02:25 sbin
drwxr-xr-x  4 root root 4.0K Jun 17 02:25 usr
$ echo KatacodaPrivateBuild > busybox/release
$ tar -C busybox -c . | docker import - busybox
sha256:789a9bfde66bd50bb57d102c08f2bbe40c6fadf9a5cd3de0ace609b9e467a683
$ docker run busybox cat /release
KatacodaPrivateBuild


```