## Steeltoe是什么

Spring Cloud作为组装式框架的典范，被中小企业广泛使用。Steeltoe是帮助.NET开发的服务接入Spring Cloud技术栈的官方支持工具。也就是说，微服务的系统框架，还是由Spring Cloud来实现，而业务服务，通过.NET Core来实现。后面我们将基于Steeltoe来尝试实现微服务系统框架。

![](https://img2020.cnblogs.com/blog/1114902/202003/1114902-20200307164818524-1028921296.png)

steeltoe主要包含以下功能：断路器，配置中心，服务连接器（MSSql、MySql、Oauth、Mongodb、Redis等），服务发现，网络文件共享（windows），动态日志，云管理（服务监控），云安全（Jwt认证），开发工具（Steeltoe CLI）。steeltoe源码结构如上图。

steeltoe项目地址：<https://github.com/SteeltoeOSS/steeltoe>
steeltoe样例地址：<https://github.com/SteeltoeOSS/Samples>
steeltoe文档：<https://steeltoe.io/docs/>

steeltoe运行于Cloud Foundry，关于Cloud Foundry的介绍，请查看：<https://blog.csdn.net/qq_30154571/article/details/84955097>


## Consul服务发现

参考： <https://steeltoe.io/service-discovery/get-started/consul>

我没有使用docker for windows，而是在virtualbox 中创建了centos 7虚拟机运行docker，所以需要对官方教程的网络配置有调整。如果你的环境为docker for windows，则可以直接使用localhost

我的虚拟机网络环境为virtualbox的host only network + nat双网卡配置。
网关ip为192.168.56.1对应开发物理机ip，虚拟机ip为192.168.56.104对应consul ip。
virtualbox的网络环境配置，可参考我的另一篇博文<https://www.cnblogs.com/wswind/p/10832740.html>

在centos中通过命令运行consul：
```
docker pull consul
docker run -d -p 8500:8500 consul #-d意味着后台运行
```

通过<https://start.steeltoe.io/>创建模板，选择NetCore3.1，组件选择Discovery。

![](https://img2020.cnblogs.com/blog/1114902/202003/1114902-20200316001459022-208745295.png)


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

```json
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

修改WeatherForecastController，通过依赖注入IDiscoveryClient，创建DiscoveryHttpClientHandler，然后通过consul注册的服务地址来访问之前创建的服务。

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

## Service Connectors with Microsoft SQL

![](https://img2020.cnblogs.com/blog/1114902/202003/1114902-20200316001526907-2062050483.png)


首先通过生成器<https://start.steeltoe.io/>创建项目，选netcore 3.1 + SQL Server

创建项目默认安装了Nuget包 Microsoft.EntityFrameworkCore.SqlServer并添加了依赖注入

```csharp
services.AddSqlServerConnection(Configuration);
```

不过.NET Core 3.1模板目前有问题缺少了一些包引用，导致无法编译通过，需要手动安装

```
System.Data.SqlClient
Steeltoe.CloudFoundry.ConnectorCore
```



另外，生成的模板项目中的nuget包Microsoft.EntityFrameworkCore.SqlServer不是必须的，如果不使用EfCore其实无需引入这个包。

在appsettings.json中添加数据库的连接配置

```json
{
...


 "sqlserver": {
	"credentials": {
		"server": "127.0.0.1",
		"port": "1433",
		"username": "sa",
		"password": "sa"
		
	}
 }

...
}
```

sql server需要启用tcp/ip以允许外部访问，开启后需要重启服务

![](https://img2020.cnblogs.com/blog/1114902/202003/1114902-20200316001547132-886913233.png)



模板中，向我们展示了这个包的用法

```csharp
...
    public class ValuesController : ControllerBase
    {
        private readonly SqlConnection _dbConnection;
        public ValuesController([FromServices] SqlConnection dbConnection)
        {
            _dbConnection = dbConnection;
        }

        // GET api/values
        [HttpGet]
        public ActionResult<IEnumerable<string>> Get()
        {
            List<string> tables = new List<string>();
        
            _dbConnection.Open();
            DataTable dt = _dbConnection.GetSchema("Tables");
            _dbConnection.Close();
            foreach (DataRow row in dt.Rows)
            {
                string tablename = (string)row[2];
                tables.Add(tablename);
            }
            return tables;
        }
...        
```





运行效果如下：

![](https://img2020.cnblogs.com/blog/1114902/202003/1114902-20200316001603416-1409503210.png)


通过Dapper来配合使用 Steeltoe.CloudFoundry.ConnectorCore 应该会很方便。

不过也有一个问题，依赖注入是直接注入了SqlConnection，不清楚如果同时连接多数据库能否拓展。不过对于微服务开发而言，这种情况很少见。



## Service Connectors with Redis Cache

首先运行redis

```
docker run -d -p 6379:6379 redis
```

通过 [Steeltoe Initializr](https://start.steeltoe.io/)创建项目选择.NET Core3.1 + Redis

项目中添加了包：

```
Microsoft.Extensions.Caching.StackExchangeRedis
Steeltoe.CloudFoundry.ConnectorCore
```

在依赖注入时，添加了：

```csharp
services.AddDistributedRedisCache(Configuration);
```

添加配置文件：

```json
{
...

 "redis": {
	"client": {
		
		"host": "192.168.56.104",
		"port": "6379",
	}
 }
...
}
```

默认的控制器代码如下：

```csharp
    public class ValuesController : ControllerBase
    {
        private readonly IDistributedCache _cache;
        public ValuesController(IDistributedCache cache)
        {
            _cache = cache;
        }

        // GET api/values
        [HttpGet]
        public async Task<IEnumerable<string>> Get()
        {
            await _cache.SetStringAsync("MyValue1", "123");
            await _cache.SetStringAsync("MyValue2", "456");
            string myval1 = await _cache.GetStringAsync("MyValue1");
            string myval2 = await _cache.GetStringAsync("MyValue2");
            return new string[]{ myval1, myval2};
        }
```

运行结果如下：

![](https://img2020.cnblogs.com/blog/1114902/202003/1114902-20200316001622499-750930758.png)





## Using Service Connectors with RabbitMQ

运行rabbitmq
```
docker run -d -p 5672:5672 rabbitmq
```
Steeltoe Initializr 创建.netcore 3.1 + rabbitmq

```
#引用包
Steeltoe.CloudFoundry.ConnectorCore
RabbitMQ.Client

#依赖注入
services.AddRabbitMQConnection(Configuration);
```

添加连接配置

```
 "rabbitmq": {
	"client": {
		"server": "192.168.56.104",
		"port": "5672",
		"username": "guest",
		"password": "guest",
		
	}
 }
```

 注意官方教程中的配置有误，此处为ip配置为server而非host，见Issue：<https://github.com/SteeltoeOSS/steeltoe/issues/263>

controller的样例代码如下，自己发送了五条消息并自己接收打印。

```csharp
public ValuesController(ILogger<ValuesController> logger, [FromServices] ConnectionFactory factory)
        {
            _logger = logger;
            _factory = factory;
        }
        
        // GET api/values
        [HttpGet]
        public ActionResult<string> Get()
        {
            using (var connection = _factory.CreateConnection())
            using (var channel = connection.CreateModel())
            {
                //the queue
                channel.QueueDeclare(queue: queueName,
                             durable: false,
                             exclusive: false,
                             autoDelete: false,
                             arguments: null);
                // consumer
                var consumer = new EventingBasicConsumer(channel);
                consumer.Received += (model, ea) =>
                {
                    string msg = Encoding.UTF8.GetString(ea.Body);
                    _logger.LogInformation("Received message: " + msg);
                };
                channel.BasicConsume(queue: queueName,
                                     autoAck: true,
                                     consumer: consumer);
                // publisher
                int i = 0;
                while (i<5) { //write a message every second, for 5 seconds
                    var body = Encoding.UTF8.GetBytes($"Message {++i}");
                    channel.BasicPublish(exchange: "",
                                         routingKey: queueName,
                                         basicProperties: null,
                                         body: body);
                    Thread.Sleep(1000);
                }
            }
            return "Wrote 5 message to the info log. Have a look!";
        }
```



