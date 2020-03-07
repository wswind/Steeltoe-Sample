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

我没有使用docker for windows，而是在virtualbox 中创建了centos 7虚拟机运行docker，所以需要对官方教程的网络配置有调整。如果你的环境为docker for windows，则可以直接使用localhost

我的虚拟机网络环境为virtualbox的host only network + nat双网卡配置。
网关ip为192.168.56.1对应开发物理机ip，虚拟机ip为192.168.56.104对应consul ip。
virtualbox的网络环境配置，可参考我的另一篇博文<https://www.cnblogs.com/wswind/p/10832740.html>

在centos中通过命令运行consul：
```
docker pull consul
docker run -d --publish 8500:8500 consul #-d意味着后台运行
```

通过<https://start.steeltoe.io/>创建模板，选择NetCore3.1，组件选择Discovery。

![](https://steeltoe.io/images/initializr/service-discovery.png) 

此时我们可以看到，这个模板生成器，其实创建的就是一个Asp.NET Core WebAPI的空项目，唯一的不同，就是在项目文件中添加了包依赖：Steeltoe.Discovery.ClientCore。并在ConfigureServices时，调用了

```csharp
services.AddDiscoveryClient(Configuration);
```



通过在appsettings.json中添加配置，写明本机地址以及consul地址，运行项目，即可完成服务注册

```json
  "spring": {
    "application": {
      "name": "Consul-Register-Example"
    }
  },
  "consul": {
    "host": "192.168.56.104",
    "port": 8500,
    "discovery": {
      "enabled": true,
      "register": true,
      "port": "8080",
      "ipAddress": "192.168.56.1",
      "preferIpAddress": true
    }
  }
```

修改**Properties\launchSettings.json**

```
"iisSettings": {
		"windowsAuthentication": false, 
		"anonymousAuthentication": true, 
		"iisExpress": {
			"applicationUrl": "http://localhost:8080",
			"sslPort": 0
		}
	}
```



为了便于通过192.168.56.1访问服务，我添加了UseUrls：

```csharp
 var builder = WebHost.CreateDefaultBuilder(args)
                .UseUrls("http://0.0.0.0:8080")
```

如果不修改，对于健康检查貌似没什么影响，只是你注册上去的ip，就是无法访问的了。

关于asp.net core如何修改绑定ip，可参考：

<https://docs.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel?view=aspnetcore-3.1#endpoint-configuration>

<https://www.cnblogs.com/Hai--D/p/5842842.html>

运行项目后，打开consul地址，我们可以看到服务注册已经完成了。



![](https://img2020.cnblogs.com/blog/1114902/202003/1114902-20200307175836506-2126353697.png)

我们可以看到服务注册过程非常简单，steeltoe的服务注册工具的使用，仅需添加几行json配置即可完成。大大减少了我们的代码开发量。

上面是服务注册，接下面我们来讲解服务发现。由于已经了解steeltoe的模板生成器只是在空模板上加了nuget包的引用，我们也无需再通过它生成项目了。

通过.net core cli创建空项目，并添加nuget包即可，命令行如下：

```
dotnet new webapi -n ConsulDiscovery
cd ConsulDiscovery
dotnet add package Steeltoe.Discovery.ClientCore
```

在ConfigureSerivces中，设置DI services.AddDiscoveryClient(Configuration);

然后修改appsettings.json

```json
{
...

	"spring": {
		"application": {
			"name": "Consul-Discover-Example"
		}
	},
	"consul": {
		"host": "192.168.56.104",
		"port": 8500,
		"discovery": {
			"enabled": true,
			"register": false
		}
	}

...
}
```

修改WeatherForecastController，通过依赖注入IDiscoveryClient，创建DiscoveryHttpClientHandler，然后通过consule注册的服务地址来访问之前创建的服务。

```
        DiscoveryHttpClientHandler _handler;
        public WeatherForecastController(ILogger<WeatherForecastController> logger, IDiscoveryClient client)
        {
            _logger = logger;
            _handler = new DiscoveryHttpClientHandler(client);
        }

        [HttpGet]
        public async Task<string> Get()
        {
            var client = new HttpClient(_handler, false);
            return await client.GetStringAsync("http://Consul-Register-Example/api/values");
        }
```

启动服务后，可以看到返回值：

![](https://img2020.cnblogs.com/blog/1114902/202003/1114902-20200307181740506-836395060.png)

上述过程的UML序列图如下，服务注册客户端首先进行服务注册，服务发现通过读取Consul中的服务地址，来进行访问。

![](https://img2020.cnblogs.com/blog/1114902/202003/1114902-20200307235348082-1349478332.png)



## 本文源码地址

<https://github.com/wswind/Steeltoe-Sample>