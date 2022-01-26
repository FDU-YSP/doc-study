Docker可以通过从`Dockerfile`读取指令操作然后自动地构建镜像。Docker是一个文本文档，里面包含了用户能够调用的每一行的command命令并集成到一个镜像中。用户可以使用`docker build`命令创建一个自动化构建任务，构建任务会执行所有服务性的命令行指令。

这页文档主要是用来描述和解释用户可以在`Dockerfile`. 当你完成这页文档的阅读，可以参考这个 [Dockerfile最佳实践](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/) 继续学习。

### 用法

[docker build](https://docs.docker.com/engine/reference/commandline/build/) 的构建命令是从定义好的`Dockerfile`和上下文环境来构建容器镜像。构建中的上下文环境(context)主要是指一些文件的集合，文件的指向的可能是**直接本地文件系统路径**或者是**Git repo的URL链接**。

镜像构建中的上下文会被递归处理。所以，文件路径可以包含子目录，或者URL链接里面包含子模块链接。下面的这个example就展示了一个build命令，该命令使用了当前目录(`.`)作为构建的上下文环境。

```shell
$ docker build .

Sending build context to Docker daemon 6.51MB
...
```

构建过程的是由Docker daemon具体执行，而不是CLI (CLI只是发送构建消息给Docker daemon的)。构建过程所做的第一件事是将整个上下文（递归）发送到守护进程。 在大多数情况下，最好从一个空目录作为上下文开始，并将 Dockerfile 保存在该目录中。 仅添加构建 Dockerfile 所需的文件。

>警告
>
>不要使用根目录`/`来作为你镜像构建上下文环境的`PATH`，因为它会导致构建将硬盘驱动器的全部内容传输到 Docker 守护程序。

如果要在构建的上下文中使用文件，`Dockerfile`中应该以指令的方式在文件中定义出来。举个例子，一个`COPY`指令。为了提高镜像构建的性能，可以通过添加`.dockerignore`文件忽略一些文件和目录。如果你想了解更多如何[创建一个`.dockerignore`](https://docs.docker.com/engine/reference/builder/#dockerignore-file)文件，可以参考链接的文档。

传统上，`Dockerfile` 被称为 `Dockerfile` (严格的文件命名) 并且位于上下文的根目录中。 您可以使用带有 `docker build` 的 `-f` 参数标志来指向文件系统中任何位置的 Dockerfile。

```shell
$ docker build -f /path/to/a/Dockerfile .
```

也可以指定保存镜像的存储库和标记：

```shell
$ docker build -t shykes/myapp .
```

如果要在构建后将镜像标记到多个存储库中，请在运行构建命令时添加多个 -t 参数：

```shell
$ docker build -t shykes/myapp:1.0.2 -t shykes/myapp:latest .
```

在 Docker 守护进程运行 Dockerfile 中的指令之前，它会对 Dockerfile 进行初步验证，如果语法不正确则返回错误。

Docker 守护进程一个接一个地运行 Dockerfile 中的指令，如有必要，将每条指令的结果提交到新镜像，最后输出新镜像的 ID。 Docker 守护进程将自动清理发送的上下文。请注意，每条指令都是独立运行的，并会创建一个新镜像 - 因此 `RUN cd /tmp` 不会对下一条指令产生任何影响。

只要有可能，Docker 就会使用构建缓存来显着加速 docker 构建过程。 这由控制台输出的 `CACHED` 消息来指示是否用到了cache。 （有关更多信息，请参阅[Dockerfile最佳实践](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/)。

```shell
 $ docker build -t svendowideit/ambassador .
 
 [+] Building 0.7s (6/6) FINISHED
 => [internal] load build definition from Dockerfile                       0.1s
 => => transferring dockerfile: 286B                                       0.0s
 => [internal] load .dockerignore                                          0.1s
 => => transferring context: 2B                                            0.0s
 => [internal] load metadata for docker.io/library/alpine:3.2              0.4s
 => CACHED [1/2] FROM docker.io/library/alpine:3.2@sha256:e9a2035f9d0d7ce  0.0s
 => CACHED [2/2] RUN apk add --no-cache socat                              0.0s
 => exporting to image                                                     0.0s
 => => exporting layers                                                    0.0s
 => => writing image sha256:1affb80ca37018ac12067fa2af38cc5bcc2a8f09963de  0.0s
 => => naming to docker.io/svendowideit/ambassador                         0.0s
```

默认情况下，镜像构建cache基于您正在构建的机器上先前构建镜像后已经缓存cache的结果(本地有需要的镜像层cache，就用本地cache的；没有，就只能自行build)。 --cache-from 选项还允许您使用通过映像注册表分发的构建缓存，请参阅 `docker build` 命令参考中的[指定外部缓存源](https://docs.docker.com/engine/reference/commandline/build/#specifying-external-cache-sources)部分。

完成你的容器镜像构建后，就可以开始使用 `docker scan` [扫描](https://docs.docker.com/engine/scan/)您的镜像，并将镜像推送到 [Docker Hub](https://docs.docker.com/docker-hub/repos/)。



### buildkit (moby/buildkit)

从18.09版本开始，Docker支持一种新的后端来执行你的镜像构建，这个新的后端是 [moby/buildkit](https://github.com/moby/buildkit)项目。比于旧的实现，这个后端构建工具提供了非常多的新特性，列举如下：

- 检测并跳过执行未使用的构建阶段
- 并行构建独立的构建阶段
- 在构建之间仅增量传输构建上下文中更改的文件
- 在构建上下文中检测并跳过传输未使用的文件
- Use external Dockerfile implementations with many new features
- 使用外部 Dockerfile 实现了许多新特性
- 避免 API 的其余部分（中间镜像和容器）产生副作用
- 优先考虑构建缓存以进行自动缩减构建

为了使用这个构建工具，你需要在调用`docker build`前给CLI设置`DOCKER_BUILDKIT=1` 的环境变量。

学习用于基于 BuildKit 镜像构建的 Dockerfile 语法学习可以[参阅文档](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/experimental.md))。

