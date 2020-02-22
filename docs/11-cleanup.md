# Cleanup

Cleanup is easy since this is all configured on VMs on your PC.  BUT WAIT!

Do you want to delete this all right now? If everything was configured properly you should only have around 15GB of actual disk usage.  So you could play around with Kubernetes some more.

Maybe you could try installing a [LoadBalancer](https://jromers.github.io/article/2019/02/howto-install-ingress-and-loadbalancer/) or play around with [running some helm charts](https://docs.bitnami.com/kubernetes/how-to/create-your-first-helm-chart/).

The sky is the limit.

If you do want to destroy all this hard-earned work... if you really reallly do...

```bash
vagrant destroy -f  # I, the person running this command am a disappointment
```