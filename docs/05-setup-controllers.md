# Setup Controllers

In this section all commands need to be run on all controllers unless stated otherwise. If you’re setting up a single node, run them on there.

Run them at the same time by using [tmux](https://github.com/tmux/tmux/wiki).

You can ssh to each host by using `vagrant ssh k8s-controller-[0-2]`

## Set up etcd

#### Install etcd

```bash
{

wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.0/etcd-v3.4.0-linux-amd64.tar.gz"
  
tar -xvf etcd-v3.4.0-linux-amd64.tar.gz
sudo mv etcd-v3.4.0-linux-amd64/etcd* /usr/local/bin/

}
```

#### Configure etcd

```bash
{

sudo mkdir -p /etc/etcd /var/lib/etcd
sudo cp /vagrant/pki/ca/ca.pem /vagrant/pki/api/kubernetes-key.pem /vagrant/pki/api/kubernetes.pem /etc/etcd/
INTERNAL_IP=$(ip addr show enp0s8 | grep -Po 'inet \K[\d.]+') #THIS controller's internal IP
ETCD_NAME=$(hostname -s)

}
```

#### Create the etcd service file

```bash
{

cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos
[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster k8s-controller-0=https://10.0.0.10:2380,k8s-controller-1=https://10.0.0.11:2380,k8s-controller-2=https://10.0.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF

}
```

#### Start etcd and enable on reboot

```bash
{

sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd

}
```

#### Test etcd is working

```bash
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

You should see something like:

```bash
1472dedb851b4f1b, started, k8s-controller-2, https://10.0.0.12:2380, https://10.0.0.12:2379, false
2b2f66a230e54686, started, k8s-controller-1, https://10.0.0.11:2380, https://10.0.0.11:2379, false
ae38b1b0afdb26e5, started, k8s-controller-0, https://10.0.0.10:2380, https://10.0.0.10:2379, false

```

## Setup Kubernetes on Controllers

#### Install Kubernetes

```bash
{

sudo mkdir -p /etc/kubernetes/config

wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl"
  

chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/  

}
```

#### Configure the api server

```bash
{

sudo mkdir -p /var/lib/kubernetes/
sudo cp /vagrant/pki/ca/ca.pem /vagrant/pki/ca/ca-key.pem /vagrant/pki/api/kubernetes-key.pem /vagrant/pki/api/kubernetes.pem \
    /vagrant/pki/service-account/service-account-key.pem /vagrant/pki/service-account/service-account.pem \
    /vagrant/data-encryption/encryption-config.yaml /var/lib/kubernetes/

}
```

Create the api server service

```bash
{

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.0.0.10:2379,https://10.0.0.11:2379,https://10.0.0.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

}
```

#### Configure the controller manager

Copy the kubeconfig

```bash
sudo cp /vagrant/configs/controller/kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Create the controller manager service

```bash
{

cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --allocate-node-cidrs=true \\
  --node-cidr-mask-size=24 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --cert-dir=/var/lib/kubernetes \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF

}
```

#### Configure the scheduler

Copy the kubeconfig

```bash
sudo cp /vagrant/configs/scheduler/kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Create the yaml configs

```bash
{

cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF

}
```

Create the scheduler service

```bash
{

cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF

}
```

#### Start up the services you just created

```bash
{

sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler

}
```

#### Setup health checks

Since the /healthz check sits on port 6443, and you’re not exposing that, you need to to nginx installed as a proxy (or any other proxy service you wish of course).

```bash
{

sudo apt install -y nginx
cat > kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;
location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF
sudo rm /etc/nginx/sites-enabled/default
sudo mv kubernetes.default.svc.cluster.local /etc/nginx/sites-available/kubernetes.default.svc.cluster.local
sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
sudo systemctl restart nginx

}
```

Verify everything is working:

``` kubectl get componentstatuses --kubeconfig /vagrant/configs/admin/admin.kubeconfig```

You should see:

```bash
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
```

Check that the nginx proxy works:

```bash
curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

You should see a 200 result:
```bash
HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Mon, 17 Feb 2020 15:23:51 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: keep-alive
X-Content-Type-Options: nosniff
```

## Configure RBAC on controller

On only ***one*** of the controllers setup the ClusterRole and ClusterRoleBinding:

```bash
{

cat <<EOF | kubectl apply --kubeconfig /vagrant/configs/admin/admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
cat <<EOF | kubectl apply --kubeconfig /vagrant/configs/admin/admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF

}
```

That's it.  The controllers are ready.


Next: [Configure the load balancer](06-config-lb.md)