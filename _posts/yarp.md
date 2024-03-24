---
layout: post
title: YARP
date: 2024-03-24
categories: test
tags: yarp 
---

# 入门

##  简介

YARP  被设计为一个库，他提供核心的代理功能，然后您可以通过添加或替换模块来自定义。

## 创建新项目

```sh
dotnet new webapi -n MyProxy -f net8.0
```

或者在 vs 中创建新的 ASP.NET Core WebAPI 应用程序。

## 添加项目引用

```
<ItemGroup> 
 <PackageReference Include="Yarp.ReverseProxy" Version="2.1.0" />
</ItemGroup> 
```

或者通过vs 添加nuget 包 Yarp.ReverseProxy

## 添加 YARP 中间件

```c#
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
//注册 YARP 代理服务
builder.Services.AddReverseProxy()        .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));
var app = builder.Build();
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
app.UseAuthorization();
app.MapControllers();
//添加 YARP 中间件
app.MapReverseProxy();
app.Run();
```

## 配置

```json
  "ReverseProxy": {
      //路由
    "Routes": { 
        //百度路由
      "baidu_route": {
          //集群
        "ClusterId": "baidu_cluster",
          //规则
        "Match": {
          "Path": "{**catch-all}"
        }
      }
    },
      //集群
    "Clusters": {
        //百度集群
      "baidu_cluster": {
        "Destinations": {
          "destination1": {
            "Address": "https://www.baidu.com"
          }
        }
      }
    }
  }
```

## 运行项目

![image-20240324102214846](C:\Users\80977\AppData\Roaming\Typora\typora-user-images\image-20240324102214846.png)

# 配置文件

## 介绍

反向代理可以使用 Microsoft.Extensions 中的 IConfiguration 抽象从文件加载路由和群集的配置。此处给出的示例使用 JSON，但任何 IConfiguration 源都应该有效。如果源文件发生更改，配置也将在不重新启动代理的情况下更新。

## 加载配置

若要从 IConfiguration 加载代理配置，请在 Program.cs 中添加以下代码：

```c#
// Add the reverse proxy capability to the server
builder.Services.AddReverseProxy().LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

```



## 多个配置源

YARP 支持从多个源加载代理配置。LoadFromConfig 可以多次调用引用不同的 IConfiguration 部分，也可以与不同的配置源（如 InMemory）结合使用。路由可以引用来自其他源的集群。

```C#
services.AddReverseProxy()
    .LoadFromConfig(Configuration.GetSection("ReverseProxy1"))
    .LoadFromConfig(Configuration.GetSection("ReverseProxy2"));
```

或者

```C#
services.AddReverseProxy()
    .LoadFromMemory(routes, clusters)
    .LoadFromConfig(Configuration.GetSection("ReverseProxy"));
```

```
注意：不支持为给定路由或集群合并来自不同来源的部分配置。
```

## 配置结构

该配置由您在上面通过 指定的命名部分组成，并包含路由和集群的子部分。

```json
{
  "ReverseProxy": {
    "Routes": {
      "route1" : {
        "ClusterId": "cluster1",
        "Match": {
          "Path": "{**catch-all}",
          "Hosts" : [ "www.aaaaa.com", "www.bbbbb.com"],
        },
      }
    },
    "Clusters": {
      "cluster1": {
        "Destinations": {
          "cluster1/destination1": {
            "Address": "https://example.com/"
          }
        }
      }
    }
  }
}
```



## 路由

路由部分是路由匹配项及其关联配置的无序集合。路由至少需要以下字段：

- RouteId - 唯一名称
- ClusterId - 引用 clusters 部分中条目的名称。
- Match - 包含 Hosts 数组或 Path 模式字符串。Path 是一个 ASP.NET Core 路由模板，可以按[此处的说明](https://docs.microsoft.com/aspnet/core/fundamentals/routing#route-templates)进行定义。 路由匹配基于具有最高优先级的最具体路由，[如此处](https://docs.microsoft.com/aspnet/core/fundamentals/routing#url-matching)所述。可以使用该字段实现显式排序，值越低越优先级。`order`

可以在每个路由条目上配置[标头](https://microsoft.github.io/reverse-proxy/articles/header-routing.html)、[授权](https://microsoft.github.io/reverse-proxy/articles/authn-authz.html)、[CORS](https://microsoft.github.io/reverse-proxy/articles/cors.html) 和其他基于路由的策略。有关其他字段，请参阅[路由配置](https://microsoft.github.io/reverse-proxy/api/Yarp.ReverseProxy.Configuration.RouteConfig.html)。

代理将应用给定的匹配条件和策略，然后将请求传递到指定的集群。

## 集群

clusters 部分是命名集群的无序集合。集群主要包含命名目标及其地址的集合，其中任何一个都被认为能够处理给定路由的请求。代理将根据路由和集群配置处理请求，以便选择目标。

有关其他字段，请参阅[集群配置](https://microsoft.github.io/reverse-proxy/api/Yarp.ReverseProxy.Configuration.ClusterConfig.html)。



## 所有的配置属性



```json
{
  // Base URLs the server listens on, must be configured independently of the routes below
  "Urls": "http://localhost:5000;https://localhost:5001",
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      // Uncomment to hide diagnostic messages from runtime and proxy
      // "Microsoft": "Warning",
      // "Yarp" : "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "ReverseProxy": {
    // 路由告诉代理要转发的请求
    "Routes": {
      "minimumroute" : {
        //匹配任何内容并将其路由到www.example.com
        "ClusterId": "minimumcluster",
        "Match": {
          "Path": "{**catch-all}"
        }
      },
      "allrouteprops" : {
        // 匹配/某物/*并路由到“allclusterrops”
        "ClusterId": "allclusterprops", // 其中一个群集的名称
        "Order" : 100, // 数字越低优先级越高
        "MaxRequestBodySize" : 1000000, // 以字节为单位。服务器限制的可选覆盖（默认为30MB）。设置为-1可禁用。
        "AuthorizationPolicy" : "Anonymous", //策略名称或“默认”、“匿名”
        "CorsPolicy" : "Default", //要应用到此路由的CorsPolicy的名称或“Default”、“Disable”
        "Match": {
          "Path": "/something/{**remainder}", //使用ASP匹配的路径。NET语法。
          "Hosts" : [ "www.aaaaa.com", "www.bbbbb.com"], //要匹配的主机名（未指定）为任意
          "Methods" : [ "GET", "PUT" ], //匹配的HTTP方法，usspecified是全部
          "Headers": [ //要匹配的标头，未指定为任意标头
            {
              "Name": "MyCustomHeader", //标头的名称
              "Values": [ "value1", "value2", "another value" ], //匹配项与这些值中的任何一个都不匹配
              "Mode": "ExactHeader", //或“HeaderPrefix”、“Exists”、“Contains”、”NotContains“、”NotExists“
              "IsCaseSensitive": true
            }
          ],
          "QueryParameters": [ //要匹配的查询参数，未指定为任意
            {
              "Name": "MyQueryParameter", //查询参数的名称
              "Values": [ "value1", "value2", "another value" ], //匹配项与这些值中的任何一个都不匹配
              "Mode": "Exact", //或“前缀”、“存在”、“包含”、“不包含”
              "IsCaseSensitive": true
            }
          ]
        },
        "MetaData" : { //自定义扩展可以使用的键值对列表
          "MyName" : "MyValue"
        },
        "Transforms" : [ //转换列表。有关详细信息，请参阅Transforms文章
          {
            "RequestHeader": "MyHeader",
            "Set": "MyValue",
          }
        ]
      }
    },
    //集群告诉代理在哪里以及如何转发请求
    "Clusters": {
      "minimumcluster": {
        "Destinations": {
          "example.com": {
            "Address": "http://www.example.com/"
          }
        }
      },
      "allclusterprops": {
        "Destinations": {
          "first_destination": {
            "Address": "https://contoso.com"
          },
          "another_destination": {
            "Address": "https://10.20.30.40",
            "Health" : "https://10.20.30.40:12345/test" //覆盖活动的健康检查
          }
        },
        "LoadBalancingPolicy" : "PowerOfTwoChoices", //或者选择"FirstAlphabetical", "Random", "RoundRobin", "LeastRequests"
        "SessionAffinity": {
          "Enabled": true, //默认为“false”
          "Policy": "Cookie", // 默认值，或者“CustomHeader”
          "FailurePolicy": "Redistribute", //默认值，或者“Return503Error”
          "Settings" : {
              "CustomHeaderName": "MySessionHeaderName" // 默认为 'X-Yarp-Proxy-Affinity`
          }
        },
        "HealthCheck": {
          "Active": { //进行API调用以验证运行状况。
            "Enabled": "true",
            "Interval": "00:00:10",
            "Timeout": "00:00:10",
            "Policy": "ConsecutiveFailures",
            "Path": "/api/health" //要查询运行状况状态的api终结点
          },
          "Passive": { //基于HTTP响应代码禁用目标
            "Enabled": true, //默认为false
            "Policy" : "TransportFailureRateHealthPolicy", // Required
            "ReactivationPeriod" : "00:00:10" // 10s
          }
        },
        "HttpClient" : { //用于联系目的地的HttpClient实例的配置
          "SSLProtocols" : "Tls13",
          "DangerousAcceptAnyServerCertificate" : false,
          "MaxConnectionsPerServer" : 1024,
          "EnableMultipleHttp2Connections" : true,
          "RequestHeaderEncoding" : "Latin1", //如何解释请求标头值中的非ASCII字符
          "ResponseHeaderEncoding" : "Latin1" //如何解释响应头值中的非ASCII字符
        },
        "HttpRequest" : { //用于将请求发送到目标的选项
          "ActivityTimeout" : "00:02:00",
          "Version" : "2",
          "VersionPolicy" : "RequestVersionOrLower",
          "AllowResponseBuffering" : "false"
        },
        "MetaData" : { // //自定义键值对
          "TransportFailureRateHealthPolicy.RateLimit": "0.5", //被动健康策略使用
          "MyKey" : "MyValue"
        }
      }
    }
  }
}
```

