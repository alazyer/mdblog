# Kubernetes Components

k8s 组件通过相互协作从而实现一个可以用的 k8s集群

## 组件分类

k8s中的组件依据所在节点角色不同可以分为：

- Master Component
  - kube-apiserver
  - etcd
  - kube-scheduler
  - kube-controller-manager
  - cloud-controller-manager
- Node Component
  - kubelet
  - kube-proxy
  - container runtime
- 附加组件
  - DNS
  - Dashboard
  - Monitoring
  - Logging
  - 。。。