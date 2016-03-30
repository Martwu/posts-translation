CoreOS上的Fleet，第一部分
===

服务器宕机是经常发生的事情。然而我们还是非常希望不会因为这样导致应用程序挂了，因而影响到业务的正常服务。这也是为什么服务的高可用成了运维工程师选择把应用部署到云端上去的一个最重要的理由。

Fleet，一个CoreOS上的工具，就能解决上述的问题，把你从整天提心吊胆中释放出来。它会帮你把应用部署到健康的节点中运行起来。

然而，Fleet是怎么解决这个问题呢？

那Fleet是怎么知道某个节点是不是下线了呢？应用的重新路由又是怎么进行的呢？

我们在[前一篇文章](https://deis.com/blog/2015/run-self-sufficient-containers-coreos)里就介绍了这个过程，如果你急着回顾，我在这再概括说下。

CoreOS集群中，每个节点都运行着fleet的守护进程，用以监视节点的健康，还有与其它节点进行通信。在集群启动时，或者当集群中最近的leader不可用时，fleet的守护进程就会协助着去选举出集群中的leader。每当有请求被提交到这个集群，或者某个节点跑着服务时挂了，集群中的leader就会安排一个节点跑新的服务来响应。

接下来将模拟一个场景，我们用fleet集群来跑一些服务，然后令一个节点挂掉，以此观察下fleet是怎样重新安排节点响应请求。接着我们在继续深入了解下fleet额外的功能。

开始
---

让我现在马上来看看Fleet。特别是要看下当一个节点挂了时，重新路由是怎么进行的。

一开始，你应该跑起一个CoreOS集群。如果你不是很清楚应该怎么做，参考下《[如何在AWS上安装CoreOS](https://deis.com/blog/2016/coreos-on-aws)》这篇教程。注意：在这篇教程中，我在AWS EC2上跑了三个节点的CoreOS集群。

当你准备好集群，就可以连接进去了。

可以像这样做：

    ssh -i /Path/to/keyfile/keyfile.pem core@ec-2-server-path.compute.amazonaws.com

然后你要改这个密钥的路径成实际的路径，还有主机名改成你自己的EC2服务器的主机名。

当你连接进去后，你跑下以下命令来检查下是否已经安装了fleet：

    $ fleetctl

你应该了解下以类似下的内容：

    NAME:

        fleetctl - fleetctl is a command-line interface to fleet, the cluster-wide CoreOS init system.

    USAGE:

        fleetctl [global options] <command> [command options] [arguments...]

    [...]

定义你的服务
---

若fleet在CoreOS上预装了的，fleet也是不会自动跑起来的。

要启动fleet，我们还需要加至少一个单元的配置文件。这配置用来描述那个你希望在你的集群中跑起来的服务。

以下是用来举例的一单元的配置：

    [Unit]
    Description=MyApp
    After=docker.service
    Requires=docker.service

    [Service]
    TimeoutStartSec=0
    ExecStartPre=-/usr/bin/docker kill busybox1
    ExecStartPre=-/usr/bin/docker rm busybox1
    ExecStartPre=/usr/bin/docker pull busybox
    ExecStart=/usr/bin/docker run --name busybox1 busybox /bin/sh -c "while true; do echo Hello World; sleep 1; done"
    ExecStop=/usr/bin/docker stop busybox1

我们可以命名这个文件为`myapp.service`。

`Description`会在日志中显示出来，所以最好还是用容易辨别的值，方便你之后辨别日志。`After=docker.service`和`Requires=docker.service`意思是这个单元的配置只会在`docker.service`生效后才会启动。

`ExecStart`选项可以让你去定义一条在单元配置启动时执行的命令。`ExecStartPre`选项可以让你定义一条命令，在`ExecStart`中定义的命令之前执行。这个选项可以用以做一些类似清理环境，设置环境的行为。

千万不可以在意图跑一个docker容器时加`-d`参数来运行fleet，因为加了这参数后fleet进程就不是容器的子进程了。这样的话，systemd会认为该进程已经退出了，然后该fleet单元就会终止服务。

`ExecStop`中定义的命令则会在该单元启动失败或者终止时执行。

要想启动这个单元服务，则要移动`myapp.service`的配置文件或者创建之在那个你已经连接上的节点上去。

然后执行：

    $ fleetctl start myapp.service

你就会看到这个单元服务启动了。

然后你还可以运行一下命令来确认一下情况：

    $ fleetctl list-unit-files

这条命令会列出在该集群中的运行着的全部单元配置文件以及单元配置所在节点的IP地址。

现在我们已经添加了一个服务到集群中去并且运行着了，我们还可以重复上述的步骤继续添加多个服务到集群去并运行起来。只需要记住的是单元配置文件的名字必须在集群中是唯一的。

当完成了后，再跑下上面的命令。

结果就应该看起来是这样的：

    $ fleetctl list-unit-files
    UNIT               HASH    DSTATE   STATE    TARGET
    myapp.service      d4c61bf launched launched 85c0c595.../172.31.5.9
    anotherapp.service e55c0ae launched launched 113f16a7.../172.31.22.187
    someapp.service    391d247 launched launched a0b7a5f7.../172.31.22.22

可以看到这里我们在三台服务器上跑着三个服务（`myapp.service`，`anotherapp.service`，`someapp.service`）。在`TARGET`一列中就列出了这些服务器的IP地址。

Fleet的运转
===

就如在一开始提到的那样，fleet最好的那点就是，它不单单可以通过集群当请求被提交到集群来时安排单元服务去响应，它还可以自动重心路由应用到健康的节点去运行。

让我们来搞挂一个节点来看看自动重路由是怎么运转的。

我是在在AWS控制台上通过关停我其中一个EC2实例来实现的。你就可以用你跑虚拟机的平台所带的控制台来做就可以了。

当你搞挂一个节点后，你可以运行下：

    $ fleetctl list-unit-files

然后你应该可以看到这样的输出：

    UNIT               HASH    DSTATE   STATE    TARGET
    myapp.service      d4c61bf launched launched 85c0c595.../172.31.5.9
    anotherapp.service e55c0ae launched launched 113f16a7.../172.31.22.187
    someapp.service    391d247 launched launched a0b7a5f7.../172.31.22.187

一开始呢，`someapp.service`是跑在172.31.22.22这机器上的，但是现在是和`anotherapp.service`一起跑在172.31.22.187的。所以在172.31.22.22从集群中被移走，还有`someapp.service`被移动到跑着`anotherapp.service`的健康节点上去时，发生了什么事情？

反正fleet就自动完成了此行为，而不需要人工干预。

是不是很酷呢！

结尾
===

在这篇文章里，我们学到了：

* 怎么去创建一个我们希望在集群中运行的服务所需的描述该服务的单元配置文件。
* 怎么用fleet在集群中启动一个服务。
* 怎么去发现有节点挂了并且服务被移到健康的节点去了。

在下一篇文章，我们会在同一个场景中，继续看下怎么以更高端的方式来用fleet。
