# php-microservice-demo

近些年微服务架构大行其道，趁着最近有时间，来捣鼓捣鼓微服务是怎么一回事。

## 微服务架构

微服务的概念由 Martin Fowler 于2014年3月提出：

> 微服务架构是一种架构模式，它提倡将单一应用程序划分成一组小的服务，服务之间相互协调、互相配合，为用户提供最终价值。每个服务运行在其独立的进程中，服务和服务之间采用轻量级的通信机制相互沟通。每个服务都围绕着具体的业务进行构建，并且能够被独立的部署到生产环境、类生产环境等。另外，应尽量避免统一的、集中的服务管理机制，对具体的一个服务而言，应根据业务上下文，选择合适的语言、工具对其进行构建。

下图是一个电商系统的微服务架构图：
![](https://segmentfault.com/img/bVbxQGd?w=541&h=488/view)


微服务架构与单体应用相比，具有以下优点：

1. 每个服务都比较简单，只关注于一个业务功能；
微服务架构方式是松耦合的，每个服务可以独立测试、部署、升级、发布；
1. 每个微服务可由不同团队独立开发，可以各自选择最佳及最合适的不同的编程语言与工具；
1. 每个服务可以根据需要进行水平扩展，提高系统并发能力。
没有银弹，微服务架构在带来诸多优点的同时，也会有如下缺点：

1. 微服务架构提高了系统的复杂度，增加了运维开销及成本。如单体应用可能只需部署至一小片应用服务集群，而微服务架构可能变成需要构建/测试/部署/运行数十个独立的服务，并可能需要支持多种语言和环境；
1. 作为一种分布式系统，微服务架构引入了其他若干问题，例如消息序列化、网络延迟、异步机制、容错处理、服务雪崩等；
1. 服务管理的复杂性，如服务的注册、发现、降级、熔断等问题；
1. 服务与服务之间存在相互调用的情况，为排查系统故障带来巨大挑战。

可以说，正是传统应用架构的系统变得日益臃肿，面临难以维护、扩展的问题，同时容器化技术（Docker）的蓬勃发展和 DevOps 思想的日渐成熟，催生了新的架构设计风格 – 微服务架构的出现。

## RPC 框架

微服务架构中的各个服务通常不在同一个机器上，甚至不会在同一个网络环境里，因此微服务之间如何调用是一个亟待解决的问题，我们通常使用 RPC 协议来解决：

> RPC（Remote Procedure Call），即远程过程调用，是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一台计算机的子程序，而程序员无需额外地为这个交互作用编程。——维基百科

实现了 RPC 协议的框架，可以让服务方和调用方屏蔽各种底层细节，让调用方像调用本地函数一样调用远端的函数（服务）。RPC 框架一般为服务端和客户端提供了序列化、反序列化、连接池管理、负载均衡、故障转移、队列管理、超时管理、异步管理等职能。在网上找到一个说明 RPC 框架工作原理图：

![](https://segmentfault.com/img/bVbxQGk?w=1490&h=712)

目前，根据序列化数据时采用的技术的不同，可分为 JSON-RPC 和 gRPC 两种：

- JSON-RPC 是一种基于 JSON 格式的轻量级的 RPC 协议标准，可基于 HTTP 协议来传输，或直接基于 TCP 协议来传输。 JSON-RPC 优点是易于使用和阅读。
- gRPC 是一个高性能、通用的开源 RPC 框架，其由 Google 主要面向移动应用开发并基于 HTTP/2 协议标准而设计，基于 ProtoBuf (Protocol Buffers) 序列化协议开发，且支持众多开发语言。 gRPC 具有低延迟、高效率、高扩展性、支持分布式等优点。

## Consul

现在有了 RPC 框架，我们就可以只考虑服务与服务之间的业务调用而不用考虑底层传输细节。此时，如果服务 A 想调用服务 B 时，我们可以在服务 A 中配置服务 B 的 IP 地址和端口，然后剩下的传输细节就交给 RPC 框架。这在微服务规模很小的情况下是没有问题的，但是在服务规模很大、而且每个服务不止部署一个实例的情况下会面临巨大挑战。比如，服务 B 部署了三个实例，这时候服务 A 想调用服务 B 该请求哪个实例的 IP ？假如服务 B 部署的三个实例有两个都挂掉了，服务 A 可能会依旧去请求挂掉的实例，服务将不可用。将 IP 地址和端口写成配置文件显得很不灵活，微服务架构往往要保证高可用及动态伸缩。

因此，我们需要一个服务注册与服务发现的工具，能够动态地变更服务信息，并且找到可用的服务的 IP 地址和端口。目前市面上服务发现的工具有很多，如 Consul、ZooKeeper 、Etcd、Doozerd 等，本文主要以 Consul 软件为例。

> Consul 是一个支持多数据中心、分布式高可用的服务发现和配置共享的服务软件，由 HashiCorp 公司用 Go 语言开发, 基于 Mozilla Public License 2.0 的协议进行开源。 Consul 支持健康检查，并允许 HTTP 、gRPC 和 DNS 协议调用 API 存储键值对。

下面是引入服务注册与服务发现工具后的架构图：

![](https://segmentfault.com/img/bVbxQGp?w=425&h=357)

在这个架构中：

1. 首先 S-B 的实例启动后将自身的服务信息（主要是服务所在的 IP 地址和端口号）注册到 Consul 中。
1. Consul 会对所有注册的服务做健康检查，以此来确定哪些服务实例可用哪些不可用。
1. S-A 启动后就可以通过访问 Consul 来获取到所有健康的 S-B 实例的 IP 和端口，并将这些信息放入自己的内存中，S-A 就可用通过这些信息来调用 S-B。
1. S-A 可以通过监听 Consul 来更新存入内存中的 S-B 的服务信息。比如 S-B-1 挂了，健康检查机制就会将其标为不可用，这样的信息变动就被 S-A 监听到了，S-A 就更新自己内存中 S-B-1 的服务信息。

可见， Consul 软件除了服务注册和服务发现的功能之外，还提供了健康检查和状态变更通知的功能。

## Hyperf

对于 Java 开发者来说，有技术相当成熟的 Dubbo 和 Spring Cloud 微服务框架可供选择。作为一名 PHPer，我用 Google 查了一下「PHP + 微服务」，发现有用的相关内容少之又少 ，没有什么实质性的参考价值，无限惆怅。。。幸好，有大神在基于 Swoole 扩展的基础上，实现了高性能、高灵活性的 PHP 协程框架 Hyperf ，并提供了微服务架构的相关组件。

> Hyperf 是基于 Swoole 4.3+ 实现的高性能、高灵活性的 PHP 协程框架，内置协程服务器及大量常用的组件，性能较传统基于 PHP-FPM 的框架有质的提升，提供超高性能的同时，也保持着极其灵活的可扩展性，标准组件均基于 PSR 标准 实现，基于强大的依赖注入设计，保证了绝大部分组件或类都是 可替换 与 可复用 的。
于是，我在学习了微服务架构相关的基础知识之后，使用 Hyperf 框架构建了一个基于 PHP 的微服务集群，这是项目源码地址：https://github.com/Jochen-z/p...。该项目使用 Dokcer 搭建，docker-compose.yml 代码如下：

```
version: "3"

services:
  consul-server-leader:
    image: consul:latest
    container_name: consul-server-leader
    command: "agent -server -bootstrap -ui -node=consul-server-leader -client=0.0.0.0"
    environment:
      - CONSUL_BIND_INTERFACE=eth0
    ports:
      - "8500:8500"
    networks:
      - microservice

  microservice-1:
    build:
      context: .
    container_name: "microservice-1"
    command: "php bin/hyperf.php start"
    depends_on:
      - "consul-server-leader"
    volumes:
      - ./www/microservice-1:/var/www
    networks:
      - microservice
    tty: true

  microservice-2:
    build:
      context: .
    container_name: "microservice-2"
    command: "php bin/hyperf.php start"
    depends_on:
      - "consul-server-leader"
    volumes:
      - ./www/microservice-2:/var/www
    networks:
      - microservice
    tty: true

  app:
    build:
      context: .
    container_name: "app"
    command: "php bin/hyperf.php start"
    depends_on:
      - "microservice-1"
    volumes:
      - ./www/web:/var/www
    ports:
      - "9501:9501"
    networks:
      - microservice
    tty: true

networks:
  microservice:
    driver: bridge

volumes:
  microservice:
    driver: local

```

这里启动了一个 Consul 容器 consul-server-leader 作为服务注册和服务发现的组件，容器 microservice-1 和 microservice-2 分别提供了加法运算和除法运算的服务。容器 app 作为服务调用方，配置了 consul-server-leader 容器的 URL，通过访问 consul-server-leader 获取 microservice-1 和 microservice-2 服务的 IP 地址和端口，然后 app 通过 RPC 协议调用加法运算和除法运算的服务获取结果并返回给用户。

app 容器为 Web 应用，部署了一个 Hyperf 项目并对外提供 HTTP 服务。例如，在 App\Controller\IndexController 控制器里有 add 方法：

```
public function add(AdditionService $addition)
{
  $a = (int)$this->request->input('a', 1); # 接受前端用户参数
  $b = (int)$this->request->input('b', 2);

  return [
    'a' => $a,
    'b' => $b,
    'add' => $addition->add($a, $b) # RPC调用
  ];
}
```

在 App\JsonRpc\AdditionService 中 add 的实现：

```
class AdditionService extends AbstractServiceClient
{
    /**
     * 定义对应服务提供者的服务名称
     * @var string
     */
    protected $serviceName = 'AdditionService';

    /**
     * 定义对应服务提供者的服务协议
     * @var string
     */
    protected $protocol = 'jsonrpc-http';

    public function add(int $a, int $b): int
    {
        return $this->__request(__FUNCTION__, compact('a', 'b'));
    }
}
```

继承了 AbstractServiceClient 即可创建一个微服务客户端请求类，Hyperf 在底层帮我们实现了与 Consul 和服务提供者交互的细节，我们只要 AdditionService 类里的 add 方法即可远程调用 microservice-1 和 microservice-2 提供的服务。

至此，PHP 微服务集群搭建就完成了！

参考文章：

1. [一文详解微服务架构](https://www.cnblogs.com/skabyy/p/11396571.html)

1. [离不开的微服务架构，脱不开的RPC细节](https://mp.weixin.qq.com/s/CepNdL2A_QMcESgQVZBn2g)
1. [用 Consul 来做服务注册与服务发现](https://juejin.im/post/5ca1f1acf265da308d50be02)
1. [Hyperf文档-微服务](https://doc.hyperf.io/#/zh/microservice)
1. [PHP 微服务集群搭建](https://segmentfault.com/a/1190000020421333)
