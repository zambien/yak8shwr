# Certs and Stuff

So, we need certs. In order to have certs in our local environment we need a certification authority, client and server certs, and various other certificates needed for Kubernetes.  Let's get started...

First, let's create a pki folder so we don't clutter up our project root folder:

`mkdir -p pki/{admin,api,ca,clients,controller,proxy,scheduler,service-account}`

## CA

We'll provision a Certificate Authority which will be used to generate all of our certs.

```bash
{

cat > pki/ca/ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > pki/ca/ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Boone",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "North Carolina"
    }
  ]
}
EOF

cfssl gencert -initca pki/ca/ca-csr.json | cfssljson -bare pki/ca/ca

}
```

#### Now for the admin client cert:

```bash
{

cat > pki/admin/admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Boone",
      "O": "system:masters",
      "OU": "yak8shwr",
      "ST": "North Carolina"
    }
  ]
}
EOF

cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/admin/admin-csr.json | cfssljson -bare pki/admin/admin

}
```

#### Kubelet Client Certs

Note, we will not use an external IP address here as there shouldn't be a need to expose our VMs to the outside world for our testing.

Set your internal IP address (you need to figure this out, it varies).

```bash
INTERNAL_IP=your_internal_ip
```

Then generate the certs and keys for each worker:

```bash
{

i=0
for instance in k8s-worker-0 k8s-worker-1 k8s-worker-2; do
cat > pki/clients/${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Boone",
      "O": "system:nodes",
      "OU": "yak8shwr",
      "ST": "North Carolina"
    }
  ]
}
EOF

INSTANCE_IP="10.0.0.2"i
i=$((i+1))

cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -hostname=${instance},${INTERNAL_IP},${INSTANCE_IP} \
  -profile=kubernetes \
  pki/clients/${instance}-csr.json | cfssljson -bare pki/clients/${instance}
done

}
```

#### Controller Manager Client Cert

```bash
{

cat > pki/controller/kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Boone",
      "O": "system:kube-controller-manager",
      "OU": "yak8shwr",
      "ST": "North Carolina"
    }
  ]
}
EOF

cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
   pki/controller/kube-controller-manager-csr.json | cfssljson -bare  pki/controller/kube-controller-manager

}
```


### The Kube Proxy Client Certificate

Generate the `kube-proxy` client certificate and private key:

```bash
{

cat > pki/proxy/kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Boone",
      "O": "system:node-proxier",
      "OU": "yak8shwr",
      "ST": "North Carolina"
    }
  ]
}
EOF

cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/proxy/kube-proxy-csr.json | cfssljson -bare pki/proxy/kube-proxy

}
```

### The Scheduler Client Certificate

Generate the `kube-scheduler` client certificate and private key:

```bash
{

cat > pki/scheduler/kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Boone",
      "O": "system:kube-scheduler",
      "OU": "yak8shwr",
      "ST": "North Carolina"
    }
  ]
}
EOF

cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/scheduler/kube-scheduler-csr.json | cfssljson -bare pki/scheduler/kube-scheduler

}
```


### The Kubernetes API Server Certificate

The KUBERNETES_PUBLIC_ADDRESS will be the IP/VIP/Pool IP of the load balancer that will direct all traffic to the controller nodes. In our case it’s 10.0.0.30.

Generate the Kubernetes API Server certificate and private key:

```bash
{

KUBERNETES_PUBLIC_ADDRESS=10.0.0.30
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > pki/api/kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Boone",
      "O": "Kubernetes",
      "OU": "yak8shwr",
      "ST": "North Carolina"
    }
  ]
}
EOF

cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -hostname=10.32.0.1,10.0.0.10,10.0.0.11,10.0.0.12,${INTERNAL_IP},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  pki/api/kubernetes-csr.json | cfssljson -bare pki/api/kubernetes

}
```

> The Kubernetes API server is automatically assigned the `kubernetes` internal dns name, which will be linked to the first IP address (`10.32.0.1`) from the address range (`10.32.0.0/24`) reserved for internal cluster services during the [control plane bootstrapping](08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-server) lab.


## The Service Account Key Pair

The Kubernetes Controller Manager leverages a key pair to generate and sign service account tokens as described in the [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/) documentation.

Generate the `service-account` certificate and private key:

```bash
{

cat > pki/service-account/service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Boone",
      "O": "Kubernetes",
      "OU": "yak8shwr",
      "ST": "North Carolina"
    }
  ]
}
EOF

cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/service-account/service-account-csr.json | cfssljson -bare pki/service-account/service-account

}
```


## Distribute the Client and Server Certificates

There is no need to copy the certs around since the project root folder is mounted on the hosts at /vagrant.


> The `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, and `kubelet` client certificates will be used to generate client authentication configuration files in the next lab.

Next: [Kubeconfig and Encryption Keys](04-kubeconfigs.md)