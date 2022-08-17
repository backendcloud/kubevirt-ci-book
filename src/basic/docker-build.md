
# 镜像构建上下文（Context）
如果注意，会看到 docker build 命令最后有一个.。.表示当前目录，而 Dockerfile 就在当前目录，因此不少初学者以为这个路径是在指定 Dockerfile 所在路径，这么理解其实是不准确的。如果对应上面的命令格式，你可能会发现，这是在指定上下文路径。那么什么是上下文呢？

首先我们要理解 docker build 的工作原理。Docker 在运行时分为 Docker 引擎（也就是服务端守护进程）和客户端工具。Docker 的引擎提供了一组 REST API，被称为 Docker Remote API，而如 docker 命令这样的客户端工具，则是通过这组 API 与 Docker 引擎交互，从而完成各种功能。因此，虽然表面上我们好像是在本机执行各种 docker 功能，但实际上，一切都是使用的远程调用形式在服务端（Docker 引擎）完成。也因为这种 C/S 设计，让我们操作远程服务器的 Docker 引擎变得轻而易举。

当我们进行镜像构建的时候，并非所有定制都会通过 RUN 指令完成，经常会需要将一些本地文件复制进镜像，比如通过 COPY 指令、ADD 指令等。而 docker build 命令构建镜像，其实并非在本地构建，而是在服务端，也就是 Docker 引擎中构建的。那么在这种客户端/服务端的架构中，如何才能让服务端获得本地文件呢？

这就引入了上下文的概念。当构建的时候，用户会指定构建镜像上下文的路径，docker build 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。如果在 Dockerfile 中这么写：

```bash
COPY ./package.json /app/
```

这并不是要复制执行 docker build 命令所在的目录下的 package.json，也不是复制 Dockerfile 所在目录下的 package.json，而是复制 上下文（context） 目录下的 package.json。

因此，COPY这类指令中的源文件的路径都是相对路径。这也是初学者经常会问的为什么 COPY ../package.json /app 或者 COPY /opt/xxxx /app 无法工作的原因，因为这些路径已经超出了上下文的范围，Docker 引擎无法获得这些位置的文件。如果真的需要那些文件，应该将它们复制到上下文目录中去。

现在就可以理解刚才的命令docker build -t nginx:v3 .中的这个.，实际上是在指定上下文的目录，docker build 命令会将该目录下的内容打包交给 Docker 引擎以帮助构建镜像。

如果观察 docker build 输出，我们其实已经看到了这个发送上下文的过程：

```bash
$ docker build -t nginx:v3 .
Sending build context to Docker daemon 2.048 kB
...
```

理解构建上下文对于镜像构建是很重要的，可以避免犯一些不应该的错误。比如有些初学者在发现 COPY /opt/xxxx /app 不工作后，于是干脆将 Dockerfile 放到了硬盘根目录去构建，结果发现 docker build 执行后，在发送一个几十 GB 的东西，极为缓慢而且很容易构建失败。那是因为这种做法是在让 docker build 打包整个硬盘，这显然是使用错误。

一般来说，应该会将 Dockerfile 置于一个空目录下，或者项目根目录下。如果该目录下没有所需文件，那么应该把所需文件复制一份过来。如果目录下有些东西确实不希望构建时传给 Docker 引擎，那么可以用 .gitignore 一样的语法写一个.dockerignore，该文件是用于剔除不需要作为上下文传递给 Docker 引擎的。

那么为什么会有人误以为 . 是指定 Dockerfile 所在目录呢？这是因为在默认情况下，如果不额外指定 Dockerfile 的话，会将上下文目录下的名为 Dockerfile 的文件作为 Dockerfile。

这只是默认行为，实际上 Dockerfile 的文件名并不要求必须为 Dockerfile，而且并不要求必须位于上下文目录中，比如可以用-f ../Dockerfile.php参数指定某个文件作为 Dockerfile。

当然，一般大家习惯性的会使用默认的文件名 Dockerfile，以及会将其置于镜像构建上下文目录中。

# 迁移镜像

Docker 还提供了docker load和docker save命令，用以将镜像保存为一个 tar 文件，然后传输到另一个位置上，再加载进来。这是在没有 Docker Registry 时的做法，现在已经不推荐，镜像迁移应该直接使用 

Docker Registry，无论是直接使用 Docker Hub 还是使用内网私有 Registry 都可以。

使用docker save命令可以将镜像保存为归档文件。比如我们希望保存这个 alpine 镜像。

```bash
$ docker image ls alpine
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
alpine              latest              baa5d63471ea        5 weeks ago         4.803 MB
```

保存镜像的命令为：

```bash
$ docker save alpine | gzip > alpine-latest.tar.gz
```

然后我们将 alpine-latest.tar.gz 文件复制到了到了另一个机器上，可以用下面这个命令加载镜像：

```bash
$ docker load -i alpine-latest.tar.gz
Loaded image: alpine:latest
```

如果我们结合这两个命令以及 ssh 甚至 pv 的话，利用 Linux 强大的管道，我们可以写一个命令完成从一个机器将镜像迁移到另一个机器，并且带进度条的功能：

```bash
docker save <镜像名> | bzip2 | pv | ssh <用户名>@<主机名> 'cat | docker load'
```

# 空镜像scratch

scratch是一个空镜像，直接docker hub下载是下不了的，只能在Dockerfile中用FROM

```bash
# 不用容器，先用go build & run 执行下看下结果
[root@localhost hello]# cat app.go 
package main

import "fmt"

func main(){
    fmt.Printf("Hello World!");
}
[root@localhost hello]# go build app.go 
[root@localhost hello]# ls
app  app.go
[root@localhost hello]# ./app 
Hello World![root@localhost hello]#

# Dockerfile （使用了空镜像scratch）
[root@localhost hello]# cat Dockerfile
FROM scratch
ADD app /
CMD ["/app"]

# 构建镜像
[root@localhost hello]# docker build -t hello .
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
STEP 1/3: FROM scratch
STEP 2/3: ADD app /
--> f40b810d4f1
STEP 3/3: CMD ["/app"]
COMMIT hello
--> 3b05b3e8edb
Successfully tagged localhost/hello:latest
3b05b3e8edb740b20ed133aa65539217efe8cf2e97816b2912992fb8edb73f6b

[root@localhost hello]# docker images
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
REPOSITORY                                   TAG                   IMAGE ID      CREATED        SIZE
localhost/hello                              latest                3b05b3e8edb7  3 seconds ago  1.78 MB
# 用容器跑的结果 和 不用容器，用go build & run 一致
[root@localhost hello]# docker container run -it hello
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Hello World![root@localhost hello]# 
```


# 虚悬镜像

```bash
[root@localhost hello]# docker system df
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
TYPE           TOTAL       ACTIVE      SIZE        RECLAIMABLE
Images         21          5           19.44GB     4.2GB (22%)
Containers     10          4           823B        60B (7%)
Local Volumes  8           7           23.07GB     22.47GB (97%)
[root@localhost hello]# docker images
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
REPOSITORY                                   TAG                   IMAGE ID      CREATED         SIZE
localhost/hello                              2                     a37976309a63  16 seconds ago  7.59 MB
<none>                                       <none>                19be39c07a76  27 seconds ago  340 MB
localhost/hello                              latest                3b05b3e8edb7  19 minutes ago  1.78 MB
<none>                                       <none>                573a101bae9d  24 minutes ago  29.4 kB
quay.io/kubevirtci/k8s-1.21                  2207060141-111fd50    6fa0fcc42f47  31 hours ago    15.2 GB
quay.io/kubevirtci/gocli                     2207060141-111fd50    07874ec90d6e  31 hours ago    14.5 MB
quay.io/kubevirt/builder                     2206141426-2d31aa580  7ae2a103287f  3 weeks ago     1.98 GB
docker.io/library/golang                     alpine                155ead2e66ca  5 weeks ago     338 MB
docker.io/library/alpine                     latest                e66264b98777  6 weeks ago     5.82 MB
docker.io/kubevirtci/gocli                   latest                094672b6ef8e  21 months ago   12.6 MB
quay.io/libpod/registry                      2.7                   2d4f4b5309b1  2 years ago     26.8 MB
docker.io/backendcloud/example-hook-sidecar  mybuild5              0581e298f5bb  52 years ago    211 MB
docker.io/backendcloud/winrmcli              mybuild4              91ca4390f858  52 years ago    199 MB
docker.io/backendcloud/winrmcli              mybuild5              91ca4390f858  52 years ago    199 MB
docker.io/backendcloud/example-hook-sidecar  mybuild6              0581e298f5bb  52 years ago    211 MB
```

上面的镜像列表中，还可以看到一个特殊的镜像，这个镜像既没有仓库名，也没有标签，均为 `<none>`。

这个镜像原本是有镜像名和标签的，随着官方镜像维护，发布了新版本后，重新 docker pull 时，这个镜像名被转移到了新下载的镜像身上，而旧的镜像上的这个名称则被取消，从而成为了 `<none>`。除了 docker pull 可能导致这种情况，docker build 也同样可以导致这种现象。由于新旧镜像同名，旧镜像名称被取消，从而出现仓库名、标签均为 `<none>` 的镜像。这类无标签镜像也被称为 虚悬镜像(dangling image) ，可以用下面的命令专门显示这类镜像：

```bash
[root@localhost hello]# docker images ls -f dangling=true
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Error: cannot specify an image and a filter(s)

#注意：不是images，是image，和docker images命令不同
[root@localhost hello]# docker image ls -f dangling=true
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
REPOSITORY  TAG         IMAGE ID      CREATED       SIZE
<none>      <none>      19be39c07a76  16 hours ago  340 MB
<none>      <none>      573a101bae9d  16 hours ago  29.4 kB
```

一般来说，虚悬镜像已经失去了存在的价值，是可以随意删除的，可以用下面的命令删除。

```bash
[root@localhost hello]# docker image prune
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
WARNING! This command removes all dangling images.
Are you sure you want to continue? [y/N] y
19be39c07a766503753c91261bfd2dd786f536db69402ccbc12bdaea054e3047
6220efe05008da6279e95d7a377993b035e59833dfd75f66184a24fcf2c8a85d
906f367739194e499b1baf1459fa7c0eca2cf8249fc466e671315c1f2a13b369
[root@localhost hello]# docker image ls -f dangling=true
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
REPOSITORY  TAG         IMAGE ID      CREATED       SIZE
<none>      <none>      573a101bae9d  16 hours ago  29.4 kB
[root@localhost hello]# docker image ls -f dangling=true
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
REPOSITORY  TAG         IMAGE ID      CREATED       SIZE
<none>      <none>      573a101bae9d  16 hours ago  29.4 kB
[root@localhost hello]# docker images
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
REPOSITORY                                   TAG                   IMAGE ID      CREATED        SIZE
localhost/hello                              2                     a37976309a63  16 hours ago   7.59 MB
localhost/hello                              latest                3b05b3e8edb7  16 hours ago   1.78 MB
<none>                                       <none>                573a101bae9d  16 hours ago   29.4 kB
quay.io/kubevirtci/k8s-1.21                  2207060141-111fd50    6fa0fcc42f47  47 hours ago   15.2 GB
quay.io/kubevirtci/gocli                     2207060141-111fd50    07874ec90d6e  2 days ago     14.5 MB
quay.io/kubevirt/builder                     2206141426-2d31aa580  7ae2a103287f  3 weeks ago    1.98 GB
docker.io/library/golang                     alpine                155ead2e66ca  5 weeks ago    338 MB
docker.io/library/alpine                     latest                e66264b98777  6 weeks ago    5.82 MB
docker.io/kubevirtci/gocli                   latest                094672b6ef8e  21 months ago  12.6 MB
quay.io/libpod/registry                      2.7                   2d4f4b5309b1  2 years ago    26.8 MB
docker.io/backendcloud/winrmcli              mybuild4              91ca4390f858  52 years ago   199 MB
docker.io/backendcloud/winrmcli              mybuild5              91ca4390f858  52 years ago   199 MB
docker.io/backendcloud/example-hook-sidecar  mybuild6              0581e298f5bb  52 years ago   211 MB
docker.io/backendcloud/example-hook-sidecar  mybuild5              0581e298f5bb  52 years ago   211 MB

# 发现还有一个虚悬镜像，再删了一遍，还是没删掉，用docker rmi删除，提示有容器在使用该镜像。docker image prune这里做得不友好，一点warning信息没有。
[root@localhost hello]# docker image prune
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
WARNING! This command removes all dangling images.
Are you sure you want to continue? [y/N] y
[root@localhost hello]# docker images
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
REPOSITORY                                   TAG                   IMAGE ID      CREATED        SIZE
localhost/hello                              2                     a37976309a63  16 hours ago   7.59 MB
localhost/hello                              latest                3b05b3e8edb7  16 hours ago   1.78 MB
<none>                                       <none>                573a101bae9d  16 hours ago   29.4 kB
quay.io/kubevirtci/k8s-1.21                  2207060141-111fd50    6fa0fcc42f47  47 hours ago   15.2 GB
quay.io/kubevirtci/gocli                     2207060141-111fd50    07874ec90d6e  2 days ago     14.5 MB
quay.io/kubevirt/builder                     2206141426-2d31aa580  7ae2a103287f  3 weeks ago    1.98 GB
docker.io/library/golang                     alpine                155ead2e66ca  5 weeks ago    338 MB
docker.io/library/alpine                     latest                e66264b98777  6 weeks ago    5.82 MB
docker.io/kubevirtci/gocli                   latest                094672b6ef8e  21 months ago  12.6 MB
quay.io/libpod/registry                      2.7                   2d4f4b5309b1  2 years ago    26.8 MB
docker.io/backendcloud/winrmcli              mybuild4              91ca4390f858  52 years ago   199 MB
docker.io/backendcloud/winrmcli              mybuild5              91ca4390f858  52 years ago   199 MB
docker.io/backendcloud/example-hook-sidecar  mybuild6              0581e298f5bb  52 years ago   211 MB
docker.io/backendcloud/example-hook-sidecar  mybuild5              0581e298f5bb  52 years ago   211 MB
[root@localhost hello]# docker rmi 573a101bae9d
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Error: Image used by 095dff22c8f728ba34ead410db4c8a61b94f4e1119d7b9ec8c0d53140998e158: image is in use by a container
```

# 多阶段构建
构建镜像和运行镜像分开，有两个好处
1. 镜像很小
2. 源代码分离，安全

```bash
# 多阶段构建Dockerfile
[root@localhost hello]# cat Dockerfile 
FROM golang:alpine as builder
WORKDIR /go/src/
COPY app.go .
RUN go build -o app app.go
FROM alpine:latest as prod
WORKDIR /root/
COPY --from=0 /go/src/app .
CMD ["./app"]

# 生成镜像（第一个跑完会被自动销毁，只保留第二个镜像）
[root@localhost hello]# docker build -t hello:2 .
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
[1/2] STEP 1/4: FROM golang:alpine AS builder
[1/2] STEP 2/4: WORKDIR /go/src/
--> Using cache 906f367739194e499b1baf1459fa7c0eca2cf8249fc466e671315c1f2a13b369
--> 906f3677391
[1/2] STEP 3/4: COPY app.go .
--> Using cache 6220efe05008da6279e95d7a377993b035e59833dfd75f66184a24fcf2c8a85d
--> 6220efe0500
[1/2] STEP 4/4: RUN go build -o app app.go
--> 19be39c07a7
[2/2] STEP 1/4: FROM alpine:latest AS prod
Resolved "alpine" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Trying to pull docker.io/library/alpine:latest...
Getting image source signatures
Copying blob 2408cc74d12b skipped: already exists  
Copying config e66264b987 done  
Writing manifest to image destination
Storing signatures
[2/2] STEP 2/4: WORKDIR /root/
--> 9f975764a6d
[2/2] STEP 3/4: COPY --from=0 /go/src/app .
--> 22fd24c9b9d
[2/2] STEP 4/4: CMD ["./app"]
[2/2] COMMIT hello:2
--> a37976309a6
Successfully tagged localhost/hello:2
a37976309a6375e3107bf0c89cc373d6c0b953b6596238006aabf0ac3bcfa762

[root@localhost hello]# docker images
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
REPOSITORY                                   TAG                   IMAGE ID      CREATED         SIZE
localhost/hello                              2                     a37976309a63  16 seconds ago  7.59 MB

[root@localhost hello]# docker container run -it hello:2
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Hello World![root@localhost hello]# 
# 镜像运行符合预期（将镜像从alpine换成上一小节的scratch空镜像也可以）
```
