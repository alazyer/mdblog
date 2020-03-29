# Docker多架构构建支持

## 问题引入

默认的，使用镜像创建一个容器时，该镜像必须与 Docker 宿主机系统架构一致（Windows、macOS 除外，其使用了 binfmt_misc 提供了多种架构支持，在 Windows、macOS 系统上 (x86_64) 可以运行 arm 等其他架构的镜像），例如 Linux x86_64 架构的系统中只能使用 Linux x86_64 的镜像创建容器。

例如我们在 Linux x86_64 中构建一个`whoami-alphine`镜像。

```text
FROM alpine
CMD whoami
```

构建镜像后推送到 Docker Hub，之后如果我们尝试在树莓派 Linux arm64v8 中使用这个镜像，会发现这个镜像根本获取不到。

`$ docker run -it --rm whoami-alphine`

这是因为当执行`docker run`或者`docker pull`时，docker engine会基于宿主机的操作系统和架构来选择拉取的镜像。

因为我们是在Linux x86_64机器上构建的，因而，想找一个支持Linux arm64v8架构的镜像就找不到了。

## 解决方法

一般最容易想到的方法就是在镜像名字中加入支持的架构，比如支持amd64架构的镜像名字改为`whoami-alphine-amd64`，支持arm64v8架构的镜像命名为`whoami-alphine-arm64v8`。

这样在不同架构下只需要声明使用对应的镜像就可以。

Docker通过引入`Docker Manifest`解决了使用相同镜像名称支持不同架构的问题。`golang:alphine`就通过Docker manifest声明了多架构支持

```bash
$ docker manifest inspect golang:alpine
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1365,
         "digest": "sha256:5e28ac423243b187f464d635bcfe1e909f4a31c6c8bce51d0db0a1062bec9e16",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1365,
         "digest": "sha256:2945c46e26c9787da884b4065d1de64cf93a3b81ead1b949843dda1fcd458bae",
         "platform": {
            "architecture": "arm",
            "os": "linux",
            "variant": "v7"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1365,
         "digest": "sha256:87fff60114fd3402d0c1a7ddf1eea1ded658f171749b57dc782fd33ee2d47b2d",
         "platform": {
            "architecture": "arm64",
            "os": "linux",
            "variant": "v8"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1365,
         "digest": "sha256:607b43f1d91144f82a9433764e85eb3ccf83f73569552a49bc9788c31b4338de",
         "platform": {
            "architecture": "386",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1365,
         "digest": "sha256:25ead0e21ed5e246ce31e274b98c09aaf548606788ef28eaf375dc8525064314",
         "platform": {
            "architecture": "ppc64le",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 1365,
         "digest": "sha256:69f5907fa93ea591175b2c688673775378ed861eeb687776669a48692bb9754d",
         "platform": {
            "architecture": "s390x",
            "os": "linux"
         }
      }
   ]
}

可以看出 manifest 列表中包含了不同系统架构所对应的镜像 digest 值，这样 Docker 就可以在不同的架构中使用相同的 manifest (例如 golang:alpine) 获取对应的镜像。

当然也可以在Linux x84_64的机器上基于arm64v8版本的镜像生成容器，但是会报错（一般是exec错误）

```bash
$ docker run golang@sha256:87fff60114fd3402d0c1a7ddf1eea1ded658f171749b57dc782fd33ee2d47b2d go version
```

其实可以通过一些操作，实现在Linux x86_64机器上运行基于arm64v8架构的容器（TODO：参见后续部分）

## Macos上的多架构构建

需要安装Docker Desktop edge版本，这个里面集成了很多工具，方便多架构构建。

> Docker Desktop Edge release comes with a new CLI command called `buildx`.  Buildx allows you to locally (and soon remotely) build multi-arch images, link them together with a manifest file, and push them all to a registry – with a single command.  With the included emulation, you can transparently build more than just native images!  Buildx accomplishes this by adding new builder instances based on `BuildKit`, and leveraging Docker Desktops technology stack to run non-native binaries.

```bash
# list builder
$ docker buildx list
NAME/NODE     DRIVER/ENDPOINT     STATUS      PLATFORMS
default *     docker
  default     default             running     linux/amd64, linux/arm64, linux/arm/v7, linux/arm/v6

# create new builder
$ docker buildx create --name firsttrybuildx

# use new created builder
$ docker buildx use firsttrybuildx
```

既然已经创建出来一个新的builder，我们当然想尝试一下构建多架构的镜像，然后使用一番。

其实要想成功创建支持多架构的镜像，有一个前提条件就是基础镜像必须已经支持多架构，否则不能成功创建多架构镜像。下面的ubuntu基础镜像就支持多架构

Dockerfile

```yaml
FROM ubuntu
RUN apt-get update && apt-get install -y curl
WORKDIR /src
COPY . .
```

```bash
# build multi-arch images
$ docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 -t alazyer/demo:latest --push .
```

如果想看看执行了上一步到底做了那些事情，可以执行

```bash
$ docker buildx imagetools inspect alazyer/demo:latest
```

然后在Macos安装了Docker Desktop Edge release版本后，就可以直接通过sha256来运行其他架构的容器了

```bash
$ docker run --rm ubuntu:18.04@sha256:12b9106d200061c8eb2179c984e63cc17d5a4e7a34f6e9ad03360b8efe492e96 uname -m
aarch64
```

## Linux环境支持buildx

虽然Docker官方提供了在Macos和Windows系统上支持Buildx的Edge Release版本，但是更多的软件开发者还是使用的Linux，因而如何在Linux上支持多架构构建同样显得很重要。

安装尽量新版本的Docker(19.03+), 然后手动下载docker-buildx插件，并放到指定目录中

```bash
# Check version
$ docker version
Client:
 Version:           19.03.6
 API version:       1.40
 Go version:        go1.12.17
 Git commit:        369ce74a3c
 Built:             Fri Feb 28 23:45:43 2020
 OS/Arch:           linux/amd64
 Experimental:      false

Server:
 Engine:
  Version:          19.03.6
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.17
  Git commit:       369ce74a3c
  Built:            Wed Feb 19 01:06:16 2020
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.3.3-0ubuntu1~18.04.1
  GitCommit:
 runc:
  Version:          spec: 1.0.1-dev
  GitCommit:
 docker-init:
  Version:          0.18.0
  GitCommit:

# Get the docker buildx plugin
$ mkdir -p ~/.docker/cli-plugins
$ curl https://github.com/docker/buildx/releases/download/v0.3.1/buildx-v0.3.1.linux-amd64 -o ~/.docker/cli-plugins/docker-buildx

# Check buildx version, permission problems may exist, just chmod 777 ~/.docker/cli-plugins/docker-buildx
$ docker buildx version
```

上面虽然安装了buildx，但是其实还不能构建多架构镜像，需要执行命令，来添加qemu模拟器支持

```bash
$ docker run --rm --privileged docker/binfmt:a7996909642ee92942dcd6cff44b9b95f08dad64

# Check if qemu handlers
$ cat /proc/sys/fs/binfmt_misc/qemu-aarch64
enabled
interpreter /usr/bin/qemu-aarch64
flags: OCF
offset 0
magic 7f454c460201010000000000000000000200b7
mask ffffffffffffff00fffffffffffffffffeffff
```

后续的创建多架构镜像和查看构建出来的Manifest操作就和Macos上的一致了。

## Reference

- https://www.docker.com/blog/multi-arch-images/
- https://www.docker.com/blog/getting-started-with-docker-for-arm-on-linux/
- https://www.stereolabs.com/docs/docker/building-arm-container-on-x86/
- https://yeasy.gitbooks.io/docker_practice/image/manifest.html
