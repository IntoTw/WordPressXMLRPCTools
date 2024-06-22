---
title: 'Laravel to Java-应用灰度迁移策略'
date: 2024-06-20T17:31:16+08:00
lastmod: 2024-06-20T17:31:16+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["php转java"]
categories: ["技术"]
lightgallery: true
---

# 生产预发灰度流量方案

# **Prerequisite**

首先我们需要把我们的流量网关从nginx换成apisix

然后我们需要从以前的单套生产环境，增加到两套环境，生产+预发

# **Then**

然后，我们就可以有了这么一个结构图

我们此时有了两套环境，一套预发，一套生产，其中预发的流量通过具体的路由规则配置，目前暂且支持手机号区分。

# Advantage

权衡过多种方案，这种方案做的话，优点：

1. 隔离性强，从网关层直接一分为二。后续环境2套
2. 便于实施，配置文件迁移即可，无需修改
3. 一鱼二吃，既可以生产+预发的灰度，又可以做php+java的灰度
4. 降级优雅，降级时只需要将灰度控制的filter下掉即可
5. 语言友好，java开发的组件总比在nginx上用lua脚本或者c++写脚本要好

缺点：

1. 直接切网关，第一次上线风险较大

2. apisix作为新引入的中间件，虽然已经经过大规模实践并且有良好的社区，但是我们对这个中间件还不够熟悉。





# Apisix

在apissix上，我们要提供什么功能呢？

## **默认的灰度**

首先是默认的灰度，幸运的是apisix的底层是nginx，所以他天然就支持nginx配置，也就是说，我们可以通过配置达到这样一个效果：

![](https://images.intotw.cn/blog/2024/05/49e05d2e03c5bca6844d62b7a52de61e.jpeg)

看起来比较怪。实际上，apisix支持nginx的原生配置，也就是说，我们可以通过在其中配置2个nginx配置文件来监听2个端口，来达到对原来整体nginx集群修改尽可能少的效果。

然后我们通过apisix的插件，来进行对2个nginx入口的灰度，所以就会看起来比较奇怪：一个apisix暴露了3个端口。

实际上，下面的2个ng可以单独部署，不集成在apisix中。这样有好处也有坏处，好处就是不集成的话可以减少性能开销，并且apisix层可以比较简单的拿掉。坏处就是路由多了一层，要管理的中间件变多了，要管理的配置也变多了。

最终实际的执行顺序如下：

apisix->ext-plugin（我们的自定义插件）->traffic-split（根据规则做2个upstreams的转发）->具体的nginx->nginx上的upstream

## **Nginx的配置**



apisix->config.yaml

```java
nginx_config:
    user: root
    http_configuration_snippet: |
        server
        {
                listen       9999;
                server_name  localhost;
                location /test5 {
                        alias   /tmp/folder1/test1;
                        index  index.html;
                }
        }
        server
        {
                listen       8888;
                server_name  localhost;
                location /test5 {
                        alias  /tmp/folder2/test1;
                        index  index.html;
                }
        }
```

## **apisix创建路由**

```
curl http://127.0.0.1:9180/apisix/admin/routes/5 \
-H 'X-API-KEY: Wailiyebang' -X PUT -d '
{
    "uri": "/test5/",
    "plugins": {
        "ext-plugin-pre-req":{
            "_meta": {
                "priority": 10000
            },            
            "conf":[
                {
                    "name":"GrayFilter",
                    "value":"{\"validate_header\":\"token\",\"validate_url\":\"https://www.sso.foo.com/token/validate\",\"rejected_code\":\"403\"}"
                }
            ]
        },
        "file-logger": {
            "_meta": {
                "priority": 1000
            },     
            "path": "logs/file.log"
        },
        "traffic-split": {
            "_meta": {
                "priority": -2000
            },
            "rules": [
                {
                    "match": [
                        {
                            "vars": [
                                ["http_x-api-id","==","1"]
                            ]
                        }
                    ],
                    "weighted_upstreams": [
                        {
                            "upstream": {
                                "name": "upstream-A",
                                "type": "roundrobin",
                                "nodes": {
                                    "127.0.0.1:9999":1
                                }
                            },
                            "weight": 3
                        }
                    ]
                },
                {
                    "match": [
                        {
                            "vars": [
                                ["http_x-api-id","==","2"]
                            ]
                        }
                    ],
                    "weighted_upstreams": [
                        {
                            "upstream": {
                                "name": "upstream-B",
                                "type": "roundrobin",
                                "nodes": {
                                    "127.0.0.1:8888":1
                                }
                            },
                            "weight": 3
                        }
                    ]
                }
            ]
        }
    },
    "upstream": {
            "type": "roundrobin",
            "nodes": {
                "127.0.0.1:9999":1
            }
    }
}'
```



## **插件demo逻辑**

```java
@Component
@Slf4j
public class GrayFilter implements PluginFilter {
    public static int INDEX=1;

    @Override
    public String name() {
        return "GrayFilter";
    }

    @Override
    public void filter(HttpRequest request, HttpResponse response, PluginFilterChain chain) {
      	//灰度逻辑在此可以全部自定义逻辑，只需要打上这个x-api-id的header标即可，这里只是简单模拟一次生产一次灰度，以查看是否达到效果
        request.setHeader("x-api-id", String.valueOf(INDEX % 2+1));
        log.info("GrayFilter filter, index: {}", INDEX);
        INDEX++;
        chain.filter(request, response);
    }
}
```



## 实际效果

![](https://images.intotw.cn/blog/2024/05/e00cc3fe626580898005887feaaccfdf.png)



demo已部署在测试环境lyb-sass-test



## **插件最终逻辑（插件todo）**

![](https://images.intotw.cn/blog/2024/05/cd4f5e028bd7a715b831e469eedac0c6.png)



# tars

tars之间微服务之间调用的问题，我们区分环境通过tars的set功能做区分：

https://doc.tarsyun.com/#/dev/tars-idc-set.md

![](https://images.intotw.cn/blog/2024/05/26c324b752a59f6c266aa9cf0d8e3a30.png)

这样可以保证生产以及预发之间微服务的隔离



# 演进过程

## 迭代方式

java应用承接迭代，系统与php1:1，初次承接需要发布新java应用，后续同步更新java应用或许在java应用中修改新增接口即可

## 测试环境

测试环境按照预发环境配置，作为预发的前置验证，测试环境中在不验证灰度插件功能的情况下，可以不配置灰度插件，默认流量全走php+java混合的环境

## **发布流程**

发布流程需要做一些调整。

原先我们直接发布到生产环境，现在我们迭代需要先发布到预发环境，在预发环境验证后，再发布到生产环境。

后续我们需要对于发布步骤进行记录（开发需要提供发布操作步骤，其中需要动到线上库的需要评估生产不发的情况下对生产的影响），预发由产品+客成+指定的灰度流量验证过后，通过发布操作步骤再次发布到生产。

预发到生产的时间可以视迭代或者具体情况具体再定，比如纯php修改可以只在预发验证下就发布到生产，包含java应用新上或者java修改的，最好在预发进行充分验证



## **回滚策略**

1. 生产发现因为发布导致的问题，将应用以及灰度配置回滚到上个版本即可
