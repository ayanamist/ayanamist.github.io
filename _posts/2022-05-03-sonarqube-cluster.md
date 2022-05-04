---
layout: post
title: 如何合法的白嫖SonarQube的集群功能
---

本文基于 8.9 LTS 版本，文档地址 https://docs.sonarqube.org/8.9/ 代码地址 https://github.com/SonarSource/sonarqube/tree/branch-8.9

目前SonarQube的社区版是只支持单机部署的，支持集群部署的只有最贵的DataCenter版，但厂里的规模，单机完全扛不住，也不符合高可用的要求，但国内的国情，也不可能花钱去买DataCenter版，于是只好看看有什么曲线救国的路子。

强行将SonarQube以集群模式的配置启动，会在日志里看到有一些报错，就知道有一些暗桩，这里直接总结一下。

# SonarQube暗桩的大体架构

SonarQube在8.9版本还是用的[Pico](http://picocontainer.com/) 作为DI框架，这个框架挺小众的，最好有一些了解。

而暗桩注入则是用了JDK自带的[ServiceLoader](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html)的SPI机制，通过 `org.sonar.core.extension.CoreExtension` 这个service进行注入，社区版只有加载service的代码，但没有具体实现，需要自行反编译商业版的逻辑，商业版中对内核引擎部分的改动，都是通过这个机制进行注入的。

# org.sonar.server.platform.ClusterFeature

需要有一个插件实现这个接口，这个接口很简单，就是 `boolean isEnabled();` 这样一个简单的接口，直接写死为 `return true;` 就好了。

```java
package com.example.sonarqube;

import org.sonar.server.platform.ClusterFeature;

public class MyClusterFeature implements ClusterFeature {
    public boolean isEnabled() {
        return true;
    }
}

```

# org.sonar.process.cluster.hz.HazelcastMember

另一个暗桩比较麻烦，是缺少这个类的实现，虽然hazelcast只在app节点使用，但却在所有节点都需要被注入这个对象。反编译并美化变量名的大体代码为：

```java
package com.example.sonarqube;

import com.hazelcast.cluster.Cluster;
import com.hazelcast.cluster.MemberSelector;
import com.hazelcast.cp.IAtomicReference;
import org.jetbrains.annotations.NotNull;
import org.sonar.api.Startable;
import org.sonar.api.ce.ComputeEngineSide;
import org.sonar.api.config.Configuration;
import org.sonar.api.server.ServerSide;
import org.sonar.process.NetworkUtils;
import org.sonar.process.ProcessId;
import org.sonar.process.ProcessProperties;
import org.sonar.process.cluster.hz.DistributedAnswer;
import org.sonar.process.cluster.hz.DistributedCall;
import org.sonar.process.cluster.hz.DistributedCallback;
import org.sonar.process.cluster.hz.HazelcastMember;
import org.sonar.process.cluster.hz.HazelcastMemberBuilder;
import org.sonar.process.cluster.hz.InetAdressResolver;

import java.util.Arrays;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.locks.Lock;

@ComputeEngineSide
@ServerSide
public class MyHazelcastMember implements HazelcastMember, Startable {
    private static final String WEB_PORT = "sonar.cluster.node.web.port";

    private static final String CE_PORT = "sonar.cluster.node.ce.port";

    private final Configuration configuration;

    private final NetworkUtils networkUtils;

    private HazelcastMember hzMember = null;

    public MyHazelcastMember(Configuration configuration, NetworkUtils networkUtils) {
        this.configuration = configuration;
        this.networkUtils = networkUtils;
    }

    public <E> IAtomicReference<E> getAtomicReference(@NotNull String name) {
        return getHzMember().getAtomicReference(name);
    }

    public <K, V> Map<K, V> getReplicatedMap(@NotNull String name) {
        return getHzMember().getReplicatedMap(name);
    }

    public UUID getUuid() {
        return getHzMember().getUuid();
    }

    public Set<UUID> getMemberUuids() {
        return getHzMember().getMemberUuids();
    }

    public Lock getLock(@NotNull String name) {
        return getHzMember().getLock(name);
    }

    public long getClusterTime() {
        return getHzMember().getClusterTime();
    }

    public Cluster getCluster() {
        return getHzMember().getCluster();
    }

    public <T> DistributedAnswer<T> call(@NotNull DistributedCall<T> callable, @NotNull MemberSelector memberSelector, long timeoutMs) throws InterruptedException {
        return getHzMember().call(callable, memberSelector, timeoutMs);
    }

    public <T> void callAsync(@NotNull DistributedCall<T> callable, @NotNull MemberSelector memberSelector, @NotNull DistributedCallback<T> callback) {
        getHzMember().callAsync(callable, memberSelector, callback);
    }

    private HazelcastMember getHzMember() {
        return Objects.requireNonNull(this.hzMember, "Hazelcast member not started");
    }

    public void close() {
        if (this.hzMember != null) {
            this.hzMember.close();
            this.hzMember = null;
        }
    }

    public void start() {
        if (!isClusterEnabled())
            return;
        ProcessId processId = ProcessId.fromKey(this.configuration.get("process.key").orElseThrow(() -> new IllegalStateException("Missing process key")));
        String nodeHost = this.configuration.get(ProcessProperties.Property.CLUSTER_NODE_HOST.getKey()).orElseThrow(() -> new IllegalStateException("Missing node host"));
        this.hzMember = (new HazelcastMemberBuilder(new InetAdressResolver()))
                .setNodeName(this.configuration.get(ProcessProperties.Property.CLUSTER_NODE_NAME.getKey()).orElseThrow(() -> new IllegalStateException("Missing node name")))
                .setPort(getPort(processId, nodeHost))
                .setProcessId(processId)
                .setMembers(Arrays.asList(this.configuration.getStringArray(ProcessProperties.Property.CLUSTER_HZ_HOSTS.getKey())))
                .setNetworkInterface(nodeHost)
                .build();
    }

    private int getPort(ProcessId processId, String hostOrAddress) {
        switch (processId) {
            case WEB_SERVER:
                return fixPortIfZero(WEB_PORT, hostOrAddress);
            case COMPUTE_ENGINE:
                return fixPortIfZero(CE_PORT, hostOrAddress);
            default:
                throw new IllegalStateException(String.format("Incorrect processId [%s]", processId.getKey()));
        }
    }

    private int fixPortIfZero(String portKey, String hostOrAddress) {
        Optional<Integer> optional = this.configuration.getInt(portKey);
        return optional.orElseGet(() -> this.networkUtils.getNextAvailablePort(hostOrAddress).orElseThrow(() -> new IllegalStateException(String.format("Failed to find a free port for property '%s' for configured host: %s", portKey, hostOrAddress))));
    }

    private boolean isClusterEnabled() {
        Optional<Boolean> optional = this.configuration.getBoolean(ProcessProperties.Property.CLUSTER_ENABLED.getKey());
        return (optional.isPresent() && optional.get() == Boolean.TRUE);
    }

    public void stop() {
        close();
    }
}
```

# 注入

增加下面的文件作为注入入口

```java
package com.example.sonarqube;

import org.sonar.core.extension.CoreExtension;
import org.sonar.api.SonarQubeSide;

public class ClusterExtension implements CoreExtension {
    @Override
    public String getName() {
        return "cluster";
    }

    @Override
    public void load(Context context) {
        if (context.getRuntime().getSonarQubeSide() == SonarQubeSide.COMPUTE_ENGINE) {
            context.addExtension(MyHazelcastMember.class);
        } else if (context.getRuntime().getSonarQubeSide() == SonarQubeSide.SERVER) {
            context.addExtensions(MyClusterFeature.class, MyHazelcastMember.class);
        }
    }
}
```

在resources目录里新增 `META-INF/services/org.sonar.core.extension.CoreExtension` 文件，内容为 `com.example.sonarqube.ClusterExtension`

需要将这些类和文件，都追加到SonarQube官方的jar文件中，因为classpath是写死的，当然修改启动脚本去增加classpath也是可以的。

这样一套操作下来，就能以社区版的代码，启动集群模式了。

其实社区的[branch插件](https://github.com/mc1arke/sonarqube-community-branch-plugin)里要求注入的javaagent，也是做了类似的事情，只不过是通过javaagent的方式进行修改。

# ElasticSearch的配置

其实集群模式，主要也就分成了两类节点，app和search，其中app节点上运行两种任务，web和computeengine，都是无状态的，运维比较简单，search节点上其实就是跑了个pre-configured的ElasticSearch节点，所以如果有托管版的ES可以用，完全可以在app的配置文件里将请求指向托管版额ES，而不用去弄什么search节点，因为SonarQube对search节点的配置其实是有问题的，主要体现在 `cluster.initial_master_nodes` 这个配置项上。

按[ES的文档](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/modules-discovery-bootstrap-cluster.html)，这个配置项只应该在ES集群第一次搭建时使用，因为会要求里面配置的所有节点都存活且正常参与投票，集群一旦搭建成功，这个配置项除了产生干扰，就没有任何正向作用了。

但SonarQube的配置，却是始终将所有search节点都加入到这个配置项中，就要求所有search节点都必须100%存活，这对运维要求实在太高而且完全没有必要。

这块只能修改SonarQube的代码 `org.sonar.application.es.EsSettings` 这个类，我是新增了 `sonar.cluster.es.initial_master_nodes` 这样一个配置项来单独配置解决这个问题。

# 编译

从上面能看到，其实有一些改动是需要修改SonarQube代码的，但如果用了maven工程的形式，就势必要解决编译问题，但改动的类都是没有官方maven依赖的，虽然本地能添加一个虚假的maven依赖，但如果要交接给其他人维护终究是不方便，最终我是采用了，将SonarQube官方的fat-jar文件直接作为三方库的方式上传到厂里的maven仓库来解决的

```bash
mvn deploy:deploy-file -DgroupId=org.sonarsource.sonarqube -DartifactId=sonarqube-community -Dversion=8.9.7-SNAPSHOT -Dpackaging=jar -Dfile=sonar-application-8.9.7.52159.jar -DrepositoryId=snapshots -Durl=http://xxxxx/mvn/snapshots
```

这样就可以在maven项目中引入如下依赖来解决编译问题了

```xml
<dependency>
    <groupId>org.sonarsource.sonarqube</groupId>
    <artifactId>sonarqube-community</artifactId>
    <version>8.9.7-SNAPSHOT</version>
    <scope>provided</scope>
</dependency>
```