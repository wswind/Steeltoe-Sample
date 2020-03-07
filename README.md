## 微服务的发展

> 微服务实现框架目前分为三代。
>
> 第一代为一体式开发框架，由一个完整的分布式开发框架实现，如Dubbo(Java)、Orleans(.Net)等。
>
> 第二代为组装式框架，将原本一体式框架中的关键功能抽离出来，转而由独立的服务中间件来提供，代表为Spring Boot，以及Asp.NET Core eShop 2.x中自行组装的微服务技术栈；
>
> 第三代为服务网格，它将原本由Spring Cloud等框架的职能，转移服务网格中，通过k8s中前置的系统级微服务来提供。系统级微服务与开发者设计的微服务运行在同一个Pod中。使得原本由开发框架提供的功能，转而成为容器托管平台的系统级功能。
>
> 参考：
>
> <https://www.cnblogs.com/edisonchou/p/java_spring_cloud_foundation_sample_list.html>
>
> <https://mp.weixin.qq.com/s/pOmHnZbFcgoRrYvh6nWRhA>

## Steeltoe是什么

Spring Cloud作为组装式框架的典范，被中小企业广泛使用。Steeltoe是帮助.NET开发的服务接入Spring Cloud技术栈的官方支持工具。也就是说，微服务的系统框架，还是由Spring Cloud来实现，而业务服务，通过.NET Core来实现。后面我们将基于Steeltoe来尝试实现微服务系统框架。

![](https://img2020.cnblogs.com/blog/1114902/202003/1114902-20200307164818524-1028921296.png)

steeltoe主要包含以下功能：断路器，配置中心，服务连接器（MSSql、MySql、Oauth、Mongodb、Redis等），服务发现，网络文件共享（windows），动态日志，云管理（服务监控），云安全（Jwt认证），开发工具（Steeltoe CLI）。steeltoe源码结构如上图。


steeltoe项目地址：<https://github.com/SteeltoeOSS/steeltoe>
steeltoe样例地址：<https://github.com/SteeltoeOSS/Samples>
steeltoe文档：<https://steeltoe.io/docs/>

 


## Consul服务发现

参考： <https://steeltoe.io/service-discovery/get-started/consul>
```
docker pull consul
docker run -d --publish 8500:8500 consul
```

通过<https://start.steeltoe.io/>创建模板

## Api网关





本文源码地址: <https://github.com/wswind/Steeltoe-Sample>