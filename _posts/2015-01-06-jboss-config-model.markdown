---
layout: post
title:  "JBoss Configuration Model"
date:   2015-01-06
categories: jekyll update
---

今天完成并验证了 `jboss-config` 的原型，剩下的都是往上堆代码加功能的体力活了。

要解决的问题是这样的：

* 安装 JBoss 包至目标系统
* 在目标系统配置 JBoss 的多个运行实例
* 每个运行实例都是裁剪过的 `jboss-subsystem` 
* 由可配置的输入选项控制裁剪过程
* 自动化以上过程

用户界面是这样的：

1. Install jboss jboss-config to target system
2. Generate input options for each profile
3. Activate the profile
4. As a result, jboss instance started along with system boot

建立模型如下：

* _ServerConf_ encapsulates server configuration data
    - Location of the JVM (JAVA_HOME)
    - Location of the JBoss Binary Package (JBOSS_HOME)
    - Location of the JBoss Runtime (JBOSS_BASE_DIR)
    - Default input options 
        + Management username/password
        + Management port
        + EJB port
        + JVM arguments
        + Network access points
        + Deployment path
* _ClusterConf_ encapsulates cluster configuration data
    - Node network interfaces
        + Internal interface
        + External interface
* _ServerProfile_ abstracts the actions
    - Activate profile
        + Create profile
        + Create user
        + Bootstrap
        + Online configuration
    - Deactivate profile
    - Reconfigure profile

最后给出[原型代码](https://github.com/aclisp/archived/blob/master/jboss-config/src/main/python/jboss_config/apitools/srvconf.py)，在 RHEL 6 和 JBoss EAP 6.2 上验证通过。

应用服务器的 [Microservice](http://en.wikipedia.org/wiki/Microservices) 化是一种趋势，从 [Spring Guide](http://spring.io/guides/gs/rest-service/) 的示例代码中使用了 `spring-boot-starter-web` 中可见一斑。我们的 JBoss 服务器也在向这个方向演进。

