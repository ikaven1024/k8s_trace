# Kubernetes集群管理的发展

## kubeadm

kubeadm被设计用来引导k8s集群启动，模块化、易于扩展、易于使用。在v1.5版本中已经是一个稳定版本。

在未来的v1.6版本中，将会增强kubeadm的安全性。将会启用RBAC v1.6，可以创建具有不同的权限的特定用户帐户。

v1.6版本也会添加kubeadm的端到端的测试，保证其稳定性。

## kops

kops提供自动完成集群操作，包括安装、重配置、升级、删除集群。与kubeadm不同的是，kops处理的是资源。目前仅支持AWS，在未来将支持其他运供应商。

[Multimaster Kubernetes Cluster on Amazon Using Kops](https://blog.couchbase.com/2016/november/multimaster-kubernetes-cluster-amazon-kops)

## 云提供

将云提供商提供的功能从基本组件中剔除，统一以插件形式加载，使得核心代码更加精简。

## 其他

**令牌发现**

升级令牌发现，引入新的控制器：[BootstrapSigner](https://github.com/kubernetes/kubernetes/pull/36101)。该控制器会在新的kube-public命名空间中签名一个众所周知的ConfigMap的内容（一个kubeconfig文件）此对象将可供未经身份验证的用户使用，以使用简单和短暂的共享令牌启用安全引导。详见[提议](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/bootstrap-discovery.md)

## 自托管

自托管模式提供了一种新的管理面拉起模式。首先用静态pod方式拉起临时控制面，在临时控制面中注入需要的资源（Deployments，ConfigMaps和DaemonSet），最后关闭临时控制面。自托管也简化了升级工作量。

## 总结

社区现在正在极力简化k8s集群的管理，使得k8s更容易使用，有更好的使用体验。我们的工作和项目中，集群的管理也是重要且不可或缺的一项环节，这方面的引入能够很大程度地减少我们的工作复杂度以及工作量。目前，kubeadm只能提供“全套餐协议”，定制化不足，这也是社区后面需要解决的一个问题。

## 参考文献

[A Stronger Foundation for Creating and Managing Kubernetes Clusters](http://blog.kubernetes.io/2017/01/stronger-foundation-for-creating-and-managing-kubernetes-clusters.html)



