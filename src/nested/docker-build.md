# 空镜像scratch

scratch是一个空镜像，直接docker hub下载是下不了的，只能在Dockerfile中用FROM

```bash
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
[root@localhost hello]# cat Dockerfile
FROM scratch
ADD app /
CMD ["/app"]
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
[root@localhost hello]# docker container run -it hello
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Hello World![root@localhost hello]# 
```

# 多阶段构建
构建镜像和运行镜像分开，有两个好处
1. 镜像很小
2. 源代码分离，安全

```bash
[root@localhost hello]# cat Dockerfile 
FROM golang:alpine as builder
WORKDIR /go/src/
COPY app.go .
RUN go build -o app app.go
FROM alpine:latest as prod
WORKDIR /root/
COPY --from=0 /go/src/app .
CMD ["./app"]
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
```

