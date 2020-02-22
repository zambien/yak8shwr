# Configure Remote Access

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

```bash
kubectl get componentstatuses
```


Next: [Setup DNS](09-setup-dns.md)