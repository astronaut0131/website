# Envoy网格接入

- [Envoy网格接入](#Envoy网格接入)
  - [概览](#概览)
  - [环境准备](#环境准备)
    - [部署polaris](#部署polaris)
    - [部署 polaris-controller](#部署-polaris-controller)
  - [快速接入](#快速接入)
    - [服务调用关系说明](#服务调用关系说明)
    - [启用 sidecar 自动注入功能](#启用-sidecar-自动注入功能)
    - [部署样例](#部署样例)
  - [使用服务治理能力](#使用服务治理能力)
    - [流量调度](#流量调度)
    - [访问限流](#访问限流)
    - [可观测性](#可观测性)
  - [相关链接](#相关链接)

---

## 概览
在 `Polaris` 的服务网格方案中，`Polaris` 是您的控制平面，`Envoy Sidecar` 代理是您的数据平面。

![](图片/使用envoy接入/架构图.png)

- 服务数据同步：`polaris-controller` 安装在用户的Kubernetes集群中，可以同步集群上的 Namespace，Service，Endpoints 等资源到 `polaris` 中，同时 `polaris-controller` 提供了 `Envoy Sidecar` 注入器功能，可以轻松地将 `Envoy Sidecar` 注入到您的 Kubernetes Pod 中，Envoy Sidecar 会自动去 Polaris 同步服务信息。

- 规则数据下发：```polaris```控制面通过XDS v3标准协议与envoy进行交互，支持官方开源的envoy直接接入，当前支持的envoy版本为1.18

## 环境准备

### 部署polaris

如果已经部署好了polaris，可忽略这一步。

polaris支持在kubernetes环境中进行部署，注意必须保证暴露HTTP端口为8090，gRPC端口为8091。具体部署方案请参考：

- [单机版部署指南](https://polarismesh.cn/zh/doc/%E5%BF%AB%E9%80%9F%E5%85%A5%E9%97%A8/%E5%AE%89%E8%A3%85%E6%9C%8D%E5%8A%A1%E7%AB%AF/%E5%AE%89%E8%A3%85%E5%8D%95%E6%9C%BA%E7%89%88.html#kubernetes-%E5%AE%89%E8%A3%85)
- [集群版部署指南](https://polarismesh.cn/zh/doc/%E5%BF%AB%E9%80%9F%E5%85%A5%E9%97%A8/%E5%AE%89%E8%A3%85%E6%9C%8D%E5%8A%A1%E7%AB%AF/%E5%AE%89%E8%A3%85%E9%9B%86%E7%BE%A4%E7%89%88.html#%E9%83%A8%E7%BD%B2%E5%9C%A8kubernetes)

### 部署 polaris-controller 

您需要在应用所在的 Kubernetes 集群部署 `polaris-controller` ，用于将集群中的服务数据接入到`polaris` （如果已经部署可忽略）。如果您有多个 Kubernetes 集群需要接入 `polaris` ，需要在每个集群都部署 `polaris-controller`。

#### 部署包下载

到`polaris-controller`的下载页面，根据您这边的kubernetes版本号（版本号小于等于1.21.x，选择k8s1.21.zip；版本号为1.22.x，选择k8s1.22.zip），下载最新版本polaris-controller安装包。

- [github下载](https://github.com/polarismesh/polaris-controller/releases)

#### 部署包安装

安装前，需确保kubectl命令已经加入环境变量Path中，并且可以访问kubernetes的APIServer。

以```polaris-controller-release_v1.3.0-beta.0.k8s1.21.zip```为例：

解压并进入部署包：

```
unzip polaris-controller-release_v1.3.0-beta.0.k8s1.21.zip
cd polaris-controller-release_v1.3.0-beta.0.k8s1.21
```

查询用户token，由于controller需要直接访问polaris的控制台OpenAPI，因此需要填写token。

- 打开北极星控制台，选择用户->用户列表->选择polaris用户->查看token，即可获取到token。

![](图片/使用envoy接入/查看token.png)

修改variables.txt文件，填写polaris的地址（只填IP或者域名，无需端口），如果在同一个集群中，则可以填写集群内域名，同时需要填写上一步所查询到的token

```
#polaris地址，只填IP或者域名，无需端口
POLARIS_HOST:polaris.polaris-system
#polaris的用户token
POLARIS_TOKEN:4azbewS+pdXvrMG1PtYV3SrcLxjmYd0IVNaX9oYziQygRnKzjcSbxl+Reg7zYQC1gRrGiLzmMY+w+aCxOYI=
```

执行安装部署。

```
./install.sh
```

## 快速接入

### 服务调用关系说明

![](图片/使用envoy接入/bookinfo.png)

### 启用 sidecar 自动注入功能

- 创建命名空间 `bookinfo` ：```kubectl create namespace bookinfo```

- 为 `bookinfo` 命名空间启用注入：
  
```
kubectl label namespace bookinfo polaris-injection=enabled 
```

使用一下命令来验证 `bookinfo` 命名空间是否已经正确启用：

```
kubectl get namespace -L polaris-injection
```

此时应该返回：

```
NAME             STATUS   AGE    POLARIS-INJECTION
bookinfo          Active   3d2h   enabled
```

### 部署样例

- 下载样例部署文件：[bookinfo](https://github.com/polarismesh/examples/blob/main/servicemesh/bookinfo/bookinfo.yaml)

- 执行部署：```kubectl create -f bookinfo.yaml```

- 查看容器注入是否注入成功

启动自动注入后，`polaris-controller` 会将 `Envoy Sidecar` 容器注入到在此命名空间下创建的 pod 中。

可以看到运行起来的 pod 均包含两个容器，其中第一个容器是用户的业务容器，第二个容器是由 Polaris Controller 注入器注入的 Envoy Sidecar 容器。您可以通过下面的命令来获取有关 pod 的更多信息：

```
kubectl describe pods -l app=productpage --namespace=bookinfo
```

此时应返回：

```
... ...
Init Containers:
# polaris-bootstrap-writer 产生 Envoy 的 Bootstrap 配置
polaris-bootstrap-writer:
... ... 
# istio-init 为 envoy sidecar 设置流量拦截
istio-init:
... ... 
Containers:
# demo 的业务容器
productpage:
... ...
# Envoy 是代理流量的容器
envoy:
... ... 
```

- 打开productpage界面

通过productpage暴露的地址，可以访问productpage的主界面，进入Normal User或者TestUser后，可以看到（红、黑、无）三种形态的星星，代表demo已经部署成功。

## 使用服务治理能力

### 流量调度

北极星网格支持根据http请求的头部字段进行路由，支持通过path, header, query这3种类型的属性来进行路由。

1. 使用场景

demo 项目中，productpage 会访问 reviews 服务，reviews 服务共有三个实例，三个实例分别部署了三个版本（会显示红、黑、无三种颜色的星星），需要保证特定的灰度用户（用户名为jason），请求到特定版本的 reviews 服务。

2. 配置路由规则

为 reviews 服务创建路由规则。将请求中 header 包含字段 end-user=jason 的请求，路由到 version=v2 的服务实例中。同时再创建一条路由规则，指定标签键值为任意请求，路由到 version=v1 的服务实例中。

![](图片/使用envoy接入/路由规则.png)

路由规则的标签填写格式要求：

- 对于Path：标签KEY需要填写$path
- 对于Header：标签KEY需要带上前缀$header，如$header.end-user
- 对于Query：标签KEY需要带上前缀$query，如$query.end-user

3. 验证路由是否生效

未登陆时，刷新 productpage 的页面，可以看到只返回没有颜色的星星（version=v1）。当使用 `jason` 登陆后，productpage 请求 reviews 时，会带上 header，end-user=jason，此时再刷新 productpage 页面，发现只会显示黑色的星星，即上面 version=v2 的实例。

### 访问限流

北极星网格支持单机限流和分布式限流，同时直接细粒度的配额的设置。

在envoy接入的场景中，受XDS协议的限制，当前限流粒度只能支持到header以及客户端IP这2个维度。

实现原理：polaris-sidecar提供标准的[RLS协议](https://github.com/envoyproxy/envoy/blob/6bc1b71086a7f2df8a1d9e764823b191cc77c9f6/api/envoy/service/ratelimit/v3/rls.proto)的实现，使得envoy可以直接对接北极星的限流引擎。

![](图片/使用envoy接入/分布式限流.png)

1. 使用场景

demo项目中，为details服务设置流量限制，对于jason用户的请求，设置访问的频率为5/m，其余请求不做限制。

2. 设置限流规则

指定请求中 header 包含字段 end-user=jason 的请求，设置限流规则为5/m，限流类型为分布式限流。

![](图片/使用envoy接入/限流规则.png)

详细的限流规则匹配及使用指南可参考：[访问限流](https://polarismesh.cn/zh/doc/%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97/%E8%AE%BF%E9%97%AE%E9%99%90%E6%B5%81/%E5%8D%95%E6%9C%BA%E9%99%90%E6%B5%81.html#%E5%8D%95%E6%9C%BA%E9%99%90%E6%B5%81)

3. 验证限流是否生效

未登陆时，多次刷新界面，不会出现错误。

以jason用户身份登陆，一分钟刷新超过5次，details界面出现限流的错误信息。



### 可观测性（支持中）






## 相关链接

[Polaris](https://github.com/polarismesh)

[Polaris Controller](https://github.com/PolarisMesh/polaris-controller)

[Polaris Demo](https://github.com/polarismesh/examples/tree/main/servicemesh/bookinfo)