# Moby project（Docker新框架）

[原文链接](https://blog.docker.com/2017/04/introducing-the-moby-project/)

![logo](https://i0.wp.com/blog.docker.com/wp-content/uploads/1-2.png?w=763&ssl=1)

自从Docker在4年前将软件容器技术民主化，整个生态系统都围绕容器化发展，在这个被紧紧压缩的时间里，它经历了两个不同的增长阶段。在这两个阶段里，生产容器化系统的模型都发生了变化，以适应用户群体的规模和需求，和项目长与贡献者生态系统的不断增。

Moby项目是一个新的开源项目，勇于推动软件容器化运动，并帮助生态系统将容器化作为主流。Moby项目提供了一个组件库、一个框架，用来把基于容器的定制系统集成进来，并为所有容器技术爱好者提供实验和交流想法的地方。

让我们回顾一下我们经历的路程。在2013-2014年，先驱们开始使用容器，在一个单一的开源代码库Docker和其他几个项目中协作，逐渐使Docker变得成熟。

![Production Model: open-source!](https://i0.wp.com/blog.docker.com/wp-content/uploads/2-2.png?w=975&ssl=1)

然后在2015-2016年，容器技术在原生云应用生产环境中被大量采用。在这个阶段，用户社区成长为拥有千上万次部署的项目，而背后有数百个生态系统项目和数千个贡献者支持。在此阶段，Docker将其生产模式演变为基于开放组件的方法。这样就可以增加创新和合作的涉及面。

新的独立Docker组件项目激起了合作伙伴生态系统和用户群体的增长。在此期间，我们将Docker代码库中的组件提取出来并快速创新，以便系统制造商可以在构建自己的容器系统时[独立](https://github.com/docker/hyperkit)地重用它们：[runc](https://github.com/opencontainers/runc)，[HyperKit](https://github.com/docker/hyperkit)，[VPNKit](https://github.com/docker/vpnkit)，[SwarmKit](https://github.com/docker/swarmkit)，[InfraKit](https://github.com/docker/infrakit)，[containerd](https://github.com/containerd/containerd)等。

![Production Model: OPEN COMPONENTS](https://i1.wp.com/blog.docker.com/wp-content/uploads/3-2.png?w=975&ssl=1)

在容器技术的最前沿的浪潮里，我们看到2017年出现的一个趋势是容器技术成为主流，扩展到各类计算，服务器，数据中心，云，桌面，物联网和手机；扩展到每个行业和垂直市场，金融，医疗保健，政府，旅游，制造业；并且扩展到每个案例，现代Web应用程序，传统服务器应用程序，机器学习，工业控制系统，机器人。容器化生态系统中许多新进入者的共同之处在于，他们建立专门系统来针对特定基础设施，行业或案例。

作为一家公司，Docker使用开源作为我们的创新实验室，与整个生态系统进行合作。Docker的成功与容器化生态系统的成功息息相关：如果生态系统成功，我们就能成功。因此，我们一直在规划容器化生态系统增长的下一个阶段：什么样的生产模式将帮助我们扩大容器化生态系统，以实现容器技术主流化的承诺。

去年，我们的客户开始在Linux之外的许多平台上要求提供Docker：Mac和Windows桌面，Windows Server，Amazon Web Services（AWS），Microsoft Azure或Google Cloud Platform等云平台，我们专门为这些平台创建了[十几个Docker版本](https://blog.docker.com/2017/03/docker-enterprise-edition/)。

为了让构建和发布这些专门的版本可以在一个相对较短的时间内，小团队，以可扩展的方式，并且不必重新创造轮子的情况完成; 很明显，我们需要一种新的方法。我们需要我们的团队不仅可以组合基础元件，还可以组合组件（多个元件组合在一起），借鉴[汽车行业的构想](https://en.wikipedia.org/wiki/List_of_Volkswagen_Group_platforms)，重用组件以构建完全不同的汽车。

[Scaling the Docker production model: share components AND ASSEMBLIES](https://i1.wp.com/blog.docker.com/wp-content/uploads/4-2.png?w=975&ssl=1)

我们认为将容器化生态系统扩大到下一个级别并实现容器化的主流地位的最佳途径是对生态系统层面的组件进行组合。

![next level](https://i0.wp.com/blog.docker.com/wp-content/uploads/5-2.png?w=975&ssl=1)

为了实现这一新的协作水平，今天我们宣布推出Moby项目，该项目是推动软件容器化运动的新型开源项目。它提供了数十个元件的“乐高集”，以及将其组装到基于容器的定制系统中的框架，同时Moby是所有容器爱好者进行实验和交流想法的场所。可以将Moby视为容器系统的“乐高俱乐部”。

Moby包括：

1. 一个容器化后端元件 **库**（例如，底层bulder，日志工具，卷管理，网络，镜像管理，containerd，SwarmKit，...）
1. 一个 **框架** 用来将元件组装到一个独立的容器平台，并且提供工具来对这些组件进行编译、测试和发布。
1. 一个称为 **Moby Origin** 参考组件，它是Docker容器平台的开放式基础，同时它也是使用Moby库或者其它项目的容器系统的范例。

Moby专为系统构建者而设计，他们希望构建自己的基于容器的系统，而不只是使用Docker或其他容器平台的应用程序开发人员。Moby项目的参与者可以从Docker派生的元件库中进行选择，也可以选择“将您自己的元件”（BYOC）打包成容器，并可以在所有元件之间进行混合和匹配，以创建一个定制的容器系统。

Docker使用Moby项目作为一个开放的研发实验室，来探索和开发新的元件，并与生态系统在未来的容器技术上进行协作。我们所有的开源协作将转移到Moby项目。

请加入我们，帮助软件容器技术成为主流，通过对组件的组装，将我们的生态系统和用户群体扩展到更高的层次。

