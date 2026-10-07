# FQDN in Kubernetes

All lookups below ran from the `dnsutils` Pod (namespace `default`) on minikube profile `s11`. For
the cross-namespace examples I made a second namespace:

```
$ kubectl create namespace other
namespace/other created
$ kubectl -n other create deployment echo --image=nginx:1.25-alpine
deployment.apps/echo created
$ kubectl -n other expose deployment echo --port=80
service/echo exposed
```

## 1. What is an FQDN?

A Fully Qualified Domain Name is the complete name of a host, from the host label up to the DNS
root. It does not depend on where you ask from. `www.example.com.` is an FQDN. The final dot is the
root, and most tools add it for you. A short name such as `www` is not an FQDN. The resolver must
complete it with a search domain first.

## 2. Kubernetes Service DNS

Every Service gets a DNS name, and the cluster DNS server (CoreDNS) answers for it:

| Service type | What the name returns |
|---|---|
| ClusterIP, NodePort, LoadBalancer | A record with the Service cluster IP |
| Headless (`clusterIP: None`) | One A record per ready Pod |
| ExternalName | A CNAME to the external name |

Named ports also get SRV records:

```
$ kubectl exec dnsutils -- dig +short SRV _http._tcp.web-service-clusterip.default.svc.cluster.local
0 100 8080 web-service-clusterip.default.svc.cluster.local.
```

## 3. Naming convention

```
<service>.<namespace>.svc.<cluster-domain>
web-service-clusterip.default.svc.cluster.local
```

| Part | Meaning |
|---|---|
| `web-service-clusterip` | Service name |
| `default` | namespace |
| `svc` | fixed label: this is a Service |
| `cluster.local` | cluster domain (the default; set in the kubelet and in the Corefile) |

Other forms:

| Record | Form |
|---|---|
| StatefulSet Pod behind a headless Service | `<pod>.<headless-svc>.<ns>.svc.cluster.local` |
| Any Pod, by IP | `<ip-with-dashes>.<ns>.pod.cluster.local` |
| SRV for a named port | `_<port-name>._<proto>.<svc>.<ns>.svc.cluster.local` |

## 4. Namespace-based DNS

The kubelet writes `/etc/resolv.conf` into each Pod. The search list starts with the Pod's own
namespace:

```
$ kubectl exec dnsutils -- cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- `nameserver 10.96.0.10` is the `kube-dns` Service, which points to CoreDNS.
- `search` lets short names work. `web-service-clusterip` is tried as
  `web-service-clusterip.default.svc.cluster.local` first.
- `ndots:5` means a name with fewer than 5 dots goes through the search list before it is tried as
  is.

Because the first search domain is the Pod's own namespace, a bare name finds only Services in the
same namespace. The `echo` Service lives in `other`, so the bare name fails from `default`:

```
$ kubectl exec dnsutils -- nslookup echo
Server:		10.96.0.10
Address:	10.96.0.10#53

** server can't find echo: NXDOMAIN

command terminated with exit code 1

$ kubectl exec dnsutils -- nslookup echo.other
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	echo.other.svc.cluster.local
Address: 10.100.125.255
```

`echo.other` works because the second search domain, `svc.cluster.local`, completes it.

## 5. Pod-to-Service communication

A Pod in `default` can reach a Service in any form:

```
$ kubectl exec dnsutils -- nslookup web-service-clusterip
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	web-service-clusterip.default.svc.cluster.local
Address: 10.111.27.27

$ kubectl exec dnsutils -- nslookup web-service-clusterip.default
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	web-service-clusterip.default.svc.cluster.local
Address: 10.111.27.27

$ kubectl exec dnsutils -- nslookup web-service-clusterip.default.svc
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	web-service-clusterip.default.svc.cluster.local
Address: 10.111.27.27
```

The same Pod reaches a Service in another namespace by its FQDN:

```
$ kubectl exec curl-client -- curl -s -o /dev/null -w '%{http_code}\n' http://echo.other.svc.cluster.local
200
```

Rules of thumb:

| Case | Use |
|---|---|
| Same namespace | `<service>` |
| Other namespace | `<service>.<namespace>` or the full FQDN |
| Config shared across namespaces | the full FQDN |

A short name costs extra lookups. Because of `ndots:5`, an external name such as `api.github.com`
is first tried with every search domain, and each try returns NXDOMAIN. A trailing dot
(`api.github.com.`) skips the search list. The CoreDNS log shows this cost
(`coredns/README.md`, section 4).

## 6. Real FQDN examples from this cluster

| FQDN | Answer |
|---|---|
| `kubernetes.default.svc.cluster.local` | 10.96.0.1 (API server) |
| `web-service-clusterip.default.svc.cluster.local` | 10.111.27.27 |
| `echo.other.svc.cluster.local` | 10.100.125.255 |
| `web-service-headless.default.svc.cluster.local` | 10.244.0.16, .17, .18 (Pod IPs) |
| `web-stateful-1.web-service-headless.default.svc.cluster.local` | 10.244.0.17 |
| `external-database-service.default.svc.cluster.local` | CNAME `example.com.` |
| `10-244-0-16.default.pod.cluster.local` | 10.244.0.16 |

```
$ kubectl exec dnsutils -- nslookup kubernetes.default.svc.cluster.local
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	kubernetes.default.svc.cluster.local
Address: 10.96.0.1

$ kubectl exec dnsutils -- nslookup echo.other.svc.cluster.local
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	echo.other.svc.cluster.local
Address: 10.100.125.255

$ kubectl exec dnsutils -- nslookup $(kubectl get pod web-stateful-0 -o jsonpath='{.status.podIP}' | tr . -).default.pod.cluster.local
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	10-244-0-16.default.pod.cluster.local
Address: 10.244.0.16
```

The headless and ExternalName lookups are in the main `README.md`, sections 1.4 and 1.5.
