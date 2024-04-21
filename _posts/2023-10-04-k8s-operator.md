---
layout:     post
title:      Kubernetes Operator 开发
date:       2023-10-04
author:     caodailiang
header-img: img/post-bg-coffee.jpeg
catalog: 	 true
tags:
    - kubernetes
    - operator
    - kubebuilder
---

## Operator 是什么
> 待补充


## kubebuilder
> 待补充：kubebuilder是什么

安装 Kubebuilder 命令行工具后，执行 kubebuilder init 命令，就可以生成项目。
```
kubebuilder init --project-name myk8soperator --domain c9g.io --repo github.com/caodailiang/myk8soperator
```
其中 domain 是自定义 CRD group 的 domain 后缀，repo 是对应的 go module 名。 执行命令后，可以看到 Kubebuilder 生成了项目的相关目录，Manager 代码，用于部署的 Kubernetes 配置文件。
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

在生成项目的目录和初始文件后，可以采用 kubebuilder create api 命令来创建自定义的 CRD 和其 Controller。

```
kubebuilder create api --group sampleapi --version v1alpha1 --kind Foo
```
其中 `group` 与上一条命令中的 `domain` 一起构成自定义CRD的 APIGroup，加上 `version` 一起够成自定义CRD的GroupVersion： `sampleapi.c9g.io/v1alpha1`。

执行该命令后，新增和变更的文件列表如下 ，详见：[GitHub Commit](https://github.com/caodailiang/myk8soperator/commit/a3a1a6eb94a6f7fedebf98d16542721918d865cb)
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
