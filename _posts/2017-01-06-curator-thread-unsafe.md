---
layout: post
title: Apache Curator��һ��������̲߳���ȫ����
---

�ȿ������ͼ

![screenshot](/img/2017-01-06_140232.png)

`semaphore.acquire`����`lease.close`�󷵻أ����غ�ͬʱ�������̶߳��ڶ�`this.lease`���������޸ģ����������`acquire`�����ڷ���`true`ʱ��`this.lease`�����п�����`null`�ģ�����

�����ڽ�������`release`ʱ���޷���֮ǰ��ȡ����`lease`�������`lease.close`�ģ��������Զ�Ķ��ˡ�

������������µ�curator-recipes�϶����ɴ��ڣ�https://github.com/apache/curator/blob/master/curator-recipes/src/main/java/org/apache/curator/framework/recipes/locks/InterProcessSemaphoreMutex.java

```
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>3.2.1</version>
</dependency>
```

�޸ĵķ����ܼ򵥣����ϴ��룺

![screenshot](/img/2017-01-06_150339.png)

ע�⵽�����Ȱ�`this.lease`��Ϊ�ֲ�������Ȼ�����̰�`this.lease`��Ϊ`null`�ͷŵ���������ȥ����`lease.close`������`this.lease`���������ʼ��ֻ��һ���߳����޸ģ��̰߳�ȫ�ˡ�

���⼰��ز����Ѿ������������https://issues.apache.org/jira/browse/CURATOR-376