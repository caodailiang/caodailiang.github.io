---
layout:     post
title:      使用 kubebuilder 开发 Kubernetes Operator
date:       2023-10-06
author:     caodailiang
header-img: img/post-bg-coffee.jpeg
catalog: 	 true
tags:
    - kubernetes
    - operator
    - kubebuilder
---

## Operator 是什么
Kubernetes Operator 是一种特定于应用的控制器，可扩展 Kubernetes API 的功能，来代表 Kubernetes 用户创建、配置和管理复杂应用的实例。

它基于基本 Kubernetes 资源和控制器概念构建，但又涵盖了特定于域或应用的知识，用于实现其所管理软件的整个生命周期的自动化。

Kubernetes Operator 通过自定义资源定义引入新的对象类型。Kubernetes API 可以像处理内置对象一样处理自定义资源定义，包括通过 kubectl 交互以及包含在基于角色的访问权限控制（RBAC）策略中。

## kubebuilder 是什么
Kubebuilder 是一个使用 CRDs 构建 K8s API 的 SDK，主要是： 
- 提供脚手架工具初始化 CRDs 工程，自动生成 boilerplate 代码和配置
- 提供代码库封装底层的 K8s go-client

方便用户从零开始开发 CRDs，Controllers 和 Admission Webhooks 来扩展 Kubernetes。

## Kubebuilder 的工作流程

1. 创建一个新的工程目录
2. 创建一个或多个资源 API CRD 然后将字段添加到资源
3. 在控制器中实现协调循环（reconcile loop），watch 额外的资源
4. 在集群中运行测试（自动安装 CRD 并自动启动控制器）
5. 更新引导集成测试测试新字段和业务逻辑
6. 使用用户提供的 Dockerfile 构建和发布容器

## 使用 kubebuilder 开发 Operator 示例

安装 Kubebuilder 命令行工具后，执行 `kubebuilder init` 命令，就可以生成项目。
```
kubebuilder init --project-name myk8soperator --domain c9g.io --repo github.com/caodailiang/myk8soperator
```
其中 `domain` 是自定义 CRD group 的 domain 后缀，`repo` 是对应的 go module 名。执行命令后，可以看到 Kubebuilder 生成了项目的相关目录，Manager 代码，以及用于部署的 Kubernetes 配置文件。
```
$ tree ./
./
├── cmd
│   └── main.go
├── config
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   └── manager_config_patch.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   └── rbac
│       ├── auth_proxy_client_clusterrole.yaml
│       ├── auth_proxy_role_binding.yaml
│       ├── auth_proxy_role.yaml
│       ├── auth_proxy_service.yaml
│       ├── kustomization.yaml
│       ├── leader_election_role_binding.yaml
│       ├── leader_election_role.yaml
│       ├── role_binding.yaml
│       ├── role.yaml
│       └── service_account.yaml
├── Dockerfile
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── Makefile
├── PROJECT
├── README.md
└── test
    ├── e2e
    │   ├── e2e_suite_test.go
    │   └── e2e_test.go
    └── utils
        └── utils.go

11 directories, 28 files
```

在生成项目的目录和初始文件后，可以采用 `kubebuilder create api` 命令来创建自定义的 CRD 和其 Controller。

```
kubebuilder create api --group samplecontroller --version v1alpha1 --kind Foo
```
其中 `group` 与上一条命令中的 `domain` 一起构成自定义CRD的 APIGroup，加上 `version` 一起够成自定义CRD的GroupVersion： `samplecontroller.c9g.io/v1alpha1`。

执行该命令后，新增和变更的文件列表如下：
```
$ git status
...
Changes not staged for commit:
(use "git add <file>..." to update what will be committed)
(use "git restore <file>..." to discard changes in working directory)
modified:   PROJECT
modified:   cmd/main.go
modified:   config/default/kustomization.yaml

Untracked files:
(use "git add <file>..." to include in what will be committed)
api/
config/crd/
config/rbac/foo_editor_role.yaml
config/rbac/foo_viewer_role.yaml
config/samples/
internal/
```

自定义CRD类型 Foo 的定义在 `api` 目录中，我们需要修改生成的 `api/v1alpha1/foo_types.go` 文件，在其中加入 Foo 资源的相关属性。

```
// FooSpec defines the desired state of Foo
type FooSpec struct {
        // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
        // Important: Run "make" to regenerate code after modifying this file

        Image    string `json:"image"`
        Replicas *int32 `json:"replicas"`
}

// FooStatus defines the observed state of Foo
type FooStatus struct {
        // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
        // Important: Run "make" to regenerate code after modifying this file
        AvailableReplicas int32 `json:"availableReplicas"`
}
```

修改完成后再生成 CRD 的 Kubernetes yaml 定义文件。

```
make manifests
```

CRD 描述文件已经生成，现在可以将自定义 CRD Foo 安装到 Kubernetes 集群中。

```
$ make install
...

$ kubectl get crd
NAME                           CREATED AT
foos.samplecontroller.c9g.io   2023-10-06T04:45:41Z
```

现在来尝试创建一个 foo 资源。

```
$ kubectl apply -f config/samples/samplecontroller_v1alpha1_foo.yaml
$ kubectl get foos.samplecontroller.c9g.io
NAME         AGE
foo-sample   9s
```

自定义CRD Foo 已经创建成功，现在需要完成 Controller 逻辑。修改 `internal/controller/foo_controller.go`， 在其中加入调谐逻辑。本示例中我们只是简单地把 Foo 资源的名称打印出来：

```
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.17.2/pkg/reconcile
func (r *FooReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
        _ = log.FromContext(ctx)

        // TODO(user): your logic here
        fmt.Println("reconcile foo", req.Name)

        return ctrl.Result{}, nil
}
```

Controller 逻辑完成后，就可以构建镜像了：
```
make docker-build docker-push IMG=c9g.io/myk8soperator/sample-controller:v0.4.1
```
使用构建的镜像在集群中部署 Controller：
```
make deploy IMG=c9g.io/myk8soperator/sample-controller:v0.4.1
```
查看部署的 Controller 日志，可以看到对 Foo 资源的处理记录。
```
kubectl logs deployments/myk8soperator-controller-manager -n myk8soperator-system

2023-10-06T06:57:46Z	INFO	controller-runtime.metrics	Metrics server is starting to listen	{"addr": "127.0.0.1:8080"}
2023-10-06T06:57:46Z	INFO	setup	starting manager
2023-10-06T06:57:46Z	INFO	Starting server	{"path": "/metrics", "kind": "metrics", "addr": "127.0.0.1:8080"}
2023-10-06T06:57:46Z	INFO	Starting server	{"kind": "health probe", "addr": "[::]:8081"}
...
2023-10-06T06:58:10Z	INFO	Starting EventSource	{"controller": "foo", "controllerGroup": "samplecontroller.c9g.io", "controllerKind": "Foo", "source": "kind source: *v1alpha1.Foo"}
2023-10-06T06:58:10Z	INFO	Starting Controller	{"controller": "foo", "controllerGroup": "samplecontroller.c9g.io", "controllerKind": "Foo"}
2023-10-06T06:58:10Z	INFO	Starting workers	{"controller": "foo", "controllerGroup": "samplecontroller.c9g.io", "controllerKind": "Foo", "worker count": 1}
reconcile foo foo-sample
```

调试完成后，可以使用 `make uninstall` 卸载 CRD，使用 `make undeploy` 卸载控制器，进行环境清理。 

## 参考文档
- [Kubernetes Operator 模式](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/operator/)
- [kubebuilder](https://book.kubebuilder.io/)
- [kubebuilder 快速入门](https://cloudnative.to/kubebuilder/quick-start.html)
- [Kubernetes Controller 机制详解（二）](https://www.zhaohuabing.com/post/2023-04-04-how-to-create-a-k8s-controller-2/)
