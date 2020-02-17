# Kubeconfigs and Encryption Keys

First, let's create a folder for our configs:

`mkdir -p configs`

#### Generate worker kubeconfigs

The KUBERNETES_PUBLIC_ADDRESS will be the IP/VIP/Pool IP of the load balancer that will direct all traffic to the controller nodes. In our case it’s 10.0.0.30.

Then generate the configs:

```bash

{

KUBERNETES_PUBLIC_ADDRESS=10.0.0.30

for instance in k8s-worker-0 k8s-worker-1 k8s-worker-2; do
  kubectl config set-cluster yak8shwr-kubernetes \
    --certificate-authority=pki/ca/ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=configs/clients/${instance}.kubeconfig
kubectl config set-credentials system:node:${instance} \
    --client-certificate=pki/clients/${instance}.pem \
    --client-key=pki/clients/${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=configs/clients/${instance}.kubeconfig
kubectl config set-context default \
    --cluster=yak8shwr-kubernetes \
    --user=system:node:${instance} \
    --kubeconfig=configs/clients/${instance}.kubeconfig
kubectl config use-context default --kubeconfig=configs/clients/${instance}.kubeconfig
done

}

```

#### Generate kube-proxy kubeconfig

```bash

{

KUBERNETES_PUBLIC_ADDRESS=10.0.0.30

kubectl config set-cluster yak8shwr-kubernetes \
  --certificate-authority=pki/ca/ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=configs/proxy/kube-proxy.kubeconfig
kubectl config set-credentials system:kube-proxy \
  --client-certificate=pki/proxy/kube-proxy.pem \
  --client-key=pki/proxy/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=configs/proxy/kube-proxy.kubeconfig
kubectl config set-context default \
  --cluster=yak8shwr-kubernetes \
  --user=system:kube-proxy \
  --kubeconfig=configs/proxy/kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=configs/proxy/kube-proxy.kubeconfig

}

```

#### Generate kube-controller-manager kubeconfig

```bash

{

kubectl config set-cluster yak8shwr-kubernetes \
  --certificate-authority=pki/ca/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=configs/controller/kube-controller-manager.kubeconfig
kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=pki/controller/kube-controller-manager.pem \
  --client-key=pki/controller/kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=configs/controller/kube-controller-manager.kubeconfig
kubectl config set-context default \
  --cluster=yak8shwr-kubernetes \
  --user=system:kube-controller-manager \
  --kubeconfig=configs/controller/kube-controller-manager.kubeconfig
kubectl config use-context default --kubeconfig=configs/controller/kube-controller-manager.kubeconfig

}

```

#### Generate kube-scheduler kubeconfig

```bash

{

kubectl config set-cluster yak8shwr-kubernetes \
  --certificate-authority=pki/ca/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=configs/scheduler/kube-scheduler.kubeconfig
kubectl config set-credentials system:kube-scheduler \
  --client-certificate=pki/scheduler/kube-scheduler.pem \
  --client-key=pki/scheduler/kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=configs/scheduler/kube-scheduler.kubeconfig
kubectl config set-context default \
  --cluster=yak8shwr-kubernetes \
  --user=system:kube-scheduler \
  --kubeconfig=configs/scheduler/kube-scheduler.kubeconfig
kubectl config use-context default --kubeconfig=configs/scheduler/kube-scheduler.kubeconfig

}

```

#### Generate admin user kubeconfig

```bash

{

kubectl config set-cluster yak8shwr-kubernetes \
  --certificate-authority=pki/ca/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=configs/admin/admin.kubeconfig
kubectl config set-credentials admin \
  --client-certificate=pki/admin/admin.pem \
  --client-key=pki/admin/admin-key.pem \
  --embed-certs=true \
  --kubeconfig=configs/admin/admin.kubeconfig
kubectl config set-context default \
  --cluster=yak8shwr-kubernetes \
  --user=admin \
  --kubeconfig=configs/admin/admin.kubeconfig
kubectl config use-context default --kubeconfig=admin.kubeconfig

}

```

####Generate the data encryption key and config

This will be used for encrypting data between nodes.

```bash

{

mkdir -p data-encryption

ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

cat > data-encryption/encryption-config.yaml <<EOF
apiVersion: v1
kind: EncryptionConfig
resources:
  - resources:
      - secrets
    providers:
    - identity: {}
    - aescbc:
        keys:
        - name: key1
          secret: ${ENCRYPTION_KEY}
EOF

}

```


Next: [Setup Controllers](05-setup-controllers.md)