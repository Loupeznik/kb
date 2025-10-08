# Troubleshooting Kubernetes Issues

This document provides a comprehensive guide to troubleshooting common issues in Kubernetes clusters. It covers various components such as Pods, Nodes, Services, and Networking.

## Resolving cert-manager http challenge issues

### NAT Hairpin issue in virtualized environments (Proxmox, VMWare, etc.)

When using cert-manager with the HTTP-01 challenge in a virtualized environment, you may encounter issues where cert-manager cannot complete the challenge due to NAT hairpinning problems. This is common in setups where the Kubernetes cluster is behind a NAT and the external DNS resolves to the public IP of the host, but internal requests cannot reach the service due to hairpinning restrictions.

#### Symptoms

- Cert-manager fails to complete the HTTP-01 challenge.
- The certificate remains in a "Pending" state.
- Logs indicate issues with the HTTP challenge.
- User can connect to the service on port 80 from local machine, but cert-manager cannot complete the challenge. None of the pods can connect to the service using public DNS on port 80.

For example, this can be seen in the challenge status:

```
Name:         argocd-server-tls-1-655850394-627636031
Namespace:    argo-cd
Labels:       <none>
Annotations:  <none>
API Version:  acme.cert-manager.io/v1
Kind:         Challenge
Metadata:
  Creation Timestamp:  2025-09-28T07:27:54Z
  Finalizers:
    acme.cert-manager.io/finalizer
  Generation:  1
  Owner References:
    API Version:           acme.cert-manager.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Order
    Name:                  argocd-server-tls-1-655850394
    UID:                   a915c76b-1fcb-4b2c-9974-7085f1e707a0
  Resource Version:        1889053
  UID:                     06a7e230-0181-45ea-a159-00c638904a09
Spec:
  Authorization URL:  https://acme-v02.api.letsencrypt.org/acme/authz/2689164091/589798583791
  Dns Name:           argocd.k8s-sbx-ew-01.azure.dzarsky.eu
  Issuer Ref:
    Group:  cert-manager.io
    Kind:   ClusterIssuer
    Name:   letsencrypt-prod
  Key:      P-nkcEjuuJVYyw6l7ljfVW04crtXtlBXWPvkkfn21jg.3DlVo-snRJDKKGxFA8Jb5t6Bv5IxqFYx0GtObbyh_VI
  Solver:
    http01:
      Ingress:
        Class:  nginx
  Token:        P-nkcEjuuJVYyw6l7ljfVW04crtXtlBXWPvkkfn21jg
  Type:         HTTP-01
  URL:          https://acme-v02.api.letsencrypt.org/acme/chall/2689164091/589798583791/lcs2zQ
  Wildcard:     false
Status:
  Presented:   true
  Processing:  true
  Reason:      Waiting for HTTP-01 challenge propagation: failed to perform self check GET request 'http://argocd.k8s-sbx-ew-01.azure.dzarsky.eu/.well-known/acme-challenge/P-nkcEjuuJVYyw6l7ljfVW04crtXtlBXWPvkkfn21jg': Get "http://argocd.k8s-sbx-ew-01.azure.dzarsky.eu/.well-known/acme-challenge/P-nkcEjuuJVYyw6l7ljfVW04crtXtlBXWPvkkfn21jg": dial tcp 1.2.3.4:80: connect: connection refused
  State:       pending
Events:
  Type     Reason        Age                 From                     Message
  ----     ------        ----                ----                     -------
  Normal   Started       44m                 cert-manager-challenges  Challenge scheduled for processing
  Warning  PresentError  20m (x10 over 44m)  cert-manager-challenges  Error presenting challenge: admission webhook "validate.nginx.ingress.kubernetes.io" denied the request: ingress contains invalid paths: path /.well-known/acme-challenge/P-nkcEjuuJVYyw6l7ljfVW04crtXtlBXWPvkkfn21jg cannot be used with pathType Exact
  Normal   Presented     19m                 cert-manager-challenges  Presented challenge using HTTP-01 challenge mechanism
```

#### Solution

##### Option 1 - Set up CoreDNS rewrite for split-horizon DNS

Update the CoreDNS configuration to rewrite requests for your domain to the internal service address. This allows pods within the cluster to resolve the domain to the correct internal IP address. This has to be done for each of the subdomains you want to use.

```bash
kubectl edit configmap coredns -n kube-system
```

```yaml
data:
  Corefile: |
    .:53 {
        # Original configuration
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        # NEW CONFIGURATION - Add this section for your domain
        rewrite stop {
            name regex (.*)\.k8s-sbx-ew-01\.azure\.dzarsky\.eu ingress-nginx-controller.ingress-nginx.svc.cluster.local
        }
        # Rest of the original configuration
        cache 30
        loop
        reload
        loadbalance
    }
```

After updating the CoreDNS configuration, restart the CoreDNS pods to apply the changes:

```bash
kubectl rollout restart deployment coredns -n kube-system
```

Restart cert-manager deployments to ensure it picks up the DNS changes:

```bash
kubectl rollout restart deploy -n cert-manager
```

##### Option 2 - Use DNS-01 challenge instead of HTTP-01

If modifying CoreDNS is not feasible, consider switching to the DNS-01 challenge for cert-manager. This method requires you to create DNS TXT records for domain validation, which can be automated if your DNS provider supports API access.
This approach avoids the need for HTTP access and is not affected by NAT hairpinning issues.
Refer to the cert-manager documentation for setting up DNS-01 challenges with your DNS provider.
