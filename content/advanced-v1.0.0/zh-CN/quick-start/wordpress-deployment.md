---
title: "示例二 - 部署 Wordpress" 
---

本文以创建一个部署 (Deployment) 为例，部署一个无状态的 Wordpress 应用，最终部署一个外网可访问的 [Wordpress](https://wordpress.org/) 网站。Wordpress 连接 MySQL 数据库的密码将以 [配置 (ConfigMap)](../../configuration/configmaps) 的方式进行创建和保存。

## 前提条件

- 已创建了有状态副本集 MySQL，若还未创建请参考 [快速入门 - 部署 MySQL](../mysql-deployment)。

## 部署 Wordpress

### 第一步：创建配置

Wordpress 的环境变量 `WORDPRESS_DB_PASSWORD` 即 Wordpress 连接数据库的密码属于敏感信息，不适合以明文的方式表现在步骤中，因此以创建配置 (ConfigMap) 的方式来代替该环境变量。创建的配置将在创建 Wordpress 的容器组设置时作为环境变量写入。

![创建配置](/wordpress-create-configmap.png)

1.1. 在当前项目下左侧菜单栏的 **配置中心** 选择 **配置**，点击创建，基本信息如下：

- 名称：作为 Wordpress 容器中环境变量的名称，填写 `wordpress-configmap`。
- 别名：帮助您更好的区分资源，可备注为 `连接 Mysql 密码`。
- 描述信息：mysql password。


1.2. ConfigMap 是以键值对的形式存在，因此键值对设置为 `WORDPRESS_DB_PASSWORD` 和 `123456`。

![创建 ConfigMap](/wordpress-configmap.png)

### 第二步：创建存储卷

2.1. 在当前项目下左侧菜单栏的 **存储卷**，点击创建，基本信息如下，

- 名称：wordpress-pvc
- 描述信息：Wordpress 持久化存储卷


2.2. 下一步，存储卷设置中，参考如下填写：

- 存储类型：选择预先为集群创建的存储类型，示例中是 **csi-qingcloud**
- 访问模式：选择单节点读写 (RWO)
- 存储卷容量：默认 10 Gi

2.3. 下一步，设置标签为 `app: wordpress`，点击创建。

可以看到存储卷 wordpress-pvc 已经创建成功，状态是 “准备就绪”，可挂载至工作负载。

![创建存储卷](/wordpress-pvc-list.png)

### 第三步：创建部署

登录 KubeSphere 控制台，进入已创建的项目，选择 **工作负载 → 部署**，进入列表页，点击 **创建部署**。

![创建部署](/wordpress-create-deployment.png)

### 第四步：填写基本信息

基本信息中，参考如下填写：

- 名称：必填，起一个简洁明了的名称，便于用户浏览和搜索，此处填写 `wordpress`
- 别名：可选，支持中文帮助更好的区分资源，此处填写 `Wordpress 网站`
- 更新策略：选择 RollingUpdate
   - MaxSurge：默认 25%， MaxSurge 指定的是除了 "期望副本 (DESIRED)" 数量之外，在一次滚动更新中，部署控制器还可以创建多少个新的容器组
   - MaxUnavailable：默认 25%，表示最多可以一次删除 "25% * 期望副本数量" 个容器组
- 描述信息：简单介绍该工作负载，方便用户进一步了解

![填写基本信息](/wordpress-basic.png)

### 第五步：容器组模板

容器组模板中，名称可自定义，镜像填写 `wordpress:4.8-apache`，展开高级设置，对 **端口** 和 **环境变量** 进行设置，其它项暂不作设置。参考如下填写，完成后点击 **保存**：

- CPU 和内存：此处暂不作限定，将使用在创建项目时指定的默认请求值。
- 端口：名称可自定义，选择 `TCP` 协议，填写 Wordpress 在容器内的端口 `80`。主机端口是容器映射到主机上的端口，此处暂不设置主机端口。
- 环境变量：这里需要添加两个环境变量
   -  点击 **引用配置中心**，名称填写 `WORDPRESS_DB_PASSWORD`，选择在第一步创建的配置 (ConfigMap) `wordpress-configmap` 和 `WORDPRESS_DB_PASSWORD`。
   -  点击 **添加环境变量**，名称填写 `WORDPRESS_DB_HOST`，值填写 `mysql-service`，对应的是上一篇文档 [快速入门 - 部署 MySQL](../mysql-deployment/#第六步：服务配置) 创建 MySQL 服务的名称，否则无法连接 MySQL 数据库，可在服务列表中查看其服务名。


![容器组模板](/wordpress-container-setting.png)

### 第六步：存储卷设置

此处选择 **添加已有存储卷**，选择第二步创建的存储卷 wordpress-pvc，挂载选项选择 **读写**，挂载路径为 `/var/www/html`。

![存储卷设置](/wordpress-pvc-path.png)

### 第七步：标签设置

为方便识别此应用，我们标签设置如下。下一步的节点选择器可以指定容器组调度到期望运行的节点上，此处暂不作设置，点击创建，部署创建完成。

```
app: wordpress-mysql
tier: frontend
```

创建完成后，部署的状态为 "更新中" 是由于创建后需要拉取 wordpress 镜像并创建容器 (大概一分钟左右)，可以看到容器组的状态是 "ContainerCreating"，待部署创建完成后，状态会显示 “运行中”。

![部署详情](/wordpress-deployment-list.png)

### 第八步：创建服务

8.1. 在当前项目中，左侧菜单选择 **网路与服务 → 服务**，点击 **创建**。基本信息中，信息填写如下：

- 名称：必填，起一个简洁明了的名称，便于用户浏览和搜索，此处填写 `wordpress-service`
- 别名和描述信息：wordpress 服务

![创建服务](/create-wordpress-service.png)

8.2. 服务设置参考如下填写：

- 服务类型：选择 **通过集群内部IP来访问服务 Virtual IP**，
- 选择器：点击 **指定工作负载** 可以指定上一步创建的部署 Wordpress 
- 端口：端口名称可自定义，服务的端口和目标端口都填写 `TCP` 协议的 `80` 端口，完成参数设置，选择下一步。
- 会话亲和性：None

> 说明: 若有实现基于客户端 IP 的会话亲和性的需求，可以在会话亲和性下拉框选择 "ClientIP" 或在代码模式将 service.spec.sessionAffinity 的值设置为 "ClientIP"（默认值为 "None"）。

![服务设置](/service-setting.png)

8.3. 添加标签并保存，本示例标签设置如下，添加后选择下一步。

```
app=wordpress-service
```

8.4. 服务暴露给外网访问的访问方式，支持 NodePort 和 LoadBalancer，由于示例将以应用路由 (Ingress) 的方式添加 Hostname 并暴露服务给外网，所以服务的访问方式选择 **不提供外网访问**。

### 第九步：创建应用路由

通过创建应用路由的方式可以将 WordPress 暴露出去供外网访问，与将服务直接通过 NodePort 或 LoadBalancer 暴露出去不同之处是应用路由是通过配置 Hostname 和路由规则（即 ingress）来访问，请参考以下步骤配置应用路由。关于如何管理应用路由详情请参考 [应用路由](../../ingress-service/ingress)。


9.1. 在当前项目中，左侧菜单选择 **网路与服务 → 应用路由**，点击 **创建**。创建应用路由，填入基本信息：

- 名称：必填，起一个简洁明了的名称，便于用户浏览和搜索，此处填写 `wordpress-ingress`
- 别名和描述信息：wordpress 应用路由

![创建应用路由](/wordpress-ingress-basic.png)

9.2. 下一步，点击 **添加路由规则**，参考如下配置路由规则。
- Hostname：设置为 `kubesphere.wordpress.io`，应用路由规则的访问域名，最终使用此域名来访问对应的服务 (如果访问入口是以 NodePort 的方式启用，需要保证 Hostname 能够在客户端正确解析到集群工作节点上；如果是以 LoadBalancer 方式启用，需要保证 Hostname 正确解析到负载均衡器的 IP 上)。
- 协议：选择 `http` (若使用 https 需要预先在密钥中添加相关的证书)
- Paths：应用规则的路径和对应的后端服务，路径填写 "/"，服务选择之前创建的服务 `wordpress-service`，端口填写 wordpress 服务端口 `80`，完成后点击下一步：


![设置路由规则](/wordpress-ingress-setting.png)

9.3. 添加注解，Annotation 是用户任意定义的“附加”信息，以便于外部工具进行查找，参见 [Annotation 官方文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)。本示例注解如下所示，完成后选择下一步：

```
hostname=kubesphere.wordpress.io
```

9.4. 添加标签如下，选择创建，即可创建成功。

```
app=wordpress-ingress
```

![查看应用路由创建](/ingress-create-result.png)

### 第十步：配置外网访问

在当前项目中，左侧菜单选择 **项目设置 → 外网访问**，点击 **设置网关**，即应用路由的网关入口，每个项目都有一个独立的网关入口。

![设置网关](/gateway-setting.png)

网关入口提供提供如下两种方式：
 
- NodePort：使用 NodePort 方式可以通过访问工作节点对应的端口来访问服务

- LoadBalancer：如果用 LoadBalancer 的方式暴露服务，需要有云服务厂商的 LoadBalancer 插件支持，比如 [QingCloud KubeSphere 托管服务](https://appcenter.qingcloud.com/apps/app-u0llx5j8/Kubernetes%20on%20QingCloud) 可以将公网 IP 地址的 ID 填入 Annotation 中，即可通过公网 IP 访问该服务。(如果外网访问方式设置的是 LoadBalancer，参见 [应用路由](../../ingress-service/ingress) 的 LoadBalancer 方式。)

本示例以 `NodePort` 访问方式为例配置网关入口，点击保存，将产生两个 NodePort，分别是 HTTP 协议的 80 端口和 HTTPS 协议的 443 端口。

> 注意：创建应用路由之后应该把主机的公网 IP 和 配置的域名如：`139.198.17.33 kubesphere.wordpress.io` 填入本地的 `hosts` 配置文件中，即可通过浏览器访问。注意，可能需要进行端口转发和防火墙放行对应的端口，否则外网无法访问。

![端口列表](/gateway-nodeport-list.png)

## 访问 Wordpress
至此，WordPress 就以应用路由的方式通过网关入口暴露到外网，用户可以通过示例中配置的 `kubesphere.wordpress.io:nodeport (如 31463)` 访问 WordPress 博客网站。

![访问 Wordpress](/wordpress-homepage.png)