# CoreDNS

Output is from minikube profile `s11` (Kubernetes v1.35.1, CoreDNS v1.13.1).

## 1. What is CoreDNS?

CoreDNS is a DNS server written in Go. It is a CNCF graduated project. It is built from plugins, and
each plugin does one job: cache, forward, log, or answer from the Kubernetes API. A config file
called the Corefile lists the plugins to use. In Kubernetes it runs as a Deployment in
`kube-system`, behind a Service that is still named `kube-dns` for backward compatibility:

```
$ kubectl -n kube-system get deploy,svc,pods -l k8s-app=kube-dns -o wide
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                                    SELECTOR
deployment.apps/coredns   1/1     1            1           7m45s   coredns      registry.k8s.io/coredns/coredns:v1.13.1   k8s-app=kube-dns

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE     SELECTOR
service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   7m45s   k8s-app=kube-dns

NAME                           READY   STATUS    RESTARTS   AGE     IP           NODE   NOMINATED NODE   READINESS GATES
pod/coredns-7d764666f9-5859z   1/1     Running   0          7m40s   10.244.0.2   s11    <none>           <none>
```

`10.96.0.10` is the same address that every Pod has as `nameserver` in `/etc/resolv.conf`.

## 2. Why Kubernetes uses CoreDNS

CoreDNS replaced kube-dns as the default in Kubernetes 1.13.

| kube-dns (old) | CoreDNS |
|---|---|
| Three containers: kubedns, dnsmasq, sidecar | One process |
| dnsmasq is C code with a history of CVEs | Go, memory safe |
| Fixed behaviour, set with flags | Plugin chain, changed by editing a ConfigMap |
| Restart needed to change config | `reload` plugin picks up changes |

CoreDNS also exports Prometheus metrics (port 9153), forwards to any upstream, and can add custom
zones or rewrites without a new image.

## 3. How service discovery works

1. The `kubernetes` plugin in CoreDNS watches Services and EndpointSlices through the API server.
   It keeps them in memory.
2. A new Service gets a name at once: `<svc>.<ns>.svc.cluster.local`. No one registers anything by
   hand.
3. A normal Service name returns the cluster IP. A headless Service name returns the ready Pod IPs
   from the EndpointSlice. An ExternalName Service returns a CNAME.
4. When Pods change, the EndpointSlice changes. CoreDNS sees the watch event and updates its
   headless answers.

Proof that a Service only exists in DNS while it exists in the API: a name with no Service gives
NXDOMAIN.

```
$ kubectl exec dnsutils -- nslookup no-such-svc
Server:		10.96.0.10
Address:	10.96.0.10#53

** server can't find no-such-svc: NXDOMAIN

command terminated with exit code 1
```

## 4. How a DNS query is resolved

1. The app calls the resolver in its Pod with a name, for example `echo.other.svc.cluster.local`.
2. The resolver reads `/etc/resolv.conf`. The name has fewer than 5 dots (`ndots:5`), so it tries
   each search domain first.
3. Each query goes to `10.96.0.10:53`. kube-proxy forwards it to a CoreDNS Pod.
4. CoreDNS runs the query through its plugins. Names in `cluster.local` are answered by the
   `kubernetes` plugin. `hosts` answers `host.minikube.internal`. All other names go to `forward`,
   which sends them to the node's upstream resolver.
5. The answer goes back to the Pod.

The CoreDNS log (minikube turns on the `log` plugin) shows step 2 for one curl to
`echo.other.svc.cluster.local`. The resolver first tried the name with the search domains added and
got NXDOMAIN. Then it tried the name as given and got NOERROR:

```
$ kubectl -n kube-system logs deploy/coredns --tail=5
[INFO] 10.244.0.19:33695 - 61982 "AAAA IN echo.other.svc.cluster.local.svc.cluster.local. udp 64 false 512" NXDOMAIN qr,aa,rd 157 0.000182938s
[INFO] 10.244.0.19:38163 - 5799 "AAAA IN echo.other.svc.cluster.local.cluster.local. udp 60 false 512" NXDOMAIN qr,aa,rd 153 0.000113626s
[INFO] 10.244.0.19:38163 - 5334 "A IN echo.other.svc.cluster.local.cluster.local. udp 60 false 512" NXDOMAIN qr,aa,rd 153 0.000138993s
[INFO] 10.244.0.19:44873 - 2785 "AAAA IN echo.other.svc.cluster.local. udp 46 false 512" NOERROR qr,aa,rd 139 0.000192162s
[INFO] 10.244.0.19:44873 - 2347 "A IN echo.other.svc.cluster.local. udp 46 false 512" NOERROR qr,aa,rd 90 0.00024254s
```

An external name goes through `forward`. The answer is "Non-authoritative" because it comes from
upstream:

```
$ kubectl exec dnsutils -- nslookup kubernetes.io
Server:		10.96.0.10
Address:	10.96.0.10#53

Non-authoritative answer:
Name:	kubernetes.io
Address: 15.197.167.90
Name:	kubernetes.io
Address: 3.33.186.135
```

## 5. CoreDNS configuration

The Corefile lives in the `coredns` ConfigMap:

```
$ kubectl -n kube-system get cm coredns -o yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        log
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
        hosts {
           192.168.76.1 host.minikube.internal
           fallthrough
        }
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30 {
           disable success cluster.local
           disable denial cluster.local
        }
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2026-10-07T14:38:35Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "274"
  uid: 422671c1-09b4-4aca-8cac-fa8516c87626
```

| Line | What it does |
|---|---|
| `.:53` | One server block for all zones (`.`) on port 53. |
| `log` | Logs every query. minikube adds it. The upstream default does not have it. |
| `errors` | Logs errors to stdout. |
| `health { lameduck 5s }` | `/health` on port 8080 for the liveness probe. On shutdown it waits 5 s so in-flight queries finish. |
| `ready` | `/ready` on port 8181 for the readiness probe. It is ready when all plugins are ready, which includes the first sync with the API. |
| `kubernetes cluster.local in-addr.arpa ip6.arpa` | Answers Service and Pod names in `cluster.local`, and reverse lookups. |
| `pods insecure` | Answers `<ip-with-dashes>.<ns>.pod.cluster.local` without checking that the Pod exists. |
| `fallthrough in-addr.arpa ip6.arpa` | A reverse lookup it cannot answer goes to the next plugin instead of NXDOMAIN. |
| `ttl 30` | Answers are cached by clients for 30 s. |
| `prometheus :9153` | Metrics endpoint. |
| `hosts { ... }` | Static records. minikube adds `host.minikube.internal` so Pods can reach the host. |
| `forward . /etc/resolv.conf` | Sends all other names to the upstream servers in the node's resolv.conf. |
| `cache 30` | Caches answers for up to 30 s. Here caching is off for `cluster.local`, so Service changes show at once. |
| `loop` | Detects a forwarding loop (CoreDNS forwarding to itself) and stops the process. |
| `reload` | Reloads the Corefile when the ConfigMap changes (about every 30 s), without a restart. |
| `loadbalance` | Shuffles the order of A records in each answer (round-robin). |

To change it: `kubectl -n kube-system edit cm coredns`. A common addition is a stub zone, for example
`forward corp.example 10.0.0.53` in its own server block, for company DNS.

## 6. Troubleshooting DNS issues

A checklist, from the Pod outward:

1. Check `/etc/resolv.conf` in the Pod. The nameserver must be the `kube-dns` IP (`kubectl get svc
   -n kube-system kube-dns`). The search list must be right.
2. Run `nslookup kubernetes.default`. If this fails, the problem is CoreDNS or the network, not
   your Service.
3. Check the CoreDNS Pods: `kubectl -n kube-system get pods -l k8s-app=kube-dns`. Read the logs
   with `kubectl -n kube-system logs -l k8s-app=kube-dns`.
4. Check that the `kube-dns` Service has endpoints.
5. Check kube-proxy. Without it, no Pod can reach `10.96.0.10` at all.
6. If the name resolves but the connection fails, DNS is fine. Check the Service selector and its
   endpoints.

### Walkthrough A: CoreDNS is down

I scaled CoreDNS to zero to show what the failure looks like:

```
$ kubectl -n kube-system scale deploy/coredns --replicas=0
deployment.apps/coredns scaled

$ kubectl -n kube-system wait --for=delete pod -l k8s-app=kube-dns --timeout=60s
pod/coredns-7d764666f9-5859z condition met

$ kubectl get endpointslices -n kube-system -l kubernetes.io/service-name=kube-dns
NAME             ADDRESSTYPE   PORTS     ENDPOINTS   AGE
kube-dns-hlq5r   IPv4          <unset>   <unset>     7m51s

$ kubectl exec dnsutils -- nslookup -timeout=3 web-service-clusterip
;; connection timed out; no servers could be reached

command terminated with exit code 1
```

The symptom is a timeout, not NXDOMAIN. A timeout means no DNS server answered. NXDOMAIN means a
server answered "no such name". The empty `kube-dns` endpoints show the cause. Fix: bring CoreDNS
back.

```
$ kubectl -n kube-system scale deploy/coredns --replicas=1
deployment.apps/coredns scaled

$ kubectl -n kube-system rollout status deploy/coredns --timeout=90s
Waiting for deployment "coredns" rollout to finish: 0 of 1 updated replicas are available...
deployment "coredns" successfully rolled out

$ kubectl exec dnsutils -- nslookup web-service-clusterip
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	web-service-clusterip.default.svc.cluster.local
Address: 10.111.27.27
```

### Walkthrough B: the name resolves, but the connection fails

This looks like a DNS problem, but it is not one. I made a Service whose selector (`app=broken`)
matches no Pod:

```
$ kubectl create service clusterip broken --tcp=80:80
service/broken created

$ kubectl exec dnsutils -- nslookup broken
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	broken.default.svc.cluster.local
Address: 10.103.161.232

$ kubectl exec curl-client -- curl -s -m 3 -o /dev/null -w '%{http_code}\n' http://broken
000
command terminated with exit code 7

$ kubectl get endpointslices -l kubernetes.io/service-name=broken
NAME           ADDRESSTYPE   PORTS     ENDPOINTS   AGE
broken-vsxf6   IPv4          <unset>   <unset>     0s

$ kubectl describe svc broken
...
Selector:                 app=broken
...
Endpoints:                
...
```

DNS works: the name returns a cluster IP. curl exit code 7 means "connection refused". It is not
code 6, which would mean "could not resolve host". The empty endpoints show that the selector
matches no Pod. Fix: make the Service selector match the Pod labels.

### Walkthrough C: kube-proxy down (seen by accident)

In this session kube-proxy crashed at cluster start (see the main `README.md`, Findings). CoreDNS
then stayed `0/1` and logged this:

```
$ kubectl -n kube-system logs deploy/coredns --tail=15
[INFO] plugin/ready: Plugins not ready: "kubernetes"
[ERROR] plugin/kubernetes: Failed to watch
...
```

The `kubernetes` plugin reaches the API through the Service IP `10.96.0.1`. That needs kube-proxy
rules. When CoreDNS is not ready, check kube-proxy as well as CoreDNS.

### Quick reference

| Symptom | Likely cause | Check |
|---|---|---|
| `connection timed out; no servers could be reached` | CoreDNS down, or no route to 10.96.0.10 | `kube-dns` endpoints, kube-proxy |
| `NXDOMAIN` for a Service | Wrong name or wrong namespace | `kubectl get svc -A`, use `<svc>.<ns>` |
| Resolves, connection refused | No endpoints | Service selector vs Pod labels |
| External names fail, internal work | Upstream DNS | `forward` target, node `/etc/resolv.conf` |
| Slow lookups | `ndots:5` search expansion | Use FQDN with a trailing dot, or set `dnsConfig.options` |
| CoreDNS CrashLoopBackOff with "Loop detected" | Upstream points back to CoreDNS | Node resolv.conf (often 127.0.0.53 from systemd-resolved) |
