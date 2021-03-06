## 镜像原理

docker 镜像是一个只读的 docker 容器模板，含有启动 docker 容器所需的文件系统结构及其内容，因此是启动一个 docker 容器的基础。docker 镜像的文件内容以及一些运行 docker 容器的配置文件组成了 docker 容器的静态文件系统运行环境：rootfs。可以这么理解，docker 镜像是 docker 容器的静态视角，docker 容器是 docker 镜像的运行状态。我们可以通过下图来理解 docker daemon、docker 镜像以及 docker 容器三者的关系\(此图来自互联网\)：

![](/assets/image-container-releationship.png)

**rootfs**

rootfs 是 docker 容器在启动时内部进程可见的文件系统，即 docker 容器的根目录。rootfs 通常包含一个操作系统运行所需的文件系统，例如可能包含典型的类 Unix 操作系统中的目录系统，如 /dev、/proc、/bin、/etc、/lib、/usr、/tmp 及运行 docker 容器所需的配置文件、工具等。在传统的 Linux 操作系统内核启动时，首先挂载一个只读的 rootfs，当系统检测其完整性之后，再将其切换为读写模式。而在 docker 架构中，当 docker daemon 为 docker 容器挂载 rootfs 时，沿用了 Linux 内核启动时的做法，即将 rootfs 设为只读模式。在挂载完毕之后，利用联合挂载\(union mount\)技术在已有的只读 rootfs 上再挂载一个读写层。这样，可读写的层处于 docker 容器文件系统的最顶层，其下可能联合挂载了多个只读的层，只有在 docker 容器运行过程中文件系统发生变化时，才会把变化的文件内容写到可读写层，并隐藏只读层中的旧版本文件。

**Docker 镜像的主要特点**

为了更好的理解 docker 镜像的结构，下面介绍一下 docker 镜像设计上的关键技术。

分层docker 镜像是采用分层的方式构建的，每个镜像都由一系列的 "镜像层" 组成。分层结构是 docker 镜像如此轻量的重要原因。当需要修改容器镜像内的某个文件时，只对处于最上方的读写层进行变动，不覆写下层已有文件系统的内容，已有文件在只读层中的原始版本仍然存在，但会被读写层中的新版本所隐藏。当使用 docker commit 提交这个修改过的容器文件系统为一个新的镜像时，保存的内容仅为最上层读写文件系统中被更新过的文件。分层达到了在不的容器同镜像之间共享镜像层的效果。

写时复制docker 镜像使用了写时复制\(copy-on-write\)的策略，在多个容器之间共享镜像，每个容器在启动的时候并不需要单独复制一份镜像文件，而是将所有镜像层以只读的方式挂载到一个挂载点，再在上面覆盖一个可读写的容器层。在未更改文件内容时，所有容器共享同一份数据，只有在 docker 容器运行过程中文件系统发生变化时，才会把变化的文件内容写到可读写层，并隐藏只读层中的老版本文件。写时复制配合分层机制减少了镜像对磁盘空间的占用和容器启动时间。

内容寻址在 docker 1.10 版本后，docker 镜像改动较大，其中最重要的特性便是引入了内容寻址存储\(content-addressable storage\) 的机制，根据文件的内容来索引镜像和镜像层。与之前版本对每个镜像层随机生成一个 UUID 不同，新模型对镜像层的内容计算校验和，生成一个内容哈希值，并以此哈希值代替之前的 UUID 作为镜像层的唯一标识。该机制主要提高了镜像的安全性，并在 pull、push、load 和 save 操作后检测数据的完整性。另外，基于内容哈希来索引镜像层，在一定程度上减少了 ID 的冲突并且增强了镜像层的共享。对于来自不同构建的镜像层，主要拥有相同的内容哈希，也能被不同的镜像共享。

联合挂载通俗地讲，联合挂载技术可以在一个挂载点同时挂载多个文件系统，将挂载点的原目录与被挂载内容进行整合，使得最终可见的文件系统将会包含整合之后的各层的文件和目录。实现这种联合挂载技术的文件系统通常被称为联合文件系统\(union filesystem\)。以下图所示的运行 Ubuntu:14.04 镜像后的容器中的 aufs 文件系统为例：

![](/assets/image-theory-2.png)

由于初始挂载时读写层为空，所以从用户的角度看，该容器的文件系统与底层的 rootfs 没有差别;然而从内核的角度看，则是显式区分开来的两个层次。当需要修改镜像内的某个文件时，只对处于最上方的读写层进行了变动，不复写下层已有文件系统的内容，已有文件在只读层中的原始版本仍然存在，但会被读写层中的新版本文件所隐藏，当 docker commit 这个修改过的容器文件系统为一个新的镜像时，保存的内容仅为最上层读写文件系统中被更新过的文件。联合挂载是用于将多个镜像层的文件系统挂载到一个挂载点来实现一个统一文件系统视图的途径，是下层存储驱动\(aufs、overlay等\) 实现分层合并的方式。所以严格来说，联合挂载并不是 docker 镜像的必需技术，比如在使用 device mapper 存储驱动时，其实是使用了快照技术来达到分层的效果。

**Docker 镜像的存储组织方式**

综合考虑镜像的层级结构，以及 volume、init-layer、可读写层这些概念，一个完整的、在运行的容器的所有文件系统结构可以用下图来描述：

![](/assets/image-theory-3.png)

从图中我们不难看到，除了 echo hello 进程所在的 cgroups 和 namespace 环境之外，容器文件系统其实是一个相对独立的组织。可读写部分\(read-write layer 以及 volumes\)、init-layer、只读层\(read-only layer\) 这 3 部分结构共同组成了一个容器所需的下层文件系统，它们通过联合挂载的方式巧妙地表现为一层，使得容器进程对这些层的存在一无所知。

**Docker 镜像中的关键概念**

registry我们知道，每个 docker 容器都要依赖 docker 镜像。那么当我们第一次使用 docker run 命令启动一个容器时，是从哪里获取所需的镜像呢?答案是，如果是第一次基于某个镜像启动容器，且宿主机上并不存在所需的镜像，那么 docker 将从 registry 中下载该镜像并保存到宿主机。如果宿主机上存在该镜像，则直接使用宿主机上的镜像完成容器的启动。那么 registry 是什么呢?registry 用以保存 docker 镜像，其中还包括镜像层次结构和关于镜像的元数据。可以将 registry 简单的想象成类似于 Git 仓库之类的实体。用户可以在自己的数据中心搭建私有的 registry，也可以使用 docker 官方的公用 registry 服务，即 Docker Hub。它是由 Docker 公司维护的一个公共镜像库。Docker Hub 中有两种类型的仓库，即用户仓库\(user repository\) 与顶层仓库\(top-level repository\)。用户仓库由普通的 Docker Hub 用户创建，顶层仓库则由 Docker 公司负责维护，提供官方版本镜像。理论上，顶层仓库中的镜像经过 Docker 公司验证，被认为是架构良好且安全的。

repositoryrepository 由具有某个功能的 docker 镜像的所有迭代版本构成的镜像组。Registry 由一系列经过命名的 repository 组成，repository 通过命名规范对用户仓库和顶层仓库进行组织。所谓的顶层仓库，其其名称只包含仓库名，如：

![](/assets/image-theory-4.jpg)

而用户仓库的表示类似下面：

![](/assets/image-theory-5.jpg)

可以看出，用户仓库的名称多了 "用户名/" 部分。

manifestmanifest\(描述文件\)主要存在于 registry 中作为 docker 镜像的元数据文件，在 pull、push、save 和 load 过程中作为镜像结构和基础信息的描述文件。在镜像被 pull 或者 load 到 docker 宿主机时，manifest 被转化为本地的镜像配置文件 config。在我们拉取镜像时显示的摘要\(Digest\)：![](/assets/image-theory-6.jpg)

就是对镜像的 manifest 内容计算 sha256sum 得到的。

image 和 layerdocker 内部的 image 概念是用来存储一组镜像相关的元数据信息，主要包括镜像的架构\(如 amd64\)、镜像默认配置信息、构建镜像的容器配置信息、包含所有镜像层信息的 rootfs。docker 利用 rootfs 中的 diff\_id 计算出内容寻址的索引\(chainID\) 来获取 layer 相关信息，进而获取每一个镜像层的文件内容。layer\(镜像层\) 是 docker 用来管理镜像层的一个中间概念。我们前面提到，镜像是由镜像层组成的，而单个镜像层可能被多个镜像共享，所以 docker 将 layer 与 image 的概念分离。docker 镜像管理中的 layer 主要存放了镜像层的 diff\_id、size、cache-id 和 parent 等内容，实际的文件内容则是由存储驱动来管理，并可以通过 cache-id 在本地索引到。

