## 示例

[资料参考](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

```dockerfile
FROM golang:1.11-alpine AS build

# Install tools required for project
# Run `docker build --no-cache .` to update dependencies
RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

# List project dependencies with Gopkg.toml and Gopkg.lock
# These layers are only re-built when Gopkg files are updated
COPY Gopkg.lock Gopkg.toml /go/src/project/
WORKDIR /go/src/project/
# Install library dependencies
RUN dep ensure -vendor-only

# Copy the entire project and build it
# This layer is rebuilt when a file changes in the project directory
COPY . /go/src/project/
RUN go build -o /bin/project

# This results in a single layer image
FROM scratch
# 从编译阶段拷贝编译结果到当前镜像中, 这个build是第1个FROM的别名(AS build) 
COPY --from=build /bin/project /bin/project
ENTRYPOINT ["/bin/project"]
CMD ["--help"]
```

### 多个FROM指令的意义

每一条`FROM` 指令都是一个构建阶段，多条 `FROM` 就是多阶段构建，虽然最后生成的镜像只能是最后一个阶段的结果，但是，能够将前置阶段中的文件拷贝到后边的阶段中，这就是多阶段构建的最大意义。

最大的使用场景是将编译环境和运行环境分离, 如以上示例所示, 在`golang:1.11-alpine`镜像中编译程序, 然后把编译好的文件复制到`scratch`镜像, 最终镜像即是很精简的`scratch`镜像

注: `scratch` 是内置关键词，并不是一个真实存在的镜像。 `FROM scratch` 会使用一个完全干净的文件系统，不包含任何文件。 因为Go语言编译后不需要运行时，也就不需要安装任何的运行库。 `FROM scratch` 可以使得最后生成的镜像最小化，其中只包含了 server 程序。

### RUN、CMD、ENTRYPOINT 的区别

RUN 指令在docker build时运行

CMD、ENTRYPOINT 指令在docker run时运行

#### ENTRYPOINT

```bash
ENTRYPOINT ["/bin/echo", "Hello"]  
```

当容器通过 `docker run -it [image]` 启动时，输出为：

```undefined
Hello
```

而如果通过 `docker run -it [image] CloudMan` 启动，则输出为：

```undefined
Hello CloudMan
```

将Dockerfile修改为：

```objectivec
ENTRYPOINT ["/bin/echo", "Hello"]  
CMD ["world"]
```

当容器通过 `docker run -it [image]` 启动时，输出为：

```undefined
Hello world
```

而如果通过 `docker run -it [image] CloudMan` 启动，输出依旧为：

```undefined
Hello CloudMan
```

#### 总结

- 使用 RUN 指令安装应用和软件包，构建镜像。
- 如果 Docker 镜像的用途是运行应用程序或服务，比如运行一个 MySQL，应该优先使用 Exec 格式的 ENTRYPOINT 指令。CMD 可为 ENTRYPOINT 提供额外的默认参数，同时可利用 docker run 命令行替换默认参数。
- 如果想为容器设置默认的启动命令，可使用 CMD 指令。用户可在 docker run 命令行中替换此默认命令。

###COPY、ADD 的区别

COPY指令只能从执行docker build所在的主机上读取资源并复制到镜像中

而ADD指令还支持通过URL从远程服务器读取资源并复制到镜像中(当资源需要认证时,则只能使用RUN wget或RUN curl替代)

满足同等功能的情况下，推荐使用COPY指令。ADD指令更擅长读取本地tar文件并解压缩。

### VOLUME

如:

```dockerfile
VOLUME ["/data"]
```

Dockerfile 的`VOLUME`指令主要是起到声明匿名数据卷的作用, `docker run`启动容器时, 若不含设置数据卷的参数, 则会在`/var/lib/docker/volumes`目录创建匿名卷(建议设置好)

#### 推荐使用方法

​	容器运行时应该尽量保持容器存储层不发生写操作，对于数据库类需要保存动态数据的应用，其数据库文件应该保存于卷(volume)中

1. 创建一个VOLUME

   如: `docker volume create for_nginx`

   查看该数据卷的命令: `docker volume inspect for_nginx`

2. 启动容器, 挂载数据卷到容器

   `docker run -itd -p 88:80 --mount type=volume,source=for_nginx,target=/usr/share/nginx/html nginx`

   target 指定的是数据卷在容器中的挂载位置
   
3. 效果

   - 容器中的挂载位置与宿主数据卷文件一致, 任何一方的改动都可以让另一方同步, 可理解为VOLUME帮我们做了一个类似于软连接的功能
   - 持久化: 删除容器, 宿主的数据卷仍然存在
