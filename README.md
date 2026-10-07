# Kubernetes Networking & Services Homework — Session 11

Bibek Jyoti Charah — 24bcs10112 (GitHub: SammyUrfen)

I ran every command in this repo on my own machine and pasted the output as it was printed. The
cluster is minikube with the Docker driver (profile `s11`), Kubernetes v1.35.1, on Fedora. The
`kubectl` client is v1.37.0. Long output is cut, and each cut is marked with `...`. Nothing else
is edited.

| Task | Where |
|---|---|
| 1. All 5 Service types | this file, YAML in `services/` |
| 2. Object comparison | this file, section 2 |
| 3. FQDN | `fqdn/README.md` |
| 4. CoreDNS | `coredns/README.md` |

## 1. The five Service types

The YAML comes from the instructor labs, copied into `services/`. I changed three things:

- `04-externalname/service.yaml` points at `example.com` instead of a personal domain. That domain is
  stable and answers HTTP.
- The client Pods (`*/client-pod.yaml`, `dnsutils-pod.yaml`) got CPU and memory limits.
- I added `03-loadbalancer/metallb-config.yaml` and `dnsutils-pod.yaml` (a Pod with `nslookup` and
  `dig`).

Deploy everything:

```
$ minikube addons enable metallb -p s11
* The 'metallb' addon is enabled
$ kubectl apply -f services/03-loadbalancer/metallb-config.yaml
configmap/config configured
$ for d in services/0*; do kubectl apply -f $d/; done; kubectl apply -f services/dnsutils-pod.yaml
deployment.apps/web-app-clusterip created
pod/curl-client created
service/web-service-clusterip created
deployment.apps/web-app-nodeport created
service/web-service-nodeport created
deployment.apps/web-app-loadbalancer created
configmap/config unchanged
service/web-service-loadbalancer created
pod/dns-test-client created
service/external-database-service created
statefulset.apps/web-stateful created
pod/headless-dns-client created
service/web-service-headless created
pod/dnsutils created
```

### 1.1 ClusterIP

A ClusterIP Service gets one virtual IP that only works inside the cluster. kube-proxy sends traffic
for that IP to one of the ready Pods. Here the Service listens on 8080 and forwards to port 80.

```
$ kubectl get svc web-service-clusterip
NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
web-service-clusterip   ClusterIP   10.111.27.27   <none>        8080/TCP   6m45s

$ kubectl get endpointslices -l kubernetes.io/service-name=web-service-clusterip
NAME                          ADDRESSTYPE   PORTS   ENDPOINTS                          AGE
web-service-clusterip-xh52m   IPv4          80      10.244.0.5,10.244.0.6,10.244.0.7   6m45s

$ kubectl exec curl-client -- curl -s -o /dev/null -w '%{http_code}\n' http://web-service-clusterip:8080
200

$ kubectl exec curl-client -- curl -s http://web-service-clusterip.default.svc.cluster.local:8080 | grep title
<title>Welcome to nginx!</title>
```

### 1.2 NodePort

A NodePort Service is a ClusterIP Service plus a fixed port (30000–32767) open on every node. Here
the node port is 30080. I curl it from the host through the minikube node IP.

```
$ kubectl get svc web-service-nodeport
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
web-service-nodeport   NodePort   10.105.10.121   <none>        80:30080/TCP   6m44s

$ minikube ip -p s11
192.168.76.2

$ curl -s http://192.168.76.2:30080 | grep title
<title>Welcome to nginx!</title>
```

### 1.3 LoadBalancer

A LoadBalancer Service is a NodePort Service plus an external IP from a load balancer. Minikube has
no cloud load balancer. `minikube tunnel` can fake one, but it needs sudo, and I have no sudo here.
So I used the MetalLB addon. MetalLB gives out IPs from a pool and answers ARP for them. I set the
pool to `192.168.76.200-192.168.76.210`, inside the minikube Docker network, so the host can reach
it (see `services/03-loadbalancer/metallb-config.yaml`).

```
$ kubectl get svc web-service-loadbalancer
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
web-service-loadbalancer   LoadBalancer   10.105.105.178   192.168.76.200   80:30774/TCP   6m45s

$ curl -s http://$(kubectl get svc web-service-loadbalancer -o jsonpath='{.status.loadBalancer.ingress[0].ip}') | grep title
<title>Welcome to nginx!</title>
```

Kubernetes also gave this Service node port 30774, because a LoadBalancer is built on a NodePort.

### 1.4 ExternalName

An ExternalName Service has no Pods, no cluster IP and no proxy. CoreDNS answers its name with a
CNAME to the external name. Here that is `example.com`.

```
$ kubectl get svc external-database-service
NAME                        TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
external-database-service   ExternalName   <none>       example.com   <none>    6m45s

$ kubectl exec dnsutils -- nslookup external-database-service.default.svc.cluster.local
Server:		10.96.0.10
Address:	10.96.0.10#53

external-database-service.default.svc.cluster.local	canonical name = example.com.
Name:	example.com
Address: 104.20.23.154
Name:	example.com
Address: 172.66.147.243
Name:	example.com
Address: 2606:4700:10::ac42:93f3
Name:	example.com
Address: 2606:4700:10::6814:179a

$ kubectl exec dns-test-client -- curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: example.com' http://external-database-service
200
```

The `Host` header matters. Only DNS changes, so curl would otherwise send
`Host: external-database-service`, and the remote web server would not know that name.

### 1.5 Headless

A headless Service has `clusterIP: None`. There is no virtual IP and no load balancing by
kube-proxy. DNS returns the IP of every ready Pod, and each StatefulSet Pod gets its own stable name.

```
$ kubectl get svc web-service-headless
NAME                   TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
web-service-headless   ClusterIP   None         <none>        80/TCP    6m45s

$ kubectl get pods -l app=web-headless -o wide
NAME             READY   STATUS    RESTARTS   AGE     IP            NODE   NOMINATED NODE   READINESS GATES
web-stateful-0   1/1     Running   0          6m45s   10.244.0.16   s11    <none>           <none>
web-stateful-1   1/1     Running   0          4m21s   10.244.0.17   s11    <none>           <none>
web-stateful-2   1/1     Running   0          4m19s   10.244.0.18   s11    <none>           <none>

$ kubectl exec dnsutils -- nslookup web-service-headless
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	web-service-headless.default.svc.cluster.local
Address: 10.244.0.16
Name:	web-service-headless.default.svc.cluster.local
Address: 10.244.0.18
Name:	web-service-headless.default.svc.cluster.local
Address: 10.244.0.17

$ kubectl exec dnsutils -- nslookup web-stateful-1.web-service-headless
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	web-stateful-1.web-service-headless.default.svc.cluster.local
Address: 10.244.0.17

$ kubectl exec headless-dns-client -- curl -s http://web-stateful-0.web-service-headless | grep title
<title>Welcome to nginx!</title>
```

The three A records are the three Pod IPs from `get pods`. They are not a Service IP.

### 1.6 Summary

| Type | Cluster IP | Reachable from | How I tested it |
|---|---|---|---|
| ClusterIP | yes (10.111.27.27) | inside the cluster | curl from a Pod |
| NodePort | yes | node IP + port 30080 | curl from the host |
| LoadBalancer | yes | external IP 192.168.76.200 (MetalLB) | curl from the host |
| ExternalName | no | DNS CNAME only | nslookup + curl from a Pod |
| Headless | no (`None`) | DNS returns Pod IPs | nslookup from a Pod |

## 2. Object comparison

### 2.1 Deployment vs ReplicaSet

| Point | ReplicaSet | Deployment |
|---|---|---|
| Purpose | Keeps N copies of one Pod template running. | Manages versions of an app. It creates and controls ReplicaSets. |
| Pod management | Creates and deletes Pods directly. Owns them. | Does not own Pods. Its ReplicaSets own them. |
| Scaling | `replicas` field. | `replicas` field. The Deployment passes the number to its current ReplicaSet. |
| Rolling updates | None. A template change affects only new Pods. Old Pods stay. | Yes. A new template makes a new ReplicaSet. The old one scales down step by step. It keeps history for rollback. |
| Use directly? | Rarely. | Yes, for stateless apps. |

Relationship: Deployment → ReplicaSet → Pod. Each object has an `ownerReference` to the one above:

```
$ kubectl get deploy,rs -l app=web-clusterip
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web-app-clusterip   3/3     3            3           7m6s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/web-app-clusterip-84985946cb   3         3         3       7m6s

$ kubectl get rs -l app=web-clusterip -o jsonpath='{.items[0].metadata.ownerReferences[0].kind}/{.items[0].metadata.ownerReferences[0].name}{"\n"}'
Deployment/web-app-clusterip

$ kubectl get pod -l app=web-clusterip -o jsonpath='{.items[0].metadata.ownerReferences[0].kind}/{.items[0].metadata.ownerReferences[0].name}{"\n"}'
ReplicaSet/web-app-clusterip-84985946cb
```

A rolling update makes a second ReplicaSet. The old one goes to 0 but stays, so a rollback can use it:

```
$ kubectl set image deploy/web-app-clusterip nginx-web=nginx:1.27-alpine
deployment.apps/web-app-clusterip image updated

$ kubectl rollout status deploy/web-app-clusterip --timeout=120s
Waiting for deployment "web-app-clusterip" rollout to finish: 1 out of 3 new replicas have been updated...
...
Waiting for deployment "web-app-clusterip" rollout to finish: 1 old replicas are pending termination...
deployment "web-app-clusterip" successfully rolled out

$ kubectl get rs -l app=web-clusterip
NAME                           DESIRED   CURRENT   READY   AGE
web-app-clusterip-84985946cb   0         0         0       7m25s
web-app-clusterip-cf77454cd    3         3         3       19s

$ kubectl rollout history deploy/web-app-clusterip
deployment.apps/web-app-clusterip 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

Scaling the Deployment changes only the active ReplicaSet:

```
$ kubectl scale deploy/web-app-clusterip --replicas=5
deployment.apps/web-app-clusterip scaled

$ kubectl get rs -l app=web-clusterip
NAME                           DESIRED   CURRENT   READY   AGE
web-app-clusterip-84985946cb   0         0         0       7m25s
web-app-clusterip-cf77454cd    5         5         3       19s
```

### 2.2 Deployment vs DaemonSet vs StatefulSet

| Point | Deployment | DaemonSet | StatefulSet |
|---|---|---|---|
| Use cases | Stateless apps: web servers, APIs. | One agent per node: log shippers, monitoring agents, network plugins, kube-proxy. | Stateful apps that need a fixed identity: databases, Kafka, ZooKeeper. |
| Pod creation | All at once, random names (`web-app-clusterip-cf77454cd-7czfk`). | One Pod on each matching node. A new node gets a Pod. | One at a time, in order, with fixed names (`web-stateful-0`, `-1`, `-2`). |
| Scaling | Set `replicas`. Pods are interchangeable. | No `replicas` field. It follows the node count (and node selectors). | Set `replicas`. Scale up adds the next ordinal. Scale down removes the highest first. |
| Networking | A normal Service. Clients do not care which Pod answers. | Often `hostNetwork` or a hostPort. Clients talk to the agent on their own node. | Needs a headless Service (`serviceName`). Each Pod gets a stable DNS name. |
| Storage | Usually none, or one shared volume. | Usually hostPath (node files, such as `/var/log`). | `volumeClaimTemplates`: one PVC per Pod. The PVC stays with that ordinal after a restart. |
| Examples | nginx front end | kube-proxy, Fluent Bit, node-exporter | PostgreSQL, MongoDB, Cassandra |

From this cluster: kube-proxy is a DaemonSet with one Pod on the one node, and the StatefulSet Pods
have ordinal names. Their start times are about 2 seconds apart because they start in order. Pod 0
is older because it started before the kube-proxy restart (see Findings).

```
$ kubectl get ds -n kube-system
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   7m39s

$ kubectl get pods -l app=web-headless
NAME             READY   STATUS    RESTARTS   AGE
web-stateful-0   1/1     Running   0          7m23s
web-stateful-1   1/1     Running   0          4m59s
web-stateful-2   1/1     Running   0          4m57s
```

### 2.3 ReplicaSet vs Service

| Point | ReplicaSet | Service |
|---|---|---|
| Responsibility | Keeps the right number of Pods running. Replaces Pods that die. | Gives a stable name and IP for a group of Pods. Sends traffic to the ready ones. |
| Finds Pods by | Label selector, and it owns them. | Label selector, but it does not own them. |
| Changes | Pods are replaced, and each new Pod gets a new IP. | The Service IP and DNS name stay the same for its whole life. |

Why a Service is required: Pod IPs change every time a Pod is replaced. Section 2.1 shows this. After
the rollout, all three Pods had new names and new IPs. A client cannot keep a list of Pod IPs. The
Service gives one fixed address (10.111.27.27, `web-service-clusterip`), and Kubernetes keeps the
list of Pods behind it up to date.

How traffic reaches the Pods:

1. The client resolves `web-service-clusterip` through CoreDNS and gets the Service IP 10.111.27.27.
2. The client sends a packet to 10.111.27.27:8080.
3. On the node, the kube-proxy rules (iptables) pick one ready Pod from the EndpointSlice. They
   rewrite the destination to that Pod IP and port 80.
4. The Pod answers. The reply is translated back, so the client sees it come from the Service IP.

The EndpointSlice controller keeps the slice in sync with the ready Pods that match the selector. A
Service whose selector matches no Pod has an empty slice. Connections to it fail (shown in
`coredns/README.md`, section 6).

## Findings

**kube-proxy crashed at first, and nothing could reach a Service.** Right after the cluster
started, ClusterIP and DNS lookups failed (`curl` exit code 6, `NXDOMAIN` for every name). The
LoadBalancer stayed `<pending>`. The cause was kube-proxy:

```
$ kubectl get pods -A -o wide
...
kube-system      coredns-7d764666f9-5859z                0/1     Running            0               5m23s   10.244.0.2     s11    <none>           <none>
kube-system      kube-proxy-nmfnv                        0/1     CrashLoopBackOff   5 (2m26s ago)   5m23s   192.168.76.2   s11    <none>           <none>
...
$ kubectl -n kube-system logs ds/kube-proxy --tail=8
E1007 14:41:37.107417       1 run.go:72] "command failed" err="failed complete: too many open files"
```

Without kube-proxy, no rules exist for Service IPs. CoreDNS and the MetalLB controller could not
reach the API server through `10.96.0.1`, so they were not ready either. "Too many open files" here
is the inotify instance limit on the host (`fs.inotify.max_user_instances` is 128). Three minikube
clusters were starting on this machine at the same time and share that limit. The fix was to delete
the kube-proxy Pod after the other clusters finished starting:

```
$ kubectl -n kube-system delete pod -l k8s-app=kube-proxy
pod "kube-proxy-nmfnv" deleted from kube-system namespace
$ kubectl -n kube-system rollout status ds/kube-proxy --timeout=60s
Waiting for daemon set "kube-proxy" rollout to finish: 0 of 1 updated pods are available...
daemon set "kube-proxy" successfully rolled out
```

After that, CoreDNS became ready, MetalLB assigned 192.168.76.200, and all the tests above passed. A
permanent fix is to raise `fs.inotify.max_user_instances` on the host. That needs root, so I did not
do it.

## Cleanup

```
$ minikube delete -p s11
```
