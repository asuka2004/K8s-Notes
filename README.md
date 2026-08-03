K8s二進制安裝

各個節點IP角色k8-1 192.168.88.91  etcd+apiserver+controller-manager+scheduler+kubelet+kube-proxy

k8-2 192.168.88.92  etcd+apiserver+controller-manager+scheduler+kubelet+kube-proxy

k8-3 192.168.88.93  etcd+apiserver+controller-manager+scheduler+kubelet+kube-proxy

k8-4 192.168.88.94  kubelet+kube-proxy

k8-5 192.168.88.95  kubelet+kube-proxy 

Lab   192.168.88.60 NGINX 反向代理


網段規劃
主機網段-192.168.88.0/24   Service網段-10.96.0.0/16   Pod網段-10.244.0.0/16 

