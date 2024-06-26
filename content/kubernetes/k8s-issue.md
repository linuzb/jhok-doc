+++
title = "Kubernetes 问题排查"
weight = 3
+++


## 网络问题

### dns 问题排查

请参考官方文档[Debugging DNS Resolution | Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)的排查流程。


由于官方文档给的镜像中网络排查工具不全，可以使用以下镜像。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
  namespace: default
spec:
  containers:
  - name: dnsutils
	image: lunettes/lunettes:v0.1.5
    # image: registry.cn-hangzhou.aliyuncs.com/linuzb/jessie-dnsutils:1.3
    command:
      - sleep
      - "infinity"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

### apiserver 连通性

```bash
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

curl -vvsSk -H "Authorization: Bearer $KUBE_TOKEN" https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT/api/v1/namespaces/jhub/pods


curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" https://172.16.0.100:6443/api/v1/namespaces/jhub/pods
```


### nodelocaldns

kubernetes 节点重启后，nodelocaldns crash

原因：[loop (coredns.io)](https://coredns.io/plugins/loop/#troubleshooting)

解决方案
参考 [NodeLocalDNS Loop detected for zone "." · Issue #9948 · kubernetes-sigs/kubespray (github.com)](https://github.com/kubernetes-sigs/kubespray/issues/9948)

删除coredns [重置和重新安装kubernetes中的coreDNS_如何删除现在有的coredns-CSDN博客](https://blog.csdn.net/u013007181/article/details/129731938)

> 1. Change settings in k8 config file for kubespray. I changed 2 things: `resolvconf_mode: none` and `remove_default_searchdomains: false`

然后使用 kubespray 重新部署 kubernetes