Docker可以通过从`Dockerfile`读取指令操作然后自动地构建镜像。Docker是一个文本文档，里面包含了用户能够调用的每一行的command命令并集成到一个镜像中。用户可以使用`docker build`命令创建一个自动化构建任务，构建任务会执行所有服务性的命令行指令。

这页文档主要是用来描述和解释用户可以在`Dockerfile`. 当你完成这页文档的阅读，可以参考这个 [Dockerfile最佳实践](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/) 继续学习。

### 1. 用法

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



### 2. buildkit (moby/buildkit)

从18.09版本开始，Docker支持一种新的后端来执行你的镜像构建，这个新的后端是 [moby/buildkit](https://github.com/moby/buildkit)项目。比于旧的实现，这个后端构建工具提供了非常多的新特性，列举如下：

- 检测并跳过执行未使用的构建阶段
- 并行构建独立的构建阶段
- 在构建之间仅增量传输构建上下文中更改的文件
- 在构建上下文中检测并跳过传输未使用的文件
- Use external Dockerfile implementations with many new features
- 使用外部 Dockerfile 实现了许多新特性
- 避免 API 的其余部分（中间镜像和容器）产生负面影响
- 优先考虑构建缓存以进行自动缩减构建

为了使用这个构建工具，你需要在调用`docker build`前给CLI设置`DOCKER_BUILDKIT=1` 的环境变量。

学习用于基于 BuildKit 镜像构建的 Dockerfile 语法学习可以[参阅文档](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/experimental.md)。



### 3. 语法格式

接下来是`Dockerfile`的语法格式：

```shell
# Comment
INSTRUCTION arguments
```

`Dockerfile`中的命令大小写不敏感。但是，**约定是大写的**，以便更容易地将它们与参数区分开来。

Docker按照顺序执行`Dockerfile`中的命令。`Dockerfile`必须使用`FROM`命令开头（执行逻辑上的原则）。实际上FROM命令也可能会排在[parser directives](### 4. Parser directives), [注释](https://docs.docker.com/engine/reference/builder/#format),以及全局的[ARGs](https://docs.docker.com/engine/reference/builder/#arg)后面。From命令用来指定使用哪个[基础镜像](https://docs.docker.com/glossary/#parent-image)来构建你的容器镜像。`FROM` 之前只能有一个或多个 `ARG` 指令，这些声明的参数在 `Dockerfile` 的 `FROM` 使用。

Docker会把`#`开头的一行当作注释处理， 除非这一行是合法的[parser directive](https://docs.docker.com/engine/reference/builder/#parser-directives)。行中任何其他位置的`#`标记都被视为参数。 这允许以下语句：

```shell
# Comment
RUN echo 'we are running some # of cool things'
```

注释行在执行 Dockerfile 指令之前被删除，这意味着下面示例中的注释不是由执行 `echo` 命令的 shell 处理的，下面两个示例是等价的：

```shell
RUN echo hello \
# comment
world
```

```shell
RUN echo hello \
world
```

注释中不支持换行符。

>空格使用提醒：
>
>为了向后兼容，注释 (`#`) 和指令（如 `RUN`）之前的前导空格会被忽略，但不鼓励。 在这些情况下不保留前导空格，因此以下示例是等效的：
>
>```shell
>        # this is a comment-line
>    RUN echo hello
>RUN echo world
>```
>
>```shell
># this is a comment-line
>RUN echo hello
>RUN echo world
>```
>
>但是请注意，指令参数中的空格（例如 `RUN` 之后的命令）被保留，因此以下示例将打印 `hello world` 并指定前导空格：
>
>```shell
>RUN echo "\
>     hello\
>     world"
>```

### 4. Parser directives

v是可选的，会影响 `Dockerfile` 中后续行的处理方式。 Parser directives不会将层添加到构建中，并且不会显示为构建步骤。 Parser directives以`#directive=value`的形式写成一种特殊类型的注释。 一个指令只能使用一次。

一旦处理了注释、空行或构建器指令，Docker 就不再寻找解析器指令。 相反，它将任何格式化为解析器指令的内容视为注释，并且不会尝试验证它是否可能是parser directive。 因此，所有parser directive都必须位于 `Dockerfile` 的最顶部。

Parser directives不区分大小写。 但是，约定是小写的。 约定也是在任何解析器指令之后包含一个空行。 解析器指令不支持换行符。

由于这些规则，以下示例均无效：

换行无效：

```shell
# direc \
tive=value
```

出现两次无效：

```shell
# directive=value1
# directive=value2

FROM ImageName
```

如果加在构建器指令之后，就会被视为注释：

```shell
FROM ImageName
# directive=value
```

由于出现在不是解析器指令的注释之后，因此也会被视为注释：

```shell
# About my dockerfile
# directive=value
FROM ImageName
```

由于未被识别，未知指令被视为注释。 此外，由于出现在不是解析器指令的注释之后，已知指令被视为注释。

```shell
# unknowndirective=value
# knowndirective=value
```

解析器指令中允许使用非换行空格。 因此，以下行都被同等对待：

```shell
#directive=value
# directive =value
#	      directive= value
# directive = value
#	        dIrEcTiVe=value
```

支持以下解析器指令：

+ `syntax`
+ `escape`

#### 4.1 syntax

```shell
# syntax=[remote image reference]
```

举例：

```shell
# syntax=docker/dockerfile:1
# syntax=docker.io/docker/dockerfile:1
# syntax=example.com/user/repo:tag@sha256:abcdef...
```

这些特性只有当使用[BuildKit](https://docs.docker.com/engine/reference/builder/#buildkit)做后端的时候才可用，如果使用以前的builder后端的话这些标记都会被忽略掉。

The syntax directive defines the location of the Dockerfile syntax that is used to build the Dockerfile. The BuildKit backend allows to seamlessly use external implementations that are distributed as Docker images and execute inside a container sandbox environment.

语法指令定义用于构建 Dockerfile 的 Dockerfile 语法的位置。 BuildKit 后端允许无缝使用外部实现作为分发的Docker 镜像并在容器沙箱环境中执行。

用户Dockerfile实现允许：

Custom Dockerfile implementations allows you to:

- 在不更新 Docker daemon守护进程的情况下自动获取错误修复
- 确保所有用户都使用相同的实现来构建您的 Dockerfile
- 无需更新 Docker daemon守护进程可使用最新功能
- 在将新功能或第三方功能集成到 Docker daemon守护进程之前试用它们
- Use [alternative build definitions, or create your own](https://github.com/moby/buildkit#exploring-llb)



### Official releases

Docker distributes official versions of the images that can be used for building Dockerfiles under `docker/dockerfile` repository on Docker Hub. There are two channels where new images are released: `stable` and `labs`.

Docker 分发官方版本的镜像，作为用于构建的Dockerfile存在 Docker Hub 上的 `docker/dockerfile` 存储库下。 有两个发布新镜像的渠道：`stable` 和 `labs`。

Stable channel follows [semantic versioning](https://semver.org/). For example:

stable渠道遵循[语义版本控制](https://semver.org/)。举例：

- `docker/dockerfile:1` - kept updated with the latest `1.x.x` minor *and* patch release
- `docker/dockerfile:1.2` - kept updated with the latest `1.2.x` patch release, and stops receiving updates once version `1.3.0` is released.
- `docker/dockerfile:1.2.1` - immutable: never updated

We recommend using `docker/dockerfile:1`, which always points to the latest stable release of the version 1 syntax, and receives both “minor” and “patch” updates for the version 1 release cycle. BuildKit automatically checks for updates of the syntax when performing a build, making sure you are using the most current version.

If a specific version is used, such as `1.2` or `1.2.1`, the Dockerfile needs to be updated manually to continue receiving bugfixes and new features. Old versions of the Dockerfile remain compatible with the new versions of the builder.
