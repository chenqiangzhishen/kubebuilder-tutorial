Table of Contents
=================

* [一、背景](#一背景)
* [二、环境准备](#二环境准备)
  * [1、Golang 环境搭建](#1golang-环境搭建)
     * [1.1 Golang语言版本](#11-golang语言版本)
     * [1.2 Golang环境配置](#12-golang环境配置)
  * [2、CRD Controller 开发环境搭建](#2crd-controller-开发环境搭建)
     * [2.1、使用时的一些注意事项](#21使用时的一些注意事项)
  * [3、CRD Controller的开发逻辑](#3crd-controller的开发逻辑)
  * [4、CRD controller的测试与部署](#4crd-controller的测试与部署)
* [三、演示](#三演示)
  * [3.1 构架代码生成](#31-构架代码生成)
     * [3.1.1 采用 GOPATH 来生成](#311-采用-gopath-来生成)
     * [3.1.2 采用 go mod 来生成](#312-采用-go-mod-来生成)
  * [3.2 验证生成的代码](#32-验证生成的代码)
     * [3.2.1 查看一下集群状态，确保可访问 k8s 集群](#321-查看一下集群状态确保可访问-k8s-集群)
     * [3.2.2 创建CRD](#322-创建crd)
     * [3.2.3 安装CRD](#323-安装crd)
     * [3.2.4 启动控制器](#324-启动控制器)
  * [3.3 添加编写自定义代码逻辑](#33-添加编写自定义代码逻辑)
  * [3.4 构建 docker 镜像并发布到 docker registry 中](#34-构建-docker-镜像并发布到-docker-registry-中)
  * [3.5 部署镜像到集群](#35-部署镜像到集群)
  * [3.6 删除 CRD](#36-删除-crd)

# 一、背景

目前 K8s 上编写自定义的 CRD/Controller 已经是高级玩家的基本技能了。本文档提供基本的使用方法，并一步一步的提供编写过程。

CRD/Controller 的开发方式经历过不同的阶段，从最早的参考 k8s 官方的控制器代码并手动复制，到用 `client-gen` 生成框架代码，到现在使用 `kubebuilder` ，已经非常方便我们完成 CRD/Controller，甚至 Operator 的开发。

---

# 二、环境准备

但由于中国的网络环境的限制，很多K8s开发依赖的包都无法正常的下载，对此整理出一套容易上手，并且符合未来发展趋势的开发规范出来。

## 1、Golang 环境搭建

### 1.1 Golang语言版本
使用的Golang语言版本建议>=1.13, 该版本级以上默认开启Go Module

### 1.2 Golang环境配置
以下配置针对1.13及以上版本
- 推荐打开 `export GO111MODULE=on` 以强制启用 `Go module`，它是目前最新的Golang包依赖管理工具，也是官方推荐，govender，godep等工具已经不建议继续使用
- 执行： `go env -w GOPROXY=goproxy.cn,direct` 开启代理
当然我们也可以用开源的 athens 搭建一个公司内部私有的 go proxy 代理进行加速，然后配置成类似如下：
`go env -w GOPROXY=http://athens.xxx.com,goproxy.cn,direct`
配置默认从goproxy.cn拉去Go module的依赖包，如果不存在走默认的方式，goproxy.cn 是七牛云维护的一个golang 包代理库，测试下来性能是最好的，可以拉取很多被墙掉的包

## 2、CRD Controller 开发环境搭建

使用[KubeBuilder](https://book.kubebuilder.io/introduction.html) v2版本，该框架可以方便的生成符合K8s规范的CRD文件和对应的Controller代码。

### 2.1、使用时的一些注意事项
- 使用是不建议直接在GOPATH的目录下新建项目，建议使用Go Module在该目录下初始化模块，go mod init yourModuleName
- Controller可用通过Builder上添加自定义的Watches方法来扩展自定义的Watch资源方式
- 修改DockerFile，以方便国内下载
  - a. 将FROM golang:1.12.5 as builder 替换成 FROM golang:1.13 as builder
  - b. 在COPY go.sum 下加上ENV GOPROXY=https://goproxy.cn,direct
  - c. 将FROM gcr.io/distroless/static:nonroot 替换成 FROM golang:1.13
  - d. 删除USER nonroot:nonroot
- 在config/default目录下 kustomization.yaml文件，注释掉 `manager_auth_proxy_patch.yaml`这一行，去掉`manager_prometheus_metrics_patch.yaml`这一行的注释，开启 prometheus 的 metrics 收集

## 3、CRD Controller的开发逻辑

一个控制器的实现方式有两种：
- 水平触发 : 系统仅依赖于当前状态。即使系统错过了某个事件（可能因为故障挂掉了），当它恢复时，依然可以通过查看信号的当前状态来做出正确的响应。
- 边缘触发 : 系统不仅依赖于当前状态，还依赖于过去的状态。如果系统错过了某个事件（“边缘”），则必须重新查看该事件才能恢复系统。

KubeBuilder的控制权实现的接口是Reconcile方法，该方法要求控制器的实现逻辑是基于水平触发的,  实现方不能假定每一次的对象的变更都会触发一次Reconcile，KubeBuilder为性能考虑会合并同一个对象的修改请求，这要求使用方不考虑中间的执行步骤，以面向终态的方式来实现业务逻辑。具体来说每次Status状态的构建都不能依赖过去的值，要求能够完全根据目前的环境的查询来构建值

## 4、CRD controller的测试与部署
- 每次修改结构体或者添加KubeBuilder的Marker后需要运行`make install`命令，该命令生成对应的`CRD yaml`文件并将它部署到当前配置的kubernetes环境中
- 调试时可以使用`make run`命令在本地直接启动controller，该controller连接当前配置的kubernetes的api server
- 打包镜像使用`make docker-build IMG=<some-registry>/<project-name>:tag`
- 上传镜像使用`make docker-push IMG=<some-registry>/<project-name>:tag`
- 部署镜像使用`make deploy IMG=<some-registry>/<project-name>:tag`


---

# 三、演示

## 3.1 构架代码生成

我们现在用 kubebuiler 来生成一个可跑的代码框架。这里演示两种方法，一种是采用 `GOPATH`，另一种是 `go mod`

### 3.1.1 采用 GOPATH 来生成
```bash
chenqiang@Johnny K8S-training$ cd $GOPATH/src
chenqiang@Johnny example.com$ pwd
/Users/chenqiang/go/src/
```
如果对 kubebuilder 的命令不熟悉，可以先查看一下帮助文档， `kubebuilder --help`

我现在就来执行下帮忙文档，看看有哪些东西。

```bash
chenqiang@Johnny src$ kubebuilder --help

Development kit for building Kubernetes extensions and tools.

Provides libraries and tools to create new projects, APIs and controllers.
Includes tools for packaging artifacts into an installer container.

Typical project lifecycle:

- initialize a project:

  kubebuilder init --domain example.com --license apache2 --owner "The Kubernetes authors"

- create one or more a new resource APIs and add your code to them:

  kubebuilder create api --group <group> --version <version> --kind <Kind>

Create resource will prompt the user for if it should scaffold the Resource and / or Controller. To only
scaffold a Controller for an existing Resource, select "n" for Resource. To only define
the schema for a Resource without writing a Controller, select "n" for Controller.

After the scaffold is written, api will run make on the project.

Usage:
  kubebuilder [flags]
  kubebuilder [command]

Examples:

	# Initialize your project
	kubebuilder init --domain example.com --license apache2 --owner "The Kubernetes authors"

	# Create a frigates API with Group: ship, Version: v1beta1 and Kind: Frigate
	kubebuilder create api --group ship --version v1beta1 --kind Frigate

	# Edit the API Scheme
	nano api/v1beta1/frigate_types.go

	# Edit the Controller
	nano controllers/frigate_controller.go

	# Install CRDs into the Kubernetes cluster using kubectl apply
	make install

	# Regenerate code and run against the Kubernetes cluster configured by ~/.kube/config
	make run


Available Commands:
  create      Scaffold a Kubernetes API or webhook.
  help        Help about any command
  init        Initialize a new project
  version     Print the kubebuilder version

Flags:
  -h, --help   help for kubebuilder

Use "kubebuilder [command] --help" for more information about a command.

```

上面有大量的信息，我们基本就可以cp过来，并执行一下。

```bash
chenqiang@Johnny src$ kubebuilder init --domain example.com --license apache2 --owner "The Kubernetes authors"
go get sigs.k8s.io/controller-runtime@v0.4.0
go: downloading k8s.io/apimachinery v0.0.0-20190913080033-27d36303b655
go: downloading k8s.io/client-go v0.0.0-20190918160344-1fbdaa4c8d90
go: extracting k8s.io/apimachinery v0.0.0-20190913080033-27d36303b655
go: extracting k8s.io/client-go v0.0.0-20190918160344-1fbdaa4c8d90
go: downloading github.com/spf13/pflag v1.0.3
go: downloading k8s.io/klog v0.4.0
go: downloading golang.org/x/sys v0.0.0-20190616124812-15dcb6c0061f
go: downloading k8s.io/kube-openapi v0.0.0-20190816220812-743ec37842bf
go: extracting github.com/spf13/pflag v1.0.3
go: extracting k8s.io/klog v0.4.0
go: extracting k8s.io/kube-openapi v0.0.0-20190816220812-743ec37842bf
go: extracting golang.org/x/sys v0.0.0-20190616124812-15dcb6c0061f
go mod tidy
warning: ignoring symlink /Users/chenqiang/go/src/k8s.io/kubernetes/cluster/gce/cos
warning: ignoring symlink /Users/chenqiang/go/src/k8s.io/kubernetes/cluster/gce/custom
warning: ignoring symlink /Users/chenqiang/go/src/k8s.io/kubernetes/cluster/gce/ubuntu
Running make...
make
go: creating new go.mod: module tmp
go: finding sigs.k8s.io v0.2.4
go: finding sigs.k8s.io/controller-tools v0.2.4
go: finding sigs.k8s.io/controller-tools/cmd/controller-gen v0.2.4
go: finding sigs.k8s.io/controller-tools/cmd v0.2.4
go: downloading sigs.k8s.io/controller-tools v0.2.4
go: extracting sigs.k8s.io/controller-tools v0.2.4
go: downloading github.com/spf13/cobra v0.0.5
go: downloading github.com/fatih/color v1.7.0
go: downloading gopkg.in/yaml.v3 v3.0.0-20190905181640-827449938966
go: downloading golang.org/x/tools v0.0.0-20190621195816-6e04913cbbac
go: extracting github.com/fatih/color v1.7.0
go: downloading github.com/mattn/go-isatty v0.0.8
go: downloading github.com/gobuffalo/flect v0.1.5
go: downloading github.com/mattn/go-colorable v0.1.2
go: extracting github.com/mattn/go-isatty v0.0.8
go: extracting github.com/spf13/cobra v0.0.5
go: extracting github.com/gobuffalo/flect v0.1.5
go: extracting github.com/mattn/go-colorable v0.1.2
go: extracting gopkg.in/yaml.v3 v3.0.0-20190905181640-827449938966
go: downloading github.com/inconshreveable/mousetrap v1.0.0
go: extracting github.com/inconshreveable/mousetrap v1.0.0
go: extracting golang.org/x/tools v0.0.0-20190621195816-6e04913cbbac
go: finding github.com/mattn/go-colorable v0.1.2
go: finding gopkg.in/yaml.v3 v3.0.0-20190905181640-827449938966
go: finding github.com/mattn/go-isatty v0.0.8
/Users/chenqiang/go/bin/controller-gen object:headerFile=./hack/boilerplate.go.txt paths="./..."
Error: go [list -e -json -compiled=true -test=false -export=false -deps=true -find=false -tags ignore_autogenerated -- ./...]: exit status 1: build _/Users/chenqiang/go/src: cannot find module for path _/Users/chenqiang/go/src

Usage:
  controller-gen [flags]

Examples:
	# Generate RBAC manifests and crds for all types under apis/,
	# outputting crds to /tmp/crds and everything else to stdout
	controller-gen rbac:roleName=<role name> crd paths=./apis/... output:crd:dir=/tmp/crds output:stdout

	# Generate deepcopy/runtime.Object implementations for a particular file
	controller-gen object paths=./apis/v1beta1/some_types.go

	# Generate OpenAPI v3 schemas for API packages and merge them into existing CRD manifests
	controller-gen schemapatch:manifests=./manifests output:dir=./manifests paths=./pkg/apis/...

	# Run all the generators for a given project
	controller-gen paths=./apis/...

	# Explain the markers for generating CRDs, and their arguments
	controller-gen crd -ww


Flags:
  -h, --detailed-help count   print out more detailed help
                              (up to -hhh for the most detailed output, or -hhhh for json output)
      --help                  print out usage and a summary of options
      --version               show version
  -w, --which-markers count   print out all markers available with the requested generators
                              (up to -www for the most detailed output, or -wwww for json output)


Options


generators

+webhook                                                                                                  package  generates (partial) {Mutating,Validating}WebhookConfiguration objects.
+schemapatch:manifests=<string>[,maxDescLen=<int>]                                                        package  patches existing CRDs with new schemata.
+rbac:roleName=<string>                                                                                   package  generates ClusterRole objects.
+object[:headerFile=<string>][,year=<string>]                                                             package  generates code containing DeepCopy, DeepCopyInto, and DeepCopyObject method implementations.
+crd[:crdVersions=<[]string>][,maxDescLen=<int>][,preserveUnknownFields=<bool>][,trivialVersions=<bool>]  package  generates CustomResourceDefinition objects.


generic

+paths=<[]string>  package  represents paths and go-style path patterns to use as package roots.


output rules (optionally as output:<generator>:...)

+output:artifacts[:code=<string>],config=<string>  package  outputs artifacts to different locations, depending on whether they're package-associated or not.
+output:dir=<string>                               package  outputs each artifact to the given directory, regardless of if it's package-associated or not.
+output:none                                       package  skips outputting anything.
+output:stdout                                     package  outputs everything to standard-out, with no separation.

run `controller-gen object:headerFile=./hack/boilerplate.go.txt paths=./... -w` to see all available markers, or `controller-gen object:headerFile=./hack/boilerplate.go.txt paths=./... -h` for usage
make: *** [generate] Error 1
2020/05/28 21:50:52 exit status 2
chenqiang@Johnny src$

```

会发现有问题。

Error: go [list -e -json -compiled=true -test=false -export=false -deps=true -find=false -tags ignore_autogenerated -- ./...]: exit status 1: build _/Users/chenqiang/go/src: cannot find module for path _/Users/chenqiang/go/src

其实是因为需要先在 `$GOPATH/src` 下面创建一个子目录，然后在该子目录下进行，否则就会出现找不到 module 的错误。

再来一次

```bash
chenqiang@Johnny src$ mkdir example
chenqiang@Johnny src$ cd example && kubebuilder init --domain example.com --license apache2 --owner "The Kubernetes authors"
go get sigs.k8s.io/controller-runtime@v0.4.0
go mod tidy
Running make...
make
/Users/chenqiang/go/bin/controller-gen object:headerFile=./hack/boilerplate.go.txt paths="./..."
go fmt ./...
go vet ./...
go build -o bin/manager main.go
Next: Define a resource with:
$ kubebuilder create api

```

用 `tree` 来看一下文件目录结构

```bash
chenqiang@Johnny example$ tree
.
├── Dockerfile
├── Makefile
├── PROJECT
├── bin
│   └── manager
├── config
│   ├── certmanager
│   │   ├── certificate.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_webhook_patch.yaml
│   │   └── webhookcainjection_patch.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   └── role_binding.yaml
│   └── webhook
│       ├── kustomization.yaml
│       ├── kustomizeconfig.yaml
│       └── service.yaml
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── main.go

9 directories, 29 files
```

这里 kubebuilder 帮我们生成了一下模板文件夹，包括解决 crd 的rbac, certmanager, webhook的文件。

### 3.1.2 采用 go mod 来生成

当然除了上述的方式创建，我们还可以用 `go mod init <my-proj>` 来生成。比如：

```bash
chenqiang@Johnny K8S-training$ cd kubebuilder-eg/
chenqiang@Johnny kubebuilder-eg$ ls
chenqiang@Johnny kubebuilder-eg$ go mod init my.domain
go: creating new go.mod: module my.domain
chenqiang@Johnny kubebuilder-eg$ ls
go.mod
chenqiang@Johnny kubebuilder-eg$ kubebuilder init --domain example.com --license apache2 --owner "The Kubernetes authors"
go get sigs.k8s.io/controller-runtime@v0.4.0
go: finding sigs.k8s.io v0.4.0
go mod tidy
Running make...
make
/Users/chenqiang/go/bin/controller-gen object:headerFile=./hack/boilerplate.go.txt paths="./..."
go fmt ./...
go vet ./...
go build -o bin/manager main.go
Next: Define a resource with:
$ kubebuilder create api
```
可以看出，效果其实是一样的，一个是基于 GOPATH, 一个是基于 go mod， go mod 方式灵活，不需要严格按 golang 的老的目录风格来写代码。

注意：如果出现了形如 `cannot find package ... (from $GOROOT)` 时，需要开启 `$ export GO111MODULE=on`，这样 `go mod` 才会生效。
这主要是因为：kubebuilder 依赖`go module`，所以要打开`go module`环境变量:
`export GO111MODULE=on`
另外proxy或者墙的原因，先设一下go mod的proxy：
`export GOPROXY=https://goproxy.cn`
然后就可以开始使用了

---

## 3.2 验证生成的代码

现在我们只生成了第一步的代码，我们先来看看这部分代码是否能运行及其效果如何？

这时需要保证你的终端能访问k8s的测试集群，简单就是用kubectl cluster-info看看是否出错，如果不出错，就可以run起来main.go了

### 3.2.1 查看一下集群状态，确保可访问 k8s 集群
```bash
chenqiang@Johnny kubebuilder-eg[master*]$ ls
Dockerfile Makefile   PROJECT    bin        config     go.mod     go.sum     hack       main.go
chenqiang@Johnny kubebuilder-eg[master*]$ kubectl cluster-info
Kubernetes master is running at https://10.130.62.59:443
alertmanager is running at https://10.130.62.59:443/api/v1/namespaces/kube-system/services/alertmanager:alertmanager/proxy
Elasticsearch is running at https://10.130.62.59:443/api/v1/namespaces/kube-system/services/elasticsearch-logging/proxy
Kibana is running at https://10.130.62.59:443/api/v1/namespaces/kube-system/services/kibana-logging/proxy
KubeDNS is running at https://10.130.62.59:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://10.130.62.59:443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
monitoring-grafana is running at https://10.130.62.59:443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
phoenix is running at https://10.130.62.59:443/api/v1/namespaces/kube-system/services/phoenix:phoenix/proxy
phoenix-db is running at https://10.130.62.59:443/api/v1/namespaces/kube-system/services/phoenix-db:phoenix-db/proxy
prometheus is running at https://10.130.62.59:443/api/v1/namespaces/kube-system/services/prometheus:prometheus/proxy

```
开始执行，发现正常输出日志了。

```bash
chenqiang@Johnny kubebuilder-eg[master*]$ go run main.go
2020-05-28T22:35:26.024+0800	INFO	controller-runtime.metrics	metrics server is starting to listen	{"addr": ":8080"}
2020-05-28T22:35:26.024+0800	INFO	setup	starting manager
2020-05-28T22:35:26.025+0800	INFO	controller-runtime.manager	starting metrics server	{"path": "/metrics"}

```

从main.go里面可以看出其实kubebuilder帮我们生成一个管理controller的manager的代码，但是还没添加controller（controller是指管理crd的控制器）：

```go
func main() {
...
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:             scheme,
        MetricsBindAddress: metricsAddr,
        LeaderElection:     enableLeaderElection,
        Port:               9443,
    })
...
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
...
    }
}

```

### 3.2.2 创建CRD

接下来我们就可以用kubebuilder帮我们创建一个我们想要的crd, 还是按 help 命令给出的来

	# Create a frigates API with Group: ship, Version: v1beta1 and Kind: Frigate
	kubebuilder create api --group ship --version v1beta1 --kind Frigate

```bash
chenqiang@Johnny kubebuilder-eg[master*]$ kubebuilder create api --group ship --version v1beta1 --kind Frigate
Create Resource [y/n]
y
Create Controller [y/n]
y
Writing scaffold for you to edit...
api/v1beta1/frigate_types.go
controllers/frigate_controller.go
Running make...
/Users/chenqiang/go/bin/controller-gen object:headerFile=./hack/boilerplate.go.txt paths="./..."
go fmt ./...
go vet ./...
go build -o bin/manager main.go

```

这里简单注意一下， group / version / kind 这三个属性组合起来来标识一个 k8s 的 crd。另外就是kind要首字母大写而且不能有特殊符号。
在创建过程中，我们可以选择让 kubebuilder 来是否生成 Resource / Controller 等。

执行上面的命令之后，kubebuilder 就帮我们创建了两个文件 `api/v1/frigate_types.go和controllers/frigate_controller.go`, 前者是这个crd需要定义哪些属性，而后者是对 crd 的 reconcile 的处理逻辑（也就是增删改 crd 的逻辑）, 我们后面再讲这两个文件。最后呢，在 main.go 里面，我们定义的 `Frigate` 对应的 controller 会注册到之前生成的 manager 里：

```go
function main(){
...
// 注册 Frigate 的controller到manager里
    if err = (&controllers.FrigateReconciler{
        Client: mgr.GetClient(),
        Log:    ctrl.Log.WithName("controllers").WithName("Frigate"),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr); err != nil {
...
    }
...
```

当然，如果我们反复执行`kubebuilder create api xxx`这条命令就会帮我们创建和注册不同的 controller 到 manager 里面。

### 3.2.3 安装CRD

基于上述步骤生成的代码，我们什么也不做，先来 `make isntall` 一下， 将其安装到 k8s cluster 中。

```bash
chenqiang@Johnny kubebuilder-eg[master*]$ make install
/Users/chenqiang/go/bin/controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
kustomize build config/crd | kubectl apply -f -
/bin/sh: kustomize: command not found
error: no objects passed to apply
make: *** [install] Error 1
```
这里因为需要安装 kustomize，我们的 kubebuilder 依赖它来部署。
按之前提供的部署文档安装好后，再来看看

```bash
chenqiang@Johnny kubebuilder-eg[master*]$ make install
/Users/chenqiang/go/bin/controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/frigates.ship.example.com created
chenqiang@Johnny kubebuilder-eg[master*]$ kubectl get crd | grep frigate
frigates.ship.example.com                        2020-05-29T04:26:47Z
chenqiang@Johnny kubebuilder-eg[master*]$ kubectl get crd frigates.ship.example.com -o yaml
```
此时已经将 CRD 安装到集群了。

### 3.2.4 启动控制器

再本地跑一下 controller / manager
```bash
chenqiang@Johnny kubebuilder-eg[master*]$ make run
/Users/chenqiang/go/bin/controller-gen object:headerFile=./hack/boilerplate.go.txt paths="./..."
go fmt ./...
go vet ./...
/Users/chenqiang/go/bin/controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
go run ./main.go
2020-05-29T12:31:22.409+0800	INFO	controller-runtime.metrics	metrics server is starting to listen	{"addr": ":8080"}
2020-05-29T12:31:22.410+0800	INFO	setup	starting manager
2020-05-29T12:31:22.410+0800	INFO	controller-runtime.manager	starting metrics server	{"path": "/metrics"}
2020-05-29T12:31:22.511+0800	INFO	controller-runtime.controller	Starting EventSource	{"controller": "frigate", "source": "kind source: /, Kind="}
2020-05-29T12:31:22.613+0800	INFO	controller-runtime.controller	Starting Controller	{"controller": "frigate"}
2020-05-29T12:31:22.717+0800	INFO	controller-runtime.controller	Starting workers	{"controller": "frigate", "worker count": 1}

```
至此，整个流程我们简单的跑了一遍了。

## 3.3 添加编写自定义代码逻辑


这里可以按你自己的程序进行添加。
首先，我们在 `api/v1beta1/frigate_types.go` 中添加一些自定义的字段，以演示 CRD 部分。

```git
--- a/api/v1beta1/frigate_types.go
+++ b/api/v1beta1/frigate_types.go
@@ -29,13 +29,15 @@ type FrigateSpec struct {
        // Important: Run "make" to regenerate code after modifying this file

        // Foo is an example field of Frigate. Edit Frigate_types.go to remove/update
-       Foo string `json:"foo,omitempty"`
+       Foo  string `json:"foo,omitempty"`
+       Demo string `json:"demo,omitempty"`
 }

 // FrigateStatus defines the observed state of Frigate
 type FrigateStatus struct {
        // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
        // Important: Run "make" to regenerate code after modifying this file
+       Created bool `json:"created,omitempty"`
 }
```

然后再控制器调协部分添加如下代码：

```git
--- a/controllers/frigate_controller.go
+++ b/controllers/frigate_controller.go
@@ -38,11 +38,21 @@ type FrigateReconciler struct {
 // +kubebuilder:rbac:groups=ship.example.com,resources=frigates/status,verbs=get;update;patch

 func (r *FrigateReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
-       _ = context.Background()
+       ctx := context.Background()
        _ = r.Log.WithValues("frigate", req.NamespacedName)

        // your logic here
-
+       frigate := &shipv1beta1.Frigate{}
+       if err := r.Get(ctx, req.NamespacedName, frigate); err != nil {
+               return ctrl.Result{}, client.IgnoreNotFound(err)
+       } else {
+               r.Log.V(1).Info("Get demo successfully", "Demo", frigate.Spec.Demo)
+               r.Log.V(1).Info("", "Created", frigate.Status.Created)
+       }
+       if !frigate.Status.Created {
+               frigate.Status.Created = true
+               _ = r.Update(ctx, frigate)
+       }
        return ctrl.Result{}, nil
 }

```
最后，为了不开启代理受权，但开启 promethues 监控，我们修改 `config/default/kustomization.yaml`

```git
--- a/config/default/kustomization.yaml
+++ b/config/default/kustomization.yaml
@@ -27,13 +27,13 @@ patchesStrategicMerge:
   # Protect the /metrics endpoint by putting it behind auth.
   # Only one of manager_auth_proxy_patch.yaml and
   # manager_prometheus_metrics_patch.yaml should be enabled.
-- manager_auth_proxy_patch.yaml
+  # - manager_auth_proxy_patch.yaml
   # If you want your controller-manager to expose the /metrics
   # endpoint w/o any authn/z, uncomment the following line and
   # comment manager_auth_proxy_patch.yaml.
   # Only one of manager_auth_proxy_patch.yaml and
   # manager_prometheus_metrics_patch.yaml should be enabled.
-#- manager_prometheus_metrics_patch.yaml
+- manager_prometheus_metrics_patch.yaml

```
完成后，我们重新在本地进行 `make install`, `make run`。
若没有问题，我们就可以进行接下来的步骤了。


## 3.4 构建 docker 镜像并发布到 docker registry 中

按此格式进行：

`make docker-build docker-push IMG=<some-registry>/<project-name>:tag`

这个过程中需要用到 etcd / apiserver 等二进制可执行文件。这个在解压的 kubebuilder 的文件中可以找到。
如果构建有问题，需要放到 `/usr/local/kubebuilder/bin/` 中
另外，还需要开启 docker service。这个会在构建时使用。

```bash
chenqiang@Johnny bin$ cp etcd /usr/local/kubebuilder/bin/
chenqiang@Johnny bin$ cp kube-apiserver /usr/local/kubebuilder/bin/
make docker-build docker-push IMG=docker-registry.xxx.com/chenqiang/frigate:v1.0
```
这个过程中需要注意如下问题：
请修改DockerFile中
- a. 将FROM golang:1.12.5 as builder 替换成 FROM golang:1.13 as builder
- b. 在COPY go.sum 下加上ENV GOPROXY=https://goproxy.cn,direct
- c. 将FROM gcr.io/distroless/static:nonroot 替换成 FROM golang:1.13
- d. 删除USER nonroot:nonroot

## 3.5 部署镜像到集群

```bash
$ make deploy IMG=docker-registry.xxx.com/chenqiang/frigate:v1.0
/Users/chenqiang/go/bin/controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
cd config/manager && kustomize edit set image controller=docker-registry.xxx.com/chenqiang/frigate:v1.0
kustomize build config/default | kubectl apply -f -
namespace/kubebuilder-tutorial-system created
customresourcedefinition.apiextensions.k8s.io/frigates.ship.example.com configured
role.rbac.authorization.k8s.io/kubebuilder-eg-leader-election-role created
clusterrole.rbac.authorization.k8s.io/kubebuilder-eg-manager-role created
clusterrole.rbac.authorization.k8s.io/kubebuilder-eg-proxy-role created
rolebinding.rbac.authorization.k8s.io/kubebuilder-eg-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/kubebuilder-eg-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/kubebuilder-eg-proxy-rolebinding created
service/kubebuilder-eg-controller-manager-metrics-service created
deployment.apps/kubebuilder-eg-controller-manager created
```

```bash
chenqiang@Johnny kubebuilder-eg[master*]$ kubectl -n kubebuilder-tutorial-system get all
NAME                                                     READY   STATUS         RESTARTS   AGE
pod/kubebuilder-eg-controller-manager-7fbf84bc6c-p9sbd   0/2     ErrImagePull   4          4m13s

NAME                                                        TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
service/kubebuilder-eg-controller-manager-metrics-service   ClusterIP   10.0.44.145   <none>        8443/TCP   4m13s

NAME                                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kubebuilder-eg-controller-manager   0/1     1            0           4m13s

NAME                                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/kubebuilder-eg-controller-manager-7fbf84bc6c   1         1         0       4m13s
```

会出现：

```bash
  Warning  Failed     16s   kubelet, 10.130.62.10  Failed to pull image "gcr.io/kubebuilder/kube-rbac-proxy:v0.4.1": rpc error: code = Unknown desc = Error response from daemon: Get http://gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
  Warning  Failed     16s   kubelet, 10.130.62.10  Error: ErrImagePull

```
使用国内 kubesphere 公司提供的，然后下载就可以了。（这个可以到 docker-hub 中搜索，然后找出国内谁家有提供）
之前本来国内可以使用 `gcr.azk8s.cn` 来下载 gcr.io 的镜像，但现在禁止了，只有 aws 的 IP 才可以。

```
$ docker pull kubesphere/kube-rbac-proxy:v0.4.1
$ docker tag kubesphere/kube-rbac-proxy:v0.4.1 gcr.io/kubebuilder/kube-rbac-proxy:v0.4.1
```

## 3.6 删除 CRD

执行 `make uninstall` 即可






