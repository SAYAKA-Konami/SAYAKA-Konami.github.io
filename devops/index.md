# Devops


# MCSW后端项目的自动化打包部署

## 借助阿里的容器镜像服务

> 阅读该篇文章时，你应当具有docker的初步使用经验。如简单的运行、编写Dockerfile打包等。

[点击这里创建吧](https://cr.console.aliyun.com/cn-heyuan/instance/repositories)

> 该服务是免费的，普通用户可以创建三个命名空间。每个命名空间可以创建1000个镜像服务

[这里是相应的官方指导文档](https://help.aliyun.com/document_detail/60951.html)

## 我踩的坑

由于我的项目是微服务项目，每个模块都是一个独立的服务。所以如果直接跟着官方给出的`Dockerfile`进行打包会出现各种各样的问题。

如下是我的项目结构：

![image-20230119205557652](/mcsw项目结构.png)

除了`common`模块中的内容是一些公共的依赖之外，其他模块都是一个独立的服务。

> [项目地址戳这里](https://github.com/SAYAKA-Konami/MCSW)

在阿里文档给出的镜像构建指导中，它建议我们利用***Docker***在17.05版本之后推出的多阶段构建（***multistage build***）。在文档中给出的`Dockerfile`的内容如下：

```dockerfile
# 这其实是自己构建的Maven镜像。
FROM demo-registry-vpc.cn-beijing.cr.aliyuncs.com/demo/maven-base:3.8-openjdk-8 AS builder

# add pom.xml and source code
ADD ./pom.xml pom.xml
ADD ./service service/

# package jar
RUN mvn clean package

# Second stage: minimal runtime environment
From demo-registry-vpc.cn-beijing.cr.aliyuncs.com/demo/openjdk:8-jre-alpine

# copy jar from the first stage
COPY --from=builder service/target/service-1.0-SNAPSHOT.jar service-1.0-SNAPSHOT.jar

EXPOSE 8080

CMD ["java", "-jar", "service-1.0-SNAPSHOT.jar"]
```

从第一个`From`开始到第二个`From`之间，docker会根据自行构建好的**maven**镜像来下载项目所需的所有依赖。

而第二个`From`开始到结束才是真正在构建我们所要发布的服务的镜像。

### 踩到的坑

在上述的`Dockerfile`中，我们可以看到第一阶段需要拷贝项目中的`pom.xml`文件到容器中。如果将相应的`Dockerfile`放置在对应的模块下，==我并不知道如何拷贝父`pom`文件到容器中==。此后在构建过程中在执行`mvn install`的时候就会提示找不到父`pom`的错误。

所以我的做法是将每个模块对应的`Dockerfile`放在根目录之下，形成的Dockerfile如下：

```dockerfile
# First stage: complete build environment
FROM nanwu/mcsw-maven:v6 AS builder

# 拷贝整个项目相关的pom文件到容器中
ADD  pom.xml pom.xml
ADD mcsw-cloud-account/pom.xml mcsw-cloud-account/pom.xml
ADD mcsw-cloud-common/pom.xml mcsw-cloud-common/pom.xml
ADD mcsw-cloud-gateway/pom.xml mcsw-cloud-gateway/pom.xml
ADD mcsw-cloud-homepage/pom.xml mcsw-cloud-homepage/pom.xml
ADD mcsw-cloud-offer/pom.xml mcsw-cloud-offer/pom.xml
ADD mcsw-cloud-post/pom.xml mcsw-cloud-post/pom.xml
# To resolve dependencies in a safe way (no re-download when the source code changes)
RUN  cd mcsw-cloud-account;mvn install

ADD mcsw-cloud-account/src mcsw-cloud-account/src

# package jar
RUN cd mcsw-cloud-account;mvn clean package

# Second stage: minimal runtime environment
From openjdk:8-jre-alpine

# copy jar from the first stage
COPY --from=builder mcsw-cloud-account/target/mcsw-cloud-account-1.0-SNAPSHOT.jar mcsw-cloud-account-1.0-SNAPSHOT.jar

EXPOSE 8010
# 以下是启动时指定的JVM参数
ENV JAVA_OPTS="\
-server \
-Xms128m\
-Xmx128m\
-Xss128k\
-XX:+UseConcMarkSweepGC\
-XX:+UseCMSCompactAtFullCollection\
-XX:CMSFullGCsBeforeCompaction=0\
-XX:+HeapDumpOnOutOfMemoryError\
-XX:HeapDumpPath=/ \
-Xloggc:/gc.log \
-XX:+PrintGC\
-XX:+PrintGCDetails\
-XX:+PrintGCTimeStamps"


ENTRYPOINT java ${JAVA_OPTS} -Djava.security.egd=file:/dev/./urandom -jar /mcsw-cloud-account-1.0-SNAPSHOT.jar

```

# JVM参数的设定

## CMS（Concurrent Mark Sweep）  🛠️ G1（Garbage First）

这其实是一个题外话，但是我还是忍不住想要拿出来说一下。

细心的小伙伴可能看到了我VM参数设置得很奇怪。首先对于Java这种内存大户来讲，内存的阈值居然限制在了**100M**。而且还配合了很多CMS相关的参数。

其实这只是出于无奈之举。还请听我辩解：

1. 我希望我的毕设能够投入使用，哪怕只作用于我们学院内的一部分人。而且我也非常希望能得到我们学院同学的一些就业相关的数据，看一下当下这个“互联网寒冬”究竟能让一个双非一本的学生陷入怎样的境地。
2. 我所购买的云服务器是腾讯云的新人特惠，50块钱2核2G——很遗憾的是现在两大云厂商都下架了如此具有性价比的服务器。在还要启用`Nacos`作为服务发现的情形下，我的服务器的内存可用资源捉襟见肘。所以我不得不将各个服务端内存限制在了一个窘迫的地步——这其实也是一个血的教训。如果你想用Java学习并开发一套微服务架构的项目的话，机器资源是一个你必须考虑的点。
3. 在周志明所著的《深入理解Java虚拟机》一文中写道：当运行内存在6G-8G这个点上G1和CMS的表现可能相差无几（书发版的时间为2020年）。而我所拥有的运行内存远低于这个阈值。所以我毫不犹豫的选择了CMS。

### 所以G1和CMS哪个更好？（以下仅是我个人的观点）

我会毫不犹豫的支持G1。CMS已经是被官方所抛弃，在JDK9中被标记为过期，在JDK14中被正式移除。如果你有幸用上了现今的长期支持版JDK17，那么你要考虑的应当是在G1、Shenandoah、ZGC中选择一个合适的GC。

#### 为啥G1得在大内存相比CMS才会有更好的表现？

由于我们大部分人所采用的环境依然是JDK8。G1虽然很早的就加入了JER的计划之中，不过在JDK8中，对比CMS，G1显然还是一个新生儿。其内存布局采用的是分块的思想，配合相应的预测算法。G1能在回收时做一定的取舍——==优先，或者只==回收垃圾比例大的“块”。以此来控制GC时间。这也是为什么它叫 Garbage First的缘由。

![img](/G1内存模型.png)

但是这是有代价的。分块导致了跨代引用的记录变成了跨“块”引用的记录。G1需要为每一个块都维护一个`Remember Set`从而来记录其他“块”中对自身的引用的指针。所以G1对于CMS或者更早的Parallel Scavenge都具有更高的空间负载。这个负载通常会占堆大小的==20%甚至更多==。所以在内存资源较少的情况下，还是尽可能的避免使用新生的G1。


