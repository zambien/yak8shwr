# Smoke Test

Now lets run some tests to make sure everything is working as expected.

## Data Encryption

Verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:

```bash
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

The etcd key should be prefixed with k8s:enc:aescbc:v1:key1, which indicates the aescbc provider was used to encrypt the data with the key1 encryption key.

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:

```bash
vagrant ssh k8s-controller-0 \
  --command "sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> output

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 00 0a 0c  0a 02 76 31 12 06 53 65  |y.k8s.....v1..Se|
00000040  63 72 65 74 12 77 0a 5c  0a 17 6b 75 62 65 72 6e  |cret.w.\..kubern|
00000050  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000060  79 12 00 1a 07 64 65 66  61 75 6c 74 22 00 2a 24  |y....default".*$|
00000070  39 62 32 64 39 33 65 62  2d 35 65 62 33 2d 34 64  |9b2d93eb-5eb3-4d|
00000080  62 61 2d 38 61 31 66 2d  35 35 37 31 32 32 30 62  |ba-8a1f-5571220b|
00000090  61 61 62 33 32 00 38 00  42 08 08 bf 85 c2 f2 05  |aab32.8.B.......|
000000a0  10 00 7a 00 12 0f 0a 05  6d 79 6b 65 79 12 06 6d  |..z.....mykey..m|
000000b0  79 64 61 74 61 1a 06 4f  70 61 71 75 65 1a 00 22  |ydata..Opaque.."|
000000c0  00 0a                                             |..|
000000c2
```

## Deployments

Verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```bash
kubectl create deployment nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```bash
kubectl get pods -l app=nginx
```

> output

```bash
NAME                     READY   STATUS    RESTARTS   AGE
nginx-554b9c67f9-vt5rn   1/1     Running   0          10s
```

### Port Forwarding

Verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Note, that your host must understand DNS for workers' IPs.  In order to set this up make sure that you have the following in your `/etc/hosts` on your host machine and controllers:

```bash
10.0.0.20       k8s-worker-0
10.0.0.21       k8s-worker-1
10.0.0.22       k8s-worker-2
```

Retrieve the full name of the `nginx` pod:

```bash
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```bash
kubectl port-forward $POD_NAME 8080:80
```

> output

```bash
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```bash
curl --head http://127.0.0.1:8080
```

> output

```bash
HTTP/1.1 200 OK
Server: nginx/1.17.3
Date: Sat, 14 Sep 2019 21:10:11 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Aug 2019 08:50:00 GMT
Connection: keep-alive
ETag: "5d5279b8-264"
Accept-Ranges: bytes
```

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```bash
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

In this section you will verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` pod logs:

```bash
kubectl logs $POD_NAME
```

> output

```bash
127.0.0.1 - - [14/Sep/2019:21:10:11 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.52.1" "-"
```

### Exec

Verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```bash
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```bash
nginx version: nginx/1.17.8
```

## Services

Verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```bash
kubectl expose deployment nginx --port 80 --type NodePort
```

> The LoadBalancer service type can not be used because your cluster is not configured with [cloud provider integration](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider). Setting up cloud provider integration is out of scope for this tutorial.

Make an HTTP request using a worker IP address and the `nginx` node port:

```bash
curl -I http://10.0.0.20:${NODE_PORT}
```

> output

```bash
HTTP/1.1 200 OK
Server: nginx/1.17.3
Date: Sat, 14 Sep 2019 21:12:35 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Aug 2019 08:50:00 GMT
Connection: keep-alive
ETag: "5d5279b8-264"
Accept-Ranges: bytes
```

### Challenge

Can you figure out how to expose the service via the nodeport through the haproxy load balancer?  See if you can!


Next: [Cleaning Up](11-cleanup.md)