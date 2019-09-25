```
kubectl config set-cluster k8s94 --server=https://192.168.3.94:6443 --insecure-skip-tls-verify=true --kubeconfig=/etc/k8s94
kubectl config set-context default-context --cluster=k8s94 --user=admin --kubeconfig=/etc/k8s94
kubectl config use-context default-context --kubeconfig=/etc/k8s94
kubectl config set-credentials admin --username=admin --password=admin --kubeconfig=/etc/k8s94
```
