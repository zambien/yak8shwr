# Setup DNS

[There are a ton of options to configure DNS.](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model)

We are going to use kube-router.

Apply kube-router on your main host:

```bash
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/generic-kuberouter.yaml
```

Verify it works... Depending on how fast you are you may see the router move from initializing to running.

```bash
kubectl -n kube-system get po --watch                                         ✔  2409  21:29:34
NAME                READY   STATUS            RESTARTS   AGE
kube-router-6sbbg   0/1     PodInitializing   0          10s
kube-router-g69tq   0/1     PodInitializing   0          10s
kube-router-jcvqj   0/1     PodInitializing   0          10s
kube-router-6sbbg   1/1     Running           0          20s
kube-router-g69tq   1/1     Running           0          24s
kube-router-jcvqj   1/1     Running           0          24s
```

Now we'll allow administration via our host machine.  Exit out of the virtualbox or create a new shell.

```bash
{

KUBERNETES_PUBLIC_ADDRESS=10.0.0.30

kubectl config set-cluster yak8shwr-kubernetes \
        --certificate-authority=pki/ca/ca.pem \
        --embed-certs=true \
        --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
        --kubeconfig=configs/admin/admin-remote.kubeconfig
kubectl config set-credentials admin \
        --client-certificate=pki/admin/admin.pem \
        --client-key=pki/admin/admin-key.pem \
        --kubeconfig=configs/admin/admin-remote.kubeconfig
kubectl config set-context yak8shwr-kubernetes \
        --cluster=yak8shwr-kubernetes \
        --user=admin \
        --kubeconfig=configs/admin/admin-remote.kubeconfig
kubectl config use-context yak8shwr-kubernetes --kubeconfig=configs/admin/admin-remote.kubeconfig

#Then make it permanent

cp configs/admin/admin-remote.kubeconfig ~/.kube/config

}
```

We should now be able to run `kubectl` commands with admin rights:

`kubectl get componentstatuses`


Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)