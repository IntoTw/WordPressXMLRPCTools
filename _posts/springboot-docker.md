---
title: '使用jib-maven-plugin将Spring Boot项目发布为Docker镜像'
date: 2020-10-14T00:00:00+08:00
lastmod: 2020-10-14T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["spring-boot","docker"]
categories: ["技术"]
lightgallery: true
---


## 介绍
将spring boot(cloud)项目发布到docker环境作为镜像，一般常用的一个是com.spotify的docker-maven-plugin这个maven插件，还有一个就是本文介绍的了，本文介绍的jib-maven-plugin是谷歌提供的，且配置较为简单（相对的镜像自定义能力较弱）。

## 使用
增加如下配置即可：
```xml
<build>
        <finalName>${artifactId}</finalName>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>com.google.cloud.tools</groupId>
                    <artifactId>jib-maven-plugin</artifactId>
                    <version>2.1.0</version>
                    <configuration>
                        <!--配置基本镜像，这里可以修改为自己的镜像，或精简或修改，但是一定要在私有库中有-->
                        <from>
                            <image>java:8</image>
                        </from>
                        <!--配置最终推送的地址，仓库名，镜像名-->
                        <to>
                            <tags>
                                <!--tag即镜像的版本，一般是覆盖latest并且新增一个当前版本号-->
                                <tag>latest</tag>
                                <tag>${version}</tag>
                            </tags>
                            <!--配置私有仓库地址-->
                            <image>10.10.2.62:5000/v2/${project.build.finalName}</image>
                        </to>
                        <allowInsecureRegistries>true</allowInsecureRegistries>
                        <container>
                            <!--jvm内存参数,jvm启动时的所有参数都可以在这里增加-->
                            <jvmFlags>
                                <jvmFlag>-Xms512m</jvmFlag>
                                <jvmFlag>-Xmx512m</jvmFlag>
                                <jvmFlag>-Duser.timezone=GMT+08</jvmFlag>
                                <jvmFlag>-Dfile.encoding=UTF8</jvmFlag>
                            </jvmFlags>
                            <!--要暴露的端口-->
                            <ports>
                                <port>8761</port>
                            </ports>
                            <!--修改镜像默认时间，否则会导致镜像内时区问题-->
                            <creationTime>USE_CURRENT_TIMESTAMP</creationTime>
                        </container>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
```

## 总结
这种方式比dockerFile简单，但是也不灵活，适合简单项目。