# Code Generator示例

## 地址

<https://github.com/kubernetes/code-generator>

## 涉及命令

- client-gen
- deepcopy-gen
- Informer-gen
- lister-gen
- defaulter-gen



## How-To

1. 生成相应目录

```bash
$ mkdir -p ~/go/src/github.com/alazyer/gocodes/examples/code-generator/pkg/apis
```

2. 编辑文件

samplecontroller/v1alpha1/doc.go

```go
// +k8s:deepcopy-gen=package

// Package v1alpha1 is the v1alpha1 version of the API.
// +groupName=samplecontroller.alazyer.io
package v1alpha1
```

auth/v1alpha1/doc.go

```go
// +k8s:deepcopy-gen=package

// Package v1alpha1 is the v1alpha1 version of the API.
// +groupName=auth.alazyer.io
package v1alpha1
```

samplecontroller/v1alpha1/types.go 和 auth/v1alpha1/types.go

```go
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +genclient
// +genclient:noStatus
// +genclient:nonNamespaced
// +genclient:skipVerbs=deleteCollection
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Foo is a specification for a Foo resource
type Foo struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   FooSpec   `json:"spec"`
	Status FooStatus `json:"status"`
}

// FooSpec is the spec for a Foo resource
type FooSpec struct {
	DeploymentName string `json:"deploymentName"`
	Replicas       *int32 `json:"replicas"`
}

// FooStatus is the status for a Foo resource
type FooStatus struct {
	AvailableReplicas int32 `json:"availableReplicas"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// FooList is a list of Foo resources
type FooList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`

	Items []Foo `json:"items"`
}
```

3. 生成deepcopy、client、lister和informer

> 为github.com/alazyer/gocodes/examples/code-generator/sample-controller/pkg/apis包下面的samplecontroller:v1alpha1生成代码
>
> 生成的代码放到github.com/alazyer/gocodes/examples/code-generator/sample-controller/pkg/client目录下

```bash
$ ~/go/src/k8s.io/code-generator/generate-groups.sh all \
github.com/alazyer/gocodes/examples/code-generator/sample-controller/pkg/client \
github.com/alazyer/gocodes/examples/code-generator/sample-controller/pkg/apis \
"samplecontroller:v1alpha1 auth:v1alpha1" \
--go-header-file ~/go/src/github.com/alazyer/gocodes/examples/code-generator/hack/boilerplate.go.txt
```

4. 生成目录结构如下

```bash
-> % tree ./sample-controller -d 2   
./sample-controller
└── pkg
    ├── apis
    │   ├── auth
    │   │   └── v1alpha1
    │   └── samplecontroller
    │       └── v1alpha1
    └── client
        ├── clientset
        │   └── versioned
        │       ├── fake
        │       ├── scheme
        │       └── typed
        │           ├── auth
        │           │   └── v1alpha1
        │           │       └── fake
        │           └── samplecontroller
        │               └── v1alpha1
        │                   └── fake
        ├── informers
        │   └── externalversions
        │       ├── auth
        │       │   └── v1alpha1
        │       ├── internalinterfaces
        │       └── samplecontroller
        │           └── v1alpha1
        └── listers
            ├── auth
            │   └── v1alpha1
            └── samplecontroller
                └── v1alpha1
```

## defaulter-gen

### 添加tag

```
# doc.go中添加

// +k8s:defaulter-gen=TypeMeta
```

### 编写defaulter函数

```
# defaults.go

package v1alpha1

import (
	"k8s.io/apimachinery/pkg/runtime"
)

func addDefaultingFuncs(scheme *runtime.Scheme) error {
	return RegisterDefaults(scheme)
}

func SetDefaults_FooStatus(obj *FooStatus) {
	if obj.AvailableReplicas != 0 {
		obj.AvailableReplicas = 0
	}
}
```

其中addDefaultingFuncs中调用的RegisterDefaults方法是通过有defaulter-gen生成的

### 生成文件

`defaulter-gen -i github.com/alazyer/gocodes/examples/code-generator/sample-controller/pkg/apis/auth/v1alpha1 -p github.com/alazyer/gocodes/examples/code-generator/sample-controller/pkg/apis/auth/v1alpha1`

生成的文件为`zz_generated.defaults.go`

### 注册

```go
// https://github.com/kubernetes/kubernetes/blob/6f897af2da/staging/src/k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1/register.go

package v1alpha1

import (
	"k8s.io/apimachinery/pkg/runtime"
)

var (
	SchemeBuilder      = runtime.NewSchemeBuilder(addDefaultingFuncs)
	localSchemeBuilder = &SchemeBuilder
	AddToScheme        = localSchemeBuilder.AddToScheme
)

func init() {
	localSchemeBuilder.Register(addDefaultingFuncs)
}
```



## Reference

<https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/>