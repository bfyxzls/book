## ARG

创建镜像过程中使用的变量

* `ARG <参数名>[=<默认值>]`

构建参数和`ENV`的效果一样，都是设置环境变量。所不同的是，`ARG`所设置的构建环境的环境变量，在将来容器运行时是不会存在这些环境变量的。

`Dockerfile`中的`ARG`指令是定义参数名称，以及定义其默认值。该默认值可以在构建命令`docker build`中用`--build-arg <参数名>=<值>`来覆盖。

ARG指令在FORM之前。

ARG version=1.1

FROM ubuntu:16.04

