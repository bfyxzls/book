## 服务发现和负载均衡

Pod 资源可能会因为任何意外故障而被重建，于是它需要固定的可被“发现”的方式 。

另外，Pod资源仅在集群内可见，它的客户端也可能是集群内的其他Pod 资源，若要开放给 外部网络中的用户访问，则需要事先将其暴露到集群外部，并且要为同一种工作负载的访问 流量进行负载均衡。Kubernetes使用标准的资源对象来解决此类问题，它们是用于为工作负 载添加发现机制及负载均衡功能的 Service 资源和 Endpoint 资源，以及通过七层代理实现请 求流量负载均衡的 Ingress 资源 。

